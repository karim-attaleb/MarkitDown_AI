# postgres-ha rebuild-readiness review + fix plan

## Context
Goal under review: *"a rebuild a new engineer can execute without errors"* (CLAUDE.md hard rules:
copy-pasteable, docs match reality, followable by someone not present).
Review was static only: every live-run log in `/tmp` (`site.log`, `node3-dbg.log`, …) was already gone
when checked (`ls /tmp/*.log` → no such file, 2026-09-24 12:15). No playbook was run, vault files could
not be decrypted. Templates were rendered in-memory with ansible-core 2.21.4 `Templar` against the lab
inventory (vault vars stubbed) and diffed against `reference/`.

**Bottom line: a new engineer cannot rebuild this today.** The canonical guide never mentions Ansible,
and the Ansible path has at least four hard blockers (firewall, pgBouncer cert, missing HAProxy vault,
unreachable private keys).

---

## 1. Severity-ranked findings

| # | Sev | Finding | file:line | Why it matters | Concrete fix |
|---|---|---|---|---|---|
| 1 | CRIT | BUILD-GUIDE (the declared entry point) contains **zero** mentions of Ansible; it walks the manual path incl. `deploy-certs.sh`. ansible/README says Ansible is the only path and mixing it with deploy-certs.sh caused an etcd outage. | docs/BUILD-GUIDE.md:66-83 (grep -c ansible = 0); ansible/README.md:331-345; README.md:89-91 | New engineer follows the canonical doc → the exact mixed-deployment failure the project already documented. | Rewrite BUILD-GUIDE phases 2-10 around `site.yml`/`validate.yml`; move manual steps to an appendix marked "fallback, never mix". |
| 2 | CRIT | No role opens firewalld for etcd 2379/2380, Patroni 8008, or PostgreSQL 5432 on node1-3. Kickstart enables firewalld with ssh only. | ks/vm.ks.template:31; `grep -c firewalld` = 0 in roles/etcd, patroni, postgres_base; only manual cmds in docs/os-and-packages.md:141-143 | Fresh VMs: etcd can't form quorum, replicas can't stream, HAProxy health checks to :8008 fail → nothing routes. | Add `ansible.posix.firewalld` rich rules in etcd (2379,2380) and patroni (8008 from postgres+haproxy hosts, 5432 from postgres hosts) roles, sources built from inventory. |
| 3 | CRIT | pgBouncer role never deploys `pgbouncer-service.{cert,key}.pem`, but the ini requires them (`client_tls_sslmode = require`). | roles/pgbouncer/templates/pgbouncer.ini.j2:18-20; roles/pgbouncer/tasks/main.yml (no src: for certs); only manual scp in docs/pgbouncer-bringup.md:50-59 | pgBouncer fails TLS/start on a fresh node → whole data path dead. Works today only because the cert was hand-placed earlier. | Add copy tasks (cert 0644, key 0600, owner pgbouncer) to `/etc/pgbouncer/pki/` + restorecon, notify restart. |
| 4 | CRIT | No `group_vars/haproxy/vault.yml` exists; `vault_vrrp_password` is only the placeholder, and the haproxy preflight asserts it isn't. | inventories/lab/group_vars/haproxy.yml:17; playbooks/site.yml:37-51 | `ansible-playbook site.yml` without `--limit` fails at play 2 → nothing is configured. README's bootstrap cmd hides this with `--limit postgres`. | Move haproxy.yml → `group_vars/haproxy/vars.yml`, add `group_vars/haproxy/vault.yml` (encrypted), document creation cmd. |
| 5 | CRIT | Private keys (17 `*.key.pem` + CA key) are gitignored and live only on ispahan; `pki/ca/private/` is empty in the repo. No copy-pasteable procedure to create a CA or obtain keys. | .gitignore:5-7; docs/pki.md:6-16 (describes CA, no command); roles read `{{ pki_local_dir }}` e.g. roles/etcd/tasks/main.yml:26 | A new engineer on a fresh clone cannot run any role that copies a key. Regenerating means a new CA and re-issuing all 20 leaf certs, which isn't documented as a procedure. | Add `pki/bootstrap-ca.sh` + `pki/issue-all.sh` (driven by inventory IPs) and a BUILD-GUIDE step: "either restore keys from <secrets store> or run issue-all.sh". |
| 6 | HIGH | `barman_db_role` (runs on **postgres** hosts) uses `vault_barman_password`, which is defined only in the **backup** group. | roles/barman_db_role/tasks/main.yml:47; group_vars/backup/vars.yml:14 | Undefined-var failure on node1-3, unless `postgres/vault.yml` also defines it (**UNVERIFIED**, since the vault is encrypted). | Move `vault_barman_password` to `group_vars/all/vault.yml`; add it to the cluster-wide preflight. |
| 7 | HIGH | Same scoping bug in validate: the play on `hosts: haproxy` uses `vault_pgbouncer_stats_password`, which is defined only for postgres. | playbooks/validate.yml:134,148; group_vars/postgres/vars.yml:35 | The validation play errors instead of validating (**UNVERIFIED**: depends on vault contents). README's validate cmd (`--limit postgres,backup`) silently skips the HAProxy/VIP checks entirely. | Move the pgbouncer vault vars to `all`; drop `--limit` from the documented validate command. |
| 8 | HIGH | BUILD-GUIDE Phase 9 connects via the VIP as `user=postgres`, but Ansible's userlist.txt deliberately excludes postgres. Also uses `<db>`/`<table>` placeholders. | docs/BUILD-GUIDE.md:164-173; roles/pgbouncer/tasks/main.yml:82 | The "cluster is done" test fails auth on an Ansible-built cluster, and it isn't copy-pasteable. | Create a real `app_smoke` role/db in Ansible and give literal commands; or extend validate.yml to cover 5433 read + write-rejection. |
| 9 | HIGH | Barman WAL-streams from a **physical slot on whichever node is primary** (`slot_name = barman`, `create_slot = auto`); Patroni isn't told about the slot (no `slots:` in dcs). | roles/barman/templates/postgres-ha-cluster.conf.j2:32-37; roles/patroni/templates/patroni.yml.j2:47-66 | After failover the slot doesn't exist on the new primary → WAL gap between failover and slot recreation → PITR across a failover is broken (**UNVERIFIED live**; the doc's failover test only checks `barman status`). | Add `bootstrap.dcs.slots: {barman: {type: physical}}` (Patroni permanent slot) and `patronictl edit-config` for the existing cluster. |
| 10 | HIGH | `bootstrap.dcs` and `bootstrap.pg_hba` apply only at first bootstrap. README claims `postgres_client_allowed_cidrs` "drives" pg_hba; changing it later does nothing. | roles/patroni/templates/patroni.yml.j2:47-83; ansible/README.md:491 | Docs don't match reality; a later tightening of CIDRs, or a sync-mode change in group_vars, is silently ignored. | Move pg_hba to `postgresql.pg_hba` (applied on every node, every reload), and manage dcs settings via `patronictl edit-config` task or doc it explicitly. |
| 11 | HIGH | `setup-ssh-keys.sh` omits the backup host 10.0.0.103. | scripts/setup-ssh-keys.sh:12-16 | The barman play prompts for or fails on SSH. | Generate the host list from inventory, or add the backup host. |
| 12 | HIGH | ansible/README contradicts itself and the repo: "None of this has been run…" / "ansible isn't installed here" vs requirements.yml "found the hard way on the first real run"; vault paths `group_vars/all/vault.yml` are now `inventories/lab/group_vars/…` (staged rename). | ansible/README.md:322, 538, 462-480; collections/requirements.yml:8-17 | CLAUDE.md: a stale status section is a defect; wrong paths break copy-paste. | Rewrite the status + secrets sections against the current tree. |
| 13 | HIGH | Required ansible-core `>=2.18,<2.19` is only in a comment; nothing enforces it. | collections/requirements.yml:6-17 | Engineer installs the current release (2.21) and hits the "undefined variable" breakage the file itself describes. | Add `ansible/requirements.txt` (`ansible-core>=2.18,<2.19`) + a version `assert` preflight play. |
| 14 | MED | Fresh bootstrap: patroni_db_roles/barman_db_role/pgbouncer check `/primary` once with `failed_when: false`; with `--no-block` start, a leader may not exist yet → every create task silently **skips**. | roles/patroni/tasks/main.yml:154-156; roles/pgbouncer/tasks/main.yml:28-36,54 | Roles don't get created, and the run reports success (the same failure class as commit c1cbad2). | Before the DB-role tasks, `run_once` wait until exactly one node returns 200 on `/primary` (retries). |
| 15 | MED | The barman_db_role pg_hba reload is a `when: changed` task with `failed_when: false`, not a handler. | roles/barman_db_role/tasks/main.yml:119-135 | If the run dies after blockinfile, the rerun sees no change → pg_hba is never reloaded; the error is also swallowed. | Convert it to a handler (`patronictl reload` or `pg_reload_conf`). |
| 16 | MED | Hardcoded values bypassing inventory: `10.0.0.0/24` in firewall rules, `ens160` interface, `/var/lib/pgsql/18/data`, VIP `/24`. | roles/pgbouncer/tasks/main.yml:114; roles/haproxy/tasks/main.yml:123; roles/keepalived/templates/keepalived.conf.j2:30,45; roles/barman_db_role/tasks/main.yml:112 | The template isn't reusable, which contradicts the project goal. | Introduce `cluster_subnet_cidr`, `vrrp_interface`, and reuse `postgres_data_dir`. |
| 17 | MED | Docs still reference the moved path `pgbouncer/pgbouncer.ini`. | docs/backup-bringup.md:50,153; roles/barman/templates/postgres-ha-cluster.conf.j2:23 | Dead reference. | Point to `ansible/roles/pgbouncer/templates/pgbouncer.ini.j2`. |
| 18 | MED | Placeholders across the manual docs (`<node-ip>`, `nodeN`, `<your-redhat-account>`…): 70 hits in 12 docs. | e.g. docs/haproxy-bringup.md (12), docs/patroni-bringup.md (11), docs/BUILD-GUIDE.md (9, incl. :95-98) | Violates the copy-paste rule. | Once BUILD-GUIDE is Ansible-driven, keep only literal lab values or per-host loops. |
| 19 | LOW | "11 phases" (0-10), not 10; the guide claims "No single point of failure at any layer". | docs/BUILD-GUIDE.md:15,21-22 | See §4: backup host, control node/keys, and etcd-as-shared-fate are SPOF-like. | Reword and list the known SPOFs. |
| 20 | LOW | Leaf certs expire 2027-09-16 (CA 2036); there's no rotation automation or expiry check. | `openssl x509 -enddate` on pki/*/…cert.pem | A silent outage in about 12 months. | Add an expiry check to validate.yml and a rotation runbook. |
| 21 | LOW | Lint items from the earlier run (names, prefixes, partial-become, etc.). | see previous ansible-lint output | Hygiene. | Fix during the refactor. |

## 2. reference/ vs ansible/roles/: which is authoritative, and drift

**Authoritative: `ansible/roles/`** per reference/README.md:12 and README.md:89-96. **But BUILD-GUIDE
still treats the manual path as primary** (finding #1), so the repo contradicts itself.

Rendered-vs-reference diff (all hosts; comments stripped; YAML compared semantically):

| Pair | Result |
|---|---|
| etcd.conf.yml.j2 vs reference/etcd/node{1,2,3}.conf.yml | **Identical** (semantic-equal=True ×3) |
| haproxy.cfg.j2 vs reference/haproxy/haproxy.cfg | **Identical** functional lines |
| pgbouncer.ini.j2 vs reference/pgbouncer/pgbouncer.ini | **Identical** |
| check-haproxy.sh.j2, barman.conf.j2, postgres-ha-cluster.conf.j2 | **Identical** |
| patroni.yml.j2 vs reference/patroni/node{1,2,3}.yml | Differs **only** in secrets: `REPLACE_ME_*_PASSWORD` vs vault vars (ref lines 60,64,67) + cosmetic quoting of pg_hba lines (ref 48-51). No functional drift. |
| keepalived.conf.j2 vs keepalived-haproxy-{a,b}.conf | Differs only in `auth_pass REPLACE_ME_VRRP_PASSWORD` (ref line 16). |

Caveats (**UNVERIFIED**): this proves repo-vs-repo consistency only, not that either matches the live
hosts. It also doesn't cover drift *outside* templates. The real behavioural changes are in task files
(uncommitted `db`→`login_db`, `verify-full`→`verify-ca`) and in things neither `reference/` nor the templates
contain (firewall, pgBouncer cert, pg_hba additions from barman_db_role).
**Recommendation:** since nothing has drifted, `reference/` is redundant provenance. Delete it (git history keeps it) or
keep it and add a CI check that re-renders and diffs.

## 3. Empty dirs `haproxy/ keepalived/ patroni/ pgbouncer/` and `etcd/` (README only)

- The 4 empty dirs are **not tracked by git** (git doesn't store empty dirs; `git ls-files` shows nothing
  under them). They exist only in this working copy, and a fresh clone won't have them. **Delete them** locally.
- References: no script, playbook or role uses them. Stale **doc** references to `pgbouncer/pgbouncer.ini`
  exist (finding #17). `roadmap.md:217-218` mentions `patroni/node*.yml` historically (fine as history).
- `etcd/README.md`: nothing links to it (`grep etcd/README` = 0). Its content duplicates ansible/README +
  docs/pki.md. **Delete it** and fold its one useful fact (etcd-server cert needs clientAuth) into docs/pki.md
  (already at docs/pki.md:108). README.md's repo-layout block doesn't list these dirs, so no edit is needed there.

## 4. BUILD-GUIDE phases → automation coverage

| Phase | Ansible | Manual script | By hand |
|---|---|---|---|
| 0 Architecture | – | – | reading only |
| 1 VMs | – | `vmware/New-PostgresHaVM.ps1`, `setup-ssh-keys.sh` (misses backup) | VMnet8 setup (docs/vmware-networking.md), SSH to backup host |
| 2 OS + packages | postgres_base, haproxy, keepalived, pgbouncer, barman install packages | – | **`subscription-manager register` on all 6 VMs** (interactive) |
| 3 PKI | cert *distribution* in etcd/patroni/haproxy/barman roles; **not pgBouncer** | `gen-leaf-cert.sh`, `deploy-certs*.sh` (conflicts with Ansible) | **CA creation, key availability, pgBouncer cert** |
| 4 etcd | etcd role | – | **firewall 2379/2380** |
| 5 Patroni/PG | patroni, patroni_db_roles (+ `cluster_bootstrap=true` first run) | – | **firewall 8008/5432**, failover drill |
| 6 pgBouncer | pgbouncer role | – | **TLS cert** |
| 7 HAProxy | haproxy role (incl. restorecon, seboolean, firewall) | deploy-certs-haproxy.sh (redundant) | haproxy vault file creation |
| 8 keepalived | keepalived role | – | VIP-move drill |
| 9 E2E | validate.yml (write routing only, via haproxy-a IP not VIP; no 5433/read-only check) | – | **all read/write-rejection tests; broken as written (#8)** |
| 10 Barman | barman, barman_db_role | – | backup VM creation, `barman backup`/`recover` drill, failover drill |

## 5. SPOF, quorum loss, fencing, split-brain

- **SPOFs:** backup host (single Barman, backups only); the **control node ispahan** (sole holder of the CA key and all
  private keys, so losing it means you can't redeploy); the single VMware Workstation host (all 6 VMs on one machine; **UNVERIFIED**
  whether that's the production intent); etcd shares fate with PG nodes (docs/architecture.md:69-71 acknowledges).
- **Quorum loss:** with 2 of 3 etcd down, Patroni can't renew the leader key within `ttl: 30`
  (patroni.yml.j2:49) → **primary demotes to read-only**; there's no `failsafe_mode`, so a healthy primary stops
  taking writes during an etcd-only outage (e.g. an etcd cert expiry on 2027-09-16 would do exactly this).
- **Fencing:** none. There's no `watchdog:` section in patroni.yml.j2 (grep = 0) and no softdog setup. A hung Patroni
  whose PostgreSQL keeps running is not fenced. HAProxy drops it from the pool (its `/primary` check fails), but existing
  sessions persist: `timeout client/server 30m` and no `on-marked-down shutdown-sessions` (haproxy.cfg.j2:17-18,30).
- **Split-brain (DB):** mitigated, not prevented. `synchronous_mode: true` makes an isolated old primary's commits
  block on the (now promoted) sync standby. But `synchronous_mode_strict: false` (group_vars/postgres/vars.yml:17)
  lets Patroni drop to async when the standby is gone, so an acknowledged-write loss window exists by design.
- **Split-brain (VIP):** keepalived uses multicast VRRP with simple `PASS` auth, no `unicast_peer`, no `nopreempt`
  (keepalived.conf.j2:28-50). If VRRP multicast is filtered between haproxy-a/b, **both hold the VIP**.
  The firewall allows protocol vrrp (keepalived/tasks/main.yml:31-33); VMware multicast behaviour is **UNVERIFIED**.
  `check-haproxy.sh` only checks that haproxy is listening locally, not that any backend is UP.

## 6. Not verified (flagged, not assumed)
- Vault contents (whether `vault_barman_password` / `vault_pgbouncer_stats_password` exist in other groups' vaults).
  Check: `ansible-inventory -i inventories/lab/hosts.yml --host node1 --vars --ask-vault-pass | grep -c vault_barman`.
- Live pg_hba contents (initdb defaults vs Patroni append); whether pgBouncer→127.0.0.1 non-TLS connections for
  `pgbouncer_stats`/app users match any rule. Check: `SELECT * FROM pg_hba_file_rules;` on the primary.
- Whether the lab's live configs still equal `reference/`.
- The pgBouncer version on RHEL 10 PGDG and whether it passes `replication=true` through (Barman streaming path).
- Barman slot behaviour across failover (#9).
- All live-run evidence: `/tmp` logs no longer exist.

---

## Proposed fix plan (after approval, in this order; each step is a separate commit)
1. **Unblock Ansible** (#2, #3, #4, #6, #7, #13): firewalld tasks in etcd/patroni roles; pgBouncer cert tasks;
   `group_vars/haproxy/{vars,vault}.yml`; move shared vault vars to `all`; ansible-core pin + preflight assert.
2. **Bootstrap correctness** (#14, #15, #9, #10): leader-wait before the DB-role tasks; handler for pg_hba reload;
   Patroni permanent `barman` slot; `postgresql.pg_hba` instead of `bootstrap.pg_hba`.
3. **PKI reproducibility** (#5, #20): `pki/bootstrap-ca.sh`, `pki/issue-all.sh`; cert-expiry check in validate.yml.
4. **Docs** (#1, #8, #12, #17, #18, #19): rewrite BUILD-GUIDE around site.yml/validate.yml; extend
   validate.yml to cover the Phase 9 checks through the VIP; fix ansible/README status/paths; list SPOFs.
5. **Cleanup** (§3, #11, #16, #21): delete the empty dirs, etcd/README.md, and (if agreed) reference/; parametrize
   hardcoded values; fix setup-ssh-keys.sh; fix lint findings.

Critical files: `ansible/roles/{etcd,patroni,pgbouncer,barman_db_role}/tasks/main.yml`,
`ansible/roles/patroni/templates/patroni.yml.j2`, `ansible/playbooks/{site,validate}.yml`,
`ansible/inventories/lab/group_vars/*`, `docs/BUILD-GUIDE.md`, `ansible/README.md`, `pki/`.

## Verification
- Static: `yamllint ansible/` + `ansible-lint` (baseline: 18 failures) must not regress; re-run the
  render-and-diff script from this review.
- `ansible-playbook playbooks/site.yml --syntax-check` and `--check --diff --ask-vault-pass` against the lab.
- The real acceptance test: rebuild on **fresh VMs** (new kickstart, empty firewalld) following only the new
  BUILD-GUIDE, then `ansible-playbook playbooks/validate.yml --ask-vault-pass` with **no `--limit`**, plus a
  failover drill (`patronictl switchover`) followed by `barman check` and `barman receive-wal --status`
  (or a PITR restore) to confirm there's no WAL gap across the failover.
