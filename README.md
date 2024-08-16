# Dynamic Data Masking
This is a sample implementation of MySQL Enterprise Data Masking using custom procedures to distribute data across database schemas, masking sensitive data, and filtering data based on database roles and configurable rules. 
### Deploying metadata tables
Metadata tables are custom tables to 
```
-- metadata database schema is "mask"
create database if not exists mask;

-- mask.column_role stores list of table columns and role applies on each of column to mask data 
create table if not exists mask.column_role (table_schema char(100), table_name char(100), column_name char(100), role_name char(100));

-- mask.table_view stores list of target views created based on source table 
create table if not exists mask.table_view (source_table varchar(100), target_view varchar(100), column_definition varchar(10000), column_expression text);

-- mask.column_expression stores where clause / condition for each columns to filter data in the table
create table if not exists mask.column_expression (table_schema char(100), table_name char(100), column_name char(100), expression varchar(10000));

-- mask.other_expression stores free defined where clause / condition to filter data in the table
create table if not exists mask.other_expression (table_schema char(100), table_name char(100), expression text);
```
