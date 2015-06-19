# ODK Tools
ODK Tools is a toolbox for processing [ODK](https://opendatakit.org/) survey data into MySQL databases. The toolbox relies on [Formhub](https://github.com/SEL-Columbia/formhub) as a temporary storage of ODK submissions because it conveniently stores them in JSON format; and on [META](https://github.com/ilri/meta) for storing the dictionary information and to support multiple languages. ODK Tools comprises of four command line tools performing different tasks. The toolbox is cross-platform and can be build in Windows, Linux and Mac. 

## The toolbox
- 
###ODKToMySQL
ODKToMySQL converts a ODK Excel File (XLSX survey file) into a relational MySQL schema. Even though [ODK Aggregate](https://opendatakit.org/use/aggregate/) stores submissions in MySQL, the Aggregate schema lacks basic functionality like:
 - Avoid duplicated submissions if an unique ID is used in a survey.
 - Store and link select values.
 - Store multi-select values as independent rows.
 - Relational links between repeats and sub-repeats.
 - In some cases data could end up not normalized.
 - No dictionary.
 - No multi-language support.

 ODKToMySQL creates a complete relational schemas with the following features:
 - A variable can be identified as a unique ID and becomes a primary key.
 - Select and multi-select values are stored in lookup tables. Lookup tables are then related to main tables.
 - Multi-select values are stored in sub tables where each value is recorded as an independent row.
 - Repeats are stored in independent tables sub-repeats are related to parent-repeats.
 - Repeats and variable names are stored in METAS's dictionary tables.
 - Multiple languages support.
 - Duplicated selects are avoided.
 - Yes/No selects are ignored
 - Tables with more than 100 fields (an indicator for not normalized data) are separated into sub-tables. This separation is done by the user using a separation file (created after running this tool).

 The tool produces the following output files:
 - create.sql. A [DDL](http://en.wikipedia.org/wiki/Data_definition_language) script containing all data structures, indexes and relationships.
 - insert.sql. A [DML](http://en.wikipedia.org/wiki/Data_manipulation_language) script that inserts all the select and multi-select values in the lookup tables.
 - uuid-triggers.sql. A script containing code for storing each row in each table with an [Universally unique identifier](http://en.wikipedia.org/wiki/Universally_unique_identifier) (UUID).
 - metadata.sql. A DML script that inserts the name of all tables and variables in META's dictionary tables.
 - iso639.sql. A DML script that inserts the name of all tables and variables and lookup values into META's language tables.
 - separation-TimeStamp.xml. This file is created when one or more tables might not be normalized (contains more than 100 fields). This separation file can be edited to normalize the tables and then used as an input file for this tool.
 - manifest.xml. This file maps each variable in the ODK survey with its corresponding field in a table in the MySQL database. **This file is used in subsequent tools.**

  #### *Parameters*
  - x - Input ODK XLSX file.
  - v - Main survey variable. This is the unique ID of each survey. **This IS NOT the ODK survey ID found in settings.** 
  - t - Main table. Name of the master table for the target schema. ODK surveys do not have a master table however this is necessary to store ODK variables that are not inside a repeat. **If the main survey variable is store inside a repeat then the main table is such repeat.**
  - d - Default storing language. For example: (en)English. **This is the default language for the database and might be different as the default language in the ODK survey.** If not indicated then English will be assumed.
  - l - Other languages. For example: (fr)French,(es)Español. Required if the ODK file has multiple languages.
  - y - Yes and No strings in the default language in the format "String|String". This will allow the tool to identify Yes/No lookup tables and exclude them. **It is not case sensitive.** For example, if the default language is Spanish then this value should be indicated as "Si|No". If its empty then English "Yes|No" will be assumed.
  - p - Table prefix. A prefix that can be added to each table. This is useful if a common schema is used to store different surveys.
  - c - Output DDL file. "create.sql" by default.
  - i - Output DML file. "insert.sql" by default.
  - u - Output UUID trigger file. "uuid-triggers.sql" by default.  
  - m - Output metadata file. "metadata.sql" by default.
  - T - Output translation file. "iso639.sql" by default. 
  - f - Output manifest file. "manifest.xml" by default
  - s - Input separation file. This file might be generated by this tool.
  

 ### *Example for a single language ODK*
  ```sh
$ ./odktomysql -x my_input_xlsx_file.xlsx -v QID -t maintable
```
 ### *Example for a multi-language ODK (English and Español) where English is the default storing language*
  ```sh
$ ./odktomysql -x my_input_xlsx_file.xlsx -v main_questionarie_ID -t maintable -l "(es)Español"
```
 ### *Example for a multi-language ODK (English and Español) where Spanish is the default storing language*
  ```sh
$ ./odktomysql -x my_input_xlsx_file.xlsx -v main_questionarie_ID -t maintable -d "(es)Español" -l "(en)English" -y "Si|No"
```

- 
###FormhubToJSON
In Formhub data coming from mobile devices is stored in a [Mongo](https://www.mongodb.org/) Database. Although Formhub provides exporting functions to CSV and MS Excel it does not provide exporting to more interoperable formats like [JSON](http://en.wikipedia.org/wiki/JSON). FormhubToJSON is a small Python program that extracts Formhub survey data from MongoDB to JSON files. Each data submission is exported as a JSON file using Formhub submission UUID as file name.
#### *Parameters*
  - m - URI for the Mongo Server. For example mongodb://localhost:2701
  - d - Formhub database. "formhub" by default.
  - c - Formhub collection storing the surveys. "instances" by default.
  - y - ODK Survey ID. This is found in the "settings" sheet of the ODK XLSX file.
  - o - Output directory. "./output" by default (created if it does not exists).
  - w - Overwrite JSON file if exists. False by default.
  
 ### *Example*
  ```sh
   $ python formhubtojson.py -m "mongodb://my_mongoDB_server:27017/" -y my_survey_id -o ./my_output_directory
```

- 
###JSONToMySQL
JSONToMySQL imports JSON files (generated by **FormhubToJSON**) into a MySQL schema generated by **ODKToMySQL**. The tool imports one file at a time and requires a manifest file (see **ODKToMySQL**). The tool generates an error log file in CSV format.
#### *Parameters*
  - H - MySQL host server. Default is localhost
  - P - MySQL port. Default 3306
  - s - Schema to be converted
  - u - User who has select access to the schema
  - p - Password of the user
  - m - Input manifest file.
  - j - Input JSON file.
  - o - Output log file. "output.csv" by default.

 ### *Example*
  ```sh
$ ./jsontomysql -H my_MySQL_server -u my_user -p my_pass -s my_schema -m ./my_manifest_file.xml -j ./my_JSON_file.json -o my_log_file.csv
```

- 
###ODKDataToMySQL
ODKDataToMySQL imports ODK XML data files (device data) into a MySQL schema generated by **ODKToMySQL**. The tool imports one file at a time and requires a manifest file (see **ODKToMySQL**). The tool generates an error log file in CSV format. 
#### *Parameters*
  - H - MySQL host server. "localhost" by default
  - P - MySQL port. "3306" by default.
  - s - Schema to be converted.
  - u - User who has insert access to the schema.
  - p - Password of the user.
  - m - Input manifest file.
  - d - Input ODK XML file.
  - o - Output log file. "output.csv" by default.

 ### *Example*
  ```sh
$ ./odkdatatomysql -H my_MySQL_server -u my_user -p my_pass -s my_schema -m ./my_manifest_file.xml -d ./my_ODK_XML_Data_file.xml -o my_log_file.csv
```

## Technology
ODK Tools was built using:

- [C++](https://isocpp.org/), a general-purpose programming language.
- [Qt 5](https://www.qt.io/), a cross-platform application framework.
- [Python 2.7.x](https://www.python.org/), a widely used general-purpose programming language. 
- [TClap](http://tclap.sourceforge.net/), a small, flexible library that provides a simple interface for defining and accessing command line arguments.
- [Qt XLSX](https://github.com/dbzhang800/QtXlsxWriter), a XLSX file reader and writer for Qt5.


## Building and testing
To build this site for local viewing or development:

    $ git clone https://github.com/ilri/odktools.git
    $ cd odktools
    $ qmake
    $ make

## Author
Carlos Quiros (c.f.quiros_at_cgiar.org / cquiros_at_qlands.com)

Harrison Njmaba (h.njamba_at_cgiar.org)

## License
This repository contains the code of:

- [TClap](http://tclap.sourceforge.net/) which is licensed under the [MIT license](https://raw.githubusercontent.com/twbs/bootstrap/master/LICENSE).
- [Qt XLSX](https://github.com/dbzhang800/QtXlsxWriter) which is licensed under the [MIT license](https://raw.githubusercontent.com/twbs/bootstrap/master/LICENSE).

Otherwise, the contents of this application is [GPL V3](http://www.gnu.org/copyleft/gpl.html). 
