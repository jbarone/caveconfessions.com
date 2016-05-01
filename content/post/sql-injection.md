+++
comments = true
date = "2016-05-01T00:00:00-05:00"
draft = false
image = ""
share = true
slug = "sql-injection"
tags = ["Penetration Testing", "SQL Injection"]
categories = ["Penetration Testing", "SQL Injection"]
title = "The Web's Most Wanted"

+++

OWASP has a top ten list that ranks the most critical attacks against web
applications. At the top of this list is Injection Attacks. SQL injection is
one of this type of attacks. This post is a walk through of what the attack is
and a look at more advanced versions of the attack.

<!--more-->

## SQL Primer

The Structured Query Language (SQL) is the language of databases. This is the
language that humans use to talk to databases to get information out or put
information in. It should be noted that SQL is supported by most all Relational
Database Management Systems (RDBMS). These include
[MySQL](https://www.mysql.com), [MSSQL](http://www.microsoft.com/sqlserver/),
[PostgreSQL](http://www.postgresql.org/), [SQLite](https://www.sqlite.org/),
and [Oracle](https://www.oracle.com/database/index.html) to name a few of the
more popular ones.

### Database Structure

A database is an organized collection of data. The data itself is located in
tables within the database. There are other items within a databases, but for
our purposes here we are only concerned with the data in the tables.

A table is a collection of related data held in a structured format. The data
is divided by vertical columns, or fields. Each record of data is a row in the
table. A table has a limited and specified number of columns, but an unlimited
number of rows possible.

### SQL Commands

SQL provides several commands for altering and querying the database for the
purposes of this primer we will focus on four main commands SELECT, INSERT,
UPDATE, DELETE. These commands can be split into two categories based on what
effect they have on the data.

#### Modify Data

These commands are used to add / change / remove data from a table in a
database. These commands are INSERT, UPDATE, and DELETE respectively.

##### INSERT

This command is used to add data into table. The grammar for this command is:

```sql
INSERT INTO table_name [( column_identifier [, column_identifier]...)] VALUES (insert_value[, insert_value]...)
```

<dl>
    <dt>table_name</dt>
    <dd>The name of the table where the data is being inserted</dd>
    <dt>column_identifier</dt>
    <dd>The name of the column to map the inserted value to</dd>
    <dt>insert_value</dt>
    <dd>The value to be inserted into the column identified with the same index
    as the value to be inserted.</dd>
</dl>

Examples:

```sql
INSERT INTO users VALUES (1, 'bob', 'password');
INSERT INTO example (id, name, description) VALUES (42, 'Tart', 'It is so yummy');
```

##### UPDATE

This command is used to change data in a table. It's grammar is:

```sql
UPDATE table_name
     SET column_identifier = {expression | NULL }
          [, column_identifier = {expression | NULL}]...
     [WHERE search_condition]
```

<dl>
    <dt>table_name</dt>
    <dd>The name of the table where the data is being inserted</dd>
    <dt>column_identifier</dt>
    <dd>The name of the column whose value is being changed</dd>
    <dt>expression</dt>
    <dd>This is the value that column is being set to.</dd>
    <dt>search_condition</dt>
    <dd>This is an expression that is used to filter the table so only
    specified rows are altered.</dd>
</dl>

Examples:

```sql
UPDATE users SET username = 'bobby' WHERE id = 1;
```

##### DELETE

This command is used to remove a row from a table. The grammar for this command
is:

```sql
DELETE FROM table_name [WHERE search_condition]
```

<dl>
    <dt>table_name</dt>
    <dd>The name of the table where the data is being inserted</dd>
    <dt>serch_condition</dt>
    <dd>This is an expression that is used to filter the table so only
    specified rows are removed.</dd>
</dl>

#### Retrieve Data

This is the heart of SQL. It is all well and good to modify the data in a
database but what people really want is the ability to query that data, and
generate pretty reports. To actually get the data out of the database the all
powerful SELECT statement is needed.

##### SELECT

This command is truely the work horse command of the Structured Query Language.
Which also means that the grammar for this command can be immensely complicated.
The simplest form of the grammar for this command is:

```sql
SELECT [ALL | DISTINCT] select_list
     FROM table_list
          [WHERE search_condition]
               [order_by_clause]
```

<dl>
    <dt>select_list</dt>
    <dd>This is a comma seperated list of the columns that are to be returned,
    or the wildcard (*) if all columns are to be returned.</dd>
    <dt>table_list</dt>
    <dd>This is a comma seperated list of tables to use for the data source.
    This could be made even more complicated by including JOIN statements.
    (due to the complexity of joining tables, it will be left to the reader to
    learn about them on their own)</dd>
    <dt>search_condition</dt>
    <dd>This is an expression that is used to filter the table(s) so only
    specified rows are returned.</dd>
    <dt>order_by_clause</dt>
    <dd>This specifies the colums to be sorted on, and in which direction</dd>
</dl>

Examples:

```sql
SELECT * FROM example_table;
SELECT id, username, password FROM users WHERE id = 1;
SELECT id, price, name FROM items ORDER BY price ASC;
```

##### UNION

One more SQL command that needs to be understood is UNION. This command is used
to combine the output of multiple SELECT statements into one result set. The
big thing to keep in mind with union queries is that the queries being combined
need to be returning the same number of columns, and the order will matter
because the data types need to match or be convertable.

```sql
select_query UNION [ALL] select_query
```

<dl>
    <dt>select_query</dt>
    <dd>This is a valid SELECT query</dd>
</dl>

Examples:

```sql
SELECT id, name FROM users UNION SELECT id, name FROM employees;
```

## SQL Injection

SQL Injection attacks come about when code uses unvalidated and/or unsanitized
to dynamically construct queries that are sent to the backend database. When
this happens, the risk is now open to be attacked by malicious actors. So what
exactly does this look like?

```php
<?php
if(isset($_POST['Submit']){
    $user = $_POST['user'];
    $pass = $_POST['password'];
    $re = mysql_query(
        "SELECT * FROM users " .
        "WHERE user_name = '$user' AND password = '$pass'"
    );

    if(mysql_num_rows($re) == 0){
        print 'Incorrect username or password';
    }else{
        print 'Welcome';
    }
}
?>
```

In this example code we can see that the username and the password are plucked
directly from the POST and place right into the query that is being executed.
This problem can be demonstrated easily by thinking about what happens when
malicious content is POSTed. For Examples:

    user: admin
    pass: ' OR '1'='1

If this is what is sent to the page above then there is a problem. Let's look
at what the created query actually is with these values.

```sql
SELECT * FROM users WHERE user_name = 'admin' AND password = '' OR '1'='1'
```

We can see that event with the extra single quotes (') in the pass value, the
created query is still valid and will execute. So what is this query asking?

1. Is the user_name 'admin'
2. Also make sure that the password is blank, or 1 is equal to 1.

With this question, the password should never be blank, but 1 is always equal
to 1, so the password check has essentailly been negated. If there is a user
that has the name 'admin' then, that is the row that will be returned. This is
a classic example of authentication bypass using SQL Injection.

In this world of data-driven web applications there are cases where the data
being retrieved from the database is displayed back on the page. When this data
is being filtered in some way based on user input, or input that could be
manipulated by a user, then the circumstances create a situation where an
attacker could query and alter the data at whim.


## Next Time

This is the first in a series of post that will dive into the wonderful world
of SQL injection. The next will focus on UNION based SQL Injections which is
a way to easily see the information that is being extracted.
