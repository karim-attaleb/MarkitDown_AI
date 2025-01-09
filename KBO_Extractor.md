KBO Extractor
## **1. `entity.py`**
This file defines the `Entity` class, which is responsible for handling the extraction and transformation of data from XML files into CSV format. Here's a breakdown of its key components:

### **Initialization (`__init__`)**:
- The `Entity` class is initialized with parameters like `output_dir`, `name`, `ns` (namespace), `xpath`, `parent_tags`, `direct_tags`, and `list_of_columns_and_types`.
- It creates a CSV file in the specified output directory and writes the header row based on the provided tags.
- It also initializes counters and lists for handling nested entities (L2 entities).

### **Methods**:
- **`find_nodes` and `find_value`**: These methods are used to extract data from XML nodes using XPath expressions.
- **`write_values`**: This method writes the extracted values to the CSV file. It handles date format conversion and ensures that certain text fields start with a "0" if required.
- **`convert_date_format`**: Converts date strings from `dd/mm/yyyy` format to `yyyy-mm-dd`.
- **`prepend_0`**: Ensures that certain identifiers start with a "0".
- **`close`**: Closes the CSV file after writing is complete.

---

## **2. `reader.py`**
This file is responsible for reading XML files, processing the data, and writing it to CSV files using the `Entity` class. Here's a breakdown of its key components:

### **`write` function**:
- This function writes data to the CSV files by calling the `write_values` method of the `Entity` class.
- It handles nested entities by recursively calling itself for L2 entities.

### **`get_entity` function**:
- This function creates an instance of the `Entity` class based on the configuration provided in `config.py`.

### **`fast_iter` function**:
- This function iterates over XML elements efficiently, processing each element and clearing it from memory to save resources.
- It uses the `process_element` function to handle different types of XML elements (e.g., `Enterprise`, `BusinessUnit`, etc.).

### **`process_element` function**:
- This function processes different XML elements based on their tags (e.g., `Enterprise`, `BusinessUnit`, `CancelledBusinessUnits`, etc.).
- It uses the `write` function to write data to the appropriate CSV files.

### **`convert_xml_to_csv` function**:
- This is the main function that orchestrates the conversion process.
- It initializes the `Entity` instances, processes the XML files, and writes the data to CSV files.

---

## **3. `config.py`**
This file contains configuration data for the entities and their corresponding XML tags, database column names, and data types. Here's a breakdown of its key components:

### **Data Types**:
- Defines constants for data types like `INT`, `TEXT`, `DATE`, etc.

### **Entity Configuration**:
- The `entity_config` dictionary contains configurations for various entities like `enterprise`, `header`, `denomination`, etc.
- Each entity configuration includes:
  - `parent_tags`: Tags that are inherited from parent entities.
  - `direct_tags`: Tags that are specific to the entity.
  - `xpath`: The XPath expression used to locate the entity in the XML file.
  - `l2_entities`: Nested entities that are children of the current entity.

### **Code Entity Configuration**:
- The `code_entity_config` dictionary contains configurations for code-related entities like `activitygroupcode`, `addresscode`, etc.
- These entities are used to map codes to their descriptions in different languages.

### **`get_list_of_columns_and_types` function**:
- This function generates a list of column names and their corresponding data types for a given entity.

---

## **4. `helper.py`**
This file sets up logging for the application. It configures both console and file logging with different log levels.

### **Logging Configuration**:
- Logs are written to both the console and a file (`application.log`).
- The console logs all levels (`DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`), while the file logs only `INFO`, `WARNING`, `ERROR`, and `CRITICAL`.

---

## **Interaction Between Files**
- **`reader.py`** uses the `Entity` class from `entity.py` to handle the extraction and writing of data.
- **`config.py`** provides the necessary configuration for `reader.py` and `entity.py` to map XML tags to database columns and data types.
- **`helper.py`** is used across the application for logging purposes.

---

## **Overall Workflow**
1. **Initialization**:
   - The `convert_xml_to_csv` function in `reader.py` initializes the `Entity` instances based on the configuration in `config.py`.

2. **XML Processing**:
   - The `fast_iter` function processes the XML files, extracting data using XPath expressions defined in `config.py`.
   - The `process_element` function handles different XML elements and writes the data to CSV files using the `Entity` class.

3. **Data Transformation**:
   - The `Entity` class handles the transformation of data (e.g., date format conversion, prepending "0" to identifiers) before writing it to the CSV files.

4. **Logging**:
   - The application logs its progress and any errors using the logger configured in `helper.py`.

---

## **Potential Improvements**
- **Error Handling**: The code could benefit from more robust error handling, especially in the `convert_date_format` and `prepend_0` methods, where exceptions are caught but not handled gracefully.
- **Performance**: The `fast_iter` function is designed for performance, but further optimizations could be explored for very large XML files.
- **Configuration Management**: The configuration in `config.py` is extensive and could be split into multiple files or managed using a configuration management tool.

---

## **Conclusion**
The provided code is a well-structured XML to CSV converter that handles complex nested XML structures. It uses a combination of XPath for data extraction, configuration files for mapping, and logging for monitoring the process. The code is modular, with clear separation of concerns between data extraction, transformation, and logging.
