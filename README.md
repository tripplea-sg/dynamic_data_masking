# Dynamic Data Masking
This is a sample implementation of MySQL Enterprise Data Masking using custom procedures to distribute data across database schemas, masking sensitive data, and apply rule based row access policy based on database roles and configurable rules. 
### Demo: Role based dynamic data masking and row access policies to filter data

### Deploying metadata tables
Metadata tables are custom tables to store table columns, roles, and access policies.
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
### Role based Data Masking
How to implement role based data masking
```
-- set source schema
set @schema = '<your source schema>';

-- set source table
set @table = '<your source table>';

-- map a table column with a database role
call mask.set_role('<table column>','<database role>');
```
Sample: mask revenue column in view where user does not set role "finance"
```
create role finance;
set @schema='lakehouse';
set @table='census';
call mask.set_role('revenue','finance');
```
Source code for mask.set_role:
```
drop procedure mask.set_role;
DELIMITER //
CREATE PROCEDURE mask.set_role (v_column_name char(100), v_role_name char(100))
BEGIN
	insert into mask.column_role (table_schema,table_name, column_name) select table_schema, table_name, column_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name,column_name) not in (select table_schema, table_name, column_name from mask.column_role);
	insert into mask.column_expression (table_schema,table_name, column_name) select table_schema, table_name, column_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name,column_name) not in (select table_schema, table_name, column_name from mask.column_expression);
	insert into mask.other_expression (table_schema,table_name) select distinct table_schema, table_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name) not in (select table_schema, table_name from mask.other_expression);
	update mask.column_role set role_name=v_role_name where table_schema=@schema and table_name=@table and column_name=v_column_name;
END
//
DELIMITER ;
```
### Rule based row access policy
How to implement rule based row access policy
```
-- set source schema
set @schema = '<your source schema>';

-- set source table
set @table = '<your source table>';

-- map a table column with a database role
call mask.set_where('<table column>','<where clause filtering in SQL for the column>');
```
Sample: restrict access to race other than "Other" in the view if user is not "admin"@'%'
```
set @schema='lakehouse';
set @table='census';
call mask.set_where('race','if(instr(user(),''admin'')>0,race,''Other'')');
```
Source code for mask.set_where:
```
drop procedure mask.set_where;
DELIMITER //
CREATE PROCEDURE mask.set_where (v_column_name char(100), v_expression text)
BEGIN
	insert into mask.column_expression (table_schema,table_name, column_name) select table_schema, table_name, column_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name,column_name) not in (select table_schema, table_name, column_name from mask.column_expression);
	insert into mask.column_role (table_schema,table_name, column_name) select table_schema, table_name, column_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name,column_name) not in (select table_schema, table_name, column_name from mask.column_role);
	insert into mask.other_expression (table_schema,table_name) select distinct table_schema, table_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name) not in (select table_schema, table_name from mask.other_expression);
	update mask.column_expression set expression=v_expression where table_schema=@schema and table_name=@table and column_name=v_column_name;
END
//
DELIMITER ;
```
### Other row access policy
Other row access policy is a free text to be added into "where" clause of the view
```
-- set source schema
set @schema = '<your source schema>';

-- set source table
set @table = '<your source table>';

-- set "where" clause condition
call mask.set_where_others('<where clause filtering in SQL for the column>');
```
Sample: restrict apps user to see record with native-country other than "United-States"
```
set @schema='lakehouse';
set @table='census';
call mask.set_where_others('native_country=if(instr(user(),''apps'')>0,''United-States'',native_country)');
```
Source code for mask.set_where_others:
```
drop procedure mask.set_where_others;
DELIMITER //
CREATE PROCEDURE mask.set_where_others (v_expression text)
BEGIN
	insert into mask.column_expression (table_schema,table_name, column_name) select table_schema, table_name, column_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name,column_name) not in (select table_schema, table_name, column_name from mask.column_expression);
	insert into mask.column_role (table_schema,table_name, column_name) select table_schema, table_name, column_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name,column_name) not in (select table_schema, table_name, column_name from mask.column_role);
	insert into mask.other_expression (table_schema,table_name) select distinct table_schema, table_name from information_schema.columns where table_schema=@schema and table_name=@table and (table_schema,table_name) not in (select table_schema, table_name from mask.other_expression);
	update mask.other_expression set expression=v_expression where table_schema=@schema and table_name=@table;
END
//
DELIMITER ;
```
### Print
Print is a command to show masking policy and row access policies
```
-- set source schema
set @schema = '<your source schema>';

-- set source table
set @table = '<your source table>';

-- print
call mask.print()
```
Sample:
```
set @schema='lakehouse';
set @table='census';
call mask.print();
```
Source code:
```
drop procedure mask.print;
DELIMITER //
CREATE PROCEDURE mask.print ()
BEGIN
	select a.column_name, a.role_name, (select b.expression from mask.column_expression b where b.table_schema=a.table_schema and b.table_name=a.table_name and b.column_name=a.column_name) expression from mask.column_role a where table_schema=@schema and table_name=@table;
	select a.expression from mask.other_expression a where table_schema=@schema and table_name=@table;
END
//
DELIMITER ;
```
### Commiting the View
Set variable @target as target schema where the view will be created
```
-- set source schema
set @schema = '<your source schema>';

-- set source table
set @table = '<your source table>';

-- set target schema
set @target = '<target schema>';

-- implement
call mask.push()
```
Sample: creating view apps.census based on lakehouse.census
```
set @schema='lakehouse';
set @table='census';
set @target='apps';
call mask.push();
```
Source code:
```
drop procedure mask.push;
DELIMITER //
CREATE PROCEDURE mask.push ()
BEGIN
	delete from mask.table_view where source_table=concat(@schema,'.',@table) and target_view=concat(@target,'.',@table);
	
	insert into mask.table_view (source_table, target_view, column_definition) select concat(@schema,'.',@table) source_table, concat(@target,'.',@table) target_view,group_concat(if(role_name is not null, concat('if(instr(current_role(),''',role_name,''')>0,',column_name,',mask_inner(cast(',column_name,' as char),1,1)) ',column_name), column_name)) column_list from mask.column_role where table_schema=@schema and table_name=@table group by table_schema, table_name;
	
	update mask.table_view set column_expression=(select group_concat(concat(column_name,'=',if(expression is null,column_name,expression)) separator ' and ')  from mask.column_expression a where a.table_schema=@schema and a.table_name=@table) where source_table=concat(@schema,'.',@table) and target_view=concat(@target,'.',@table); 

	set @other=(select if(isnull(expression)>0 or expression='','',concat(' and ',expression)) from mask.other_expression a where a.table_schema=@schema and a.table_name=@table);

	select concat('create or replace view ',@target,'.',@table,' as select ',column_definition,' from ',@schema,'.',@table,' where ', column_expression,@other) from mask.table_view where source_table=concat(@schema,'.',@table) and target_view=concat(@target,'.',@table) into @sql;

	PREPARE stmt FROM @sql;
    	EXECUTE stmt;
    	DEALLOCATE PREPARE stmt;
END
//
DELIMITER ;
```
