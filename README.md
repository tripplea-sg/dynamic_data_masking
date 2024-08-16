# Dynamic Data Masking
This is a sample implementation of MySQL Enterprise Data Masking using custom procedures to distribute data across database schemas, masking sensitive data, and filtering data based on database roles and configurable rules. 
### Deploying metadata tables
```
create database if not exists mask;

create table if not exists mask.column_role (table_schema char(100), table_name char(100), column_name char(100), role_name char(100));

create table if not exists mask.table_view (source_table varchar(100), target_view varchar(100), column_definition varchar(10000), column_expression text);

create table if not exists mask.column_expression (table_schema char(100), table_name char(100), column_name char(100), expression varchar(10000));

create table if not exists mask.other_expression (table_schema char(100), table_name char(100), expression text);
```
