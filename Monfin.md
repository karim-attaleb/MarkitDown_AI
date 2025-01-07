# MonFin Screening Tool for Companies: Summary and Actionable Insights

## **Context and Problem Statement**
- **Objective**: Develop a **monitoring tool (MonFin)** to detect and prevent financial crime by identifying fraudulent companies and their directors.
- **Problem**: Criminals often use companies to hide illegal activities, launder money, and generate illicit income. Current tools are reactive, and there is no comprehensive overview of companies, leading to delayed detection of red flags.
- **Solution**: A **proactive screening tool** that aggregates data from various sources to identify suspicious companies and their networks.

---

## **Key Features of the Tool**
1. **Data Aggregation**:
   - Integrate data from multiple sources (e.g., KBO, NBB, ANG, JustBan, FOD Fin, FOD SZ).
   - Focus on **publicly available data** and **privileged police data** (ANG).
2. **Indicators and Scoring Mechanism**:
   - Use **indicators** to flag suspicious companies and directors.
   - Allow **customization** of indicators and scoring based on regional needs (arrondissement-level).
3. **Dashboard**:
   - Provide a **user-friendly dashboard** for police units to visualize data, relationships between entities, and flagged companies.
   - Enable **proactive and reactive use** (e.g., identifying criminal networks, thematic analysis like dormant addresses or virtual bankruptcies).
4. **Interoperability**:
   - Ensure the tool can integrate with other police tools and systems.
   - Allow **custom queries** and adjustments without relying on external vendors.

---

## **Project Scope and Goals**
- **Long-term Goal**: Provide all police units (FGP) with a screening tool for companies and enable the use of aggregated data for other purposes.
- **Operational Value**:
  - **Reactive**: Efficiently gather information on entities in ongoing cases.
  - **Proactive**: Identify new cases and criminal networks through data analysis.
  - **Strategic**: Use data for national and regional strategic insights (e.g., emerging sectors, interregional shifts).
- **Deliverables**:
  - **Centralized Data Sources**: KBO, NBB, ANG (Phase 1), with additional sources in Phase 2 (FOD Fin, FOD SZ).
  - **Preprocessing Layer**: Transform raw data into actionable indicators.
  - **Screening Tool**: A base tool with a dashboard and pre-configured indicators, customizable by each police unit.

---

## **Project Phases**
1. **Phase 1 (2024 - Early 2025)**:
   - Consolidate the development environment.
   - Optimize data import and scoring mechanisms.
   - Test the tool with FGP Halle-Vilvoorde.
   - Prepare for national rollout.
2. **Phase 2 (Mid-2025)**:
   - Implement the tool on the national infrastructure.
   - Integrate dashboards with partner systems.
   - Conduct further testing with additional police units.
3. **Phase 3 (End of 2025)**:
   - Transition to regular operation.
   - Provide ongoing feedback, maintenance, and updates.

---

## **Budget and Resources**
- **Phase 1 & 2 Costs**:
  - **2024**: €164,318.50 (Drug Fund) + €32,393.59 (Halle-Vilvoorde operational costs).
  - **2025**: €243,000 (estimated for national implementation).
- **Phase 3 Costs**:
  - Ongoing maintenance, technical and functional optimizations, and product ownership.
- **Key Roles**:
  - **Project Manager**: Internal DGJ role for detailed oversight (Phases 1 & 2).
  - **Product Owner**: Internal DGJ role for ongoing management (Phase 3).
  - **Consultants**: Data engineers, software developers, business analysts, Power BI consultants, etc.

---

## **Critical Success Factors**
1. **Human Resources**:
   - Dedicated **Project Manager** and **Product Owner** (internal roles).
   - Expertise in financial crime and data analysis.
2. **Data Quality**:
   - Ensure access to high-quality data sources (KBO, NBB, ANG).
   - Expand with additional government data (FOD Fin, FOD SZ).
3. **Customization**:
   - Allow police units to customize indicators and dashboards based on regional needs.
4. **Interoperability**:
   - Ensure the tool can integrate with other police systems and tools.

---

## **Actionable Insights for Software Development**

1. **Data Integration**:
   - Build a robust **data pipeline** to aggregate data from multiple sources (KBO, NBB, ANG, etc.).
   - Ensure the system can handle **large datasets** (1.14 million companies in Belgium).
   - Plan for future integration of additional data sources (FOD Fin, FOD SZ).

2. **Indicator-Based Scoring**:
   - Develop a **scoring mechanism** that uses predefined indicators to flag suspicious companies.
   - Allow users to **customize indicators** and their weights based on regional needs.
   - Include **machine learning** or **AI models** to improve accuracy over time.

3. **User-Friendly Dashboard**:
   - Create an **intuitive dashboard** that visualizes company data, relationships, and flagged entities.
   - Enable **proactive analysis** (e.g., identifying criminal networks) and **reactive use** (e.g., case-specific investigations).
   - Provide **thematic analysis** options (e.g., dormant addresses, virtual bankruptcies).

4. **Interoperability and Flexibility**:
   - Ensure the tool can integrate with other police systems and tools.
   - Allow users to run **custom queries** and make adjustments without external support.
   - Build a **modular architecture** to accommodate future updates and expansions.

5. **Testing and Rollout**:
   - Start with a **pilot phase** (FGP Halle-Vilvoorde) to test functionality and gather feedback.
   - Gradually roll out the tool to other police units, ensuring proper training and support.
   - Plan for **ongoing maintenance** and updates post-launch.

6. **Budget and Resource Planning**:
   - Allocate resources for **consultants** (data engineers, software developers, etc.) during the development phase.
   - Plan for **ongoing costs** (maintenance, product ownership, technical support) post-launch.

7. **Compliance and Security**:
   - Ensure the tool complies with data privacy regulations (e.g., GDPR).
   - Implement robust **security measures** to protect sensitive police data.

---

## **Next Steps**
1. **Define Requirements**:
   - Work with stakeholders to finalize the list of data sources, indicators, and dashboard features.
2. **Develop a Prototype**:
   - Build a prototype focusing on data integration, scoring mechanisms, and a basic dashboard.
3. **Test and Iterate**:
   - Conduct pilot testing with FGP Halle-Vilvoorde and gather feedback for improvements.
4. **Plan for National Rollout**:
   - Prepare for the national implementation phase, including training and support for police units.
5. **Ensure Ongoing Support**:
   - Establish a team for ongoing maintenance, updates, and user support.

---

This Markdown document provides a clear roadmap for developing the MonFin screening tool, focusing on data integration, user customization, and proactive analysis to combat financial crime effectively. Let me know if you need further assistance!
