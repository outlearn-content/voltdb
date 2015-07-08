<!--
{
"name" : "creating",
"version" : "0.1",
"title" : "Creating the Database",
"description": "In VoltDB you define your database schema using SQL data definition language (DDL) statements just like other SQL databases.",
"freshnessDate" : 2015-07-08,
"homepage" : "http://docs.voltdb.com/tutorial/",
"license" : "All Rights Reserved"
}
-->

<!-- @section -->

## Overview

In VoltDB you define your database schema using SQL data definition language (DDL) statements just like other SQL databases. So, if we want to create a database table for the places where we live, the DDL schema might look like the following:

```
CREATE TABLE towns (
   town VARCHAR(64),
   county VARCHAR(64),
   state VARCHAR(2)
);
```

The preceding schema defines a single table with three columns: town, county, and state. We could also set options, such as default values and primary keys. But for now we will keep it as simple as possible.

<!-- @section -->

## Starting the Database and Loading the Schema

Once you have the schema defined, you can start the database and load your schema. There are several options available when starting a VoltDB database, which we will discuss later. But for now, we can use the simplest command to start the database using the default options on the current machine with the following command:

```
$ voltdb create
```

The voltdb create command tells VoltDB to create a new, empty database. Once startup completes, the server reports the following message:

```
Server completed initialization.
```

Now you are ready to load your schema. To do that you use the VoltDB interactive command line utility, sqlcmd. Create a new terminal window and issue the sqlcmd command from the shell prompt:

```
$ sqlcmd
SQL Command :: localhost:21212
1>
```

The VoltDB interactive SQL command line first reports what database it has connected to and then puts up a numbered prompt. At the prompt, you can enter DDL statements and SQL queries, execute stored procedures, or type "exit" to end the program and return to the shell prompt.

To load the schema, you can either type the DDL schema statements by hand or, if you have them in a file, you can use the FILE directive to process all of the DDL statements with a single command. Since we only have one table definition, we can type or cut & paste the DDL directly into sqlcmd:

```
1> CREATE TABLE towns (
2>   town VARCHAR(64),
3>   county VARCHAR(64),
4>   state VARCHAR(2)
5> );
```

<!-- @section -->

## Using SQL Queries

Congratulations! You have created your first VoltDB database. Of course, an empty database is not terribly useful. So the first thing you will want to do is create and retrieve a few records to prove to yourself that the database is running as you expect.

VoltDB supports all of the standard SQL query statements, such as INSERT, UPDATE, DELETE, and SELECT. You can invoke queries programmatically, through standard interfaces such as JDBC and JSON, or you can include them in stored procedures that are compiled and loaded into the database.

But for now, we'll just try some ad hoc queries using sqlcmd. Let's start by creating records using the INSERT statement. The following example creates three records, for the towns of Billerica, Buffalo, and Bay View. Be sure to include the semi-colon after each statement.

```
1> insert into towns values ('Billerica','Middlesex','MA');
2> insert into towns values ('Buffalo','Erie','NY');
3> insert into towns values ('Bay View','Erie','OH');
```

We can also use ad hoc queries to verify that our inserts worked as expected. The following queries use the SELECT statement to retrieve information about the database records.

```
4> select count(*) as total from towns;
TOTAL
------
     3

(1 row(s) affected)
5> select town, state from towns ORDER BY town;
TOWN         STATE
------------ ------
Bay View     OH
Billerica    MA
Buffalo      NY

(3 row(s) affected)
```

When you are done working with the database, you can type "exit" to end the sqlcmd session and return to the shell command prompt. Then switch back to the terminal session where you started the database and press CTRL-C to end the database process.

This ends Part One of the tutorial.
