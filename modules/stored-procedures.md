<!--
{
"name" : "stored-procedures",
"version" : "0.1",
"title" : "Stored Procedures",
"description": "Tutorial for VoltDB.",
"freshnessDate" : 2015-07-08,
"homepage" : "http://docs.voltdb.com/tutorial/",
"license" : "All Rights Reserved"
}
-->

<!-- @section -->

## Overview

We now have a complete database that we can interact with using SQL queries. For example, we can find the least populous county for any given state (California, for example) with the following SQL query:

```
$ sqlcmd
1> SELECT TOP 1 county, abbreviation, population
2>     FROM people, states WHERE people.state_num=6
3>     AND people.state_num=states.state_num
4>     ORDER BY population ASC;
```

However, typing in the same query with a different state number over and over again gets tiring very quickly. The situation gets worse as the queries get more complex.

<!-- @section -->

## Simple Stored Procedures

For queries you run frequently, only changing the input, you can create a simple stored procedure. Stored procedures let you define the query once and change the input values when you execute the procedure. Stored procedures have an additional benefit; because they are pre-compiled, the queries do not need to be planned at runtime, reducing the time it takes for each query to execute.

To create simple stored procedures — that is, procedures consisting of a single SQL query — you can define the entire procedure in your database schema using the CREATE PROCEDURE AS statement. So, for example to turn our previous query into a stored procedure, we can add the following statement to our schema:

```
CREATE PROCEDURE leastpopulated AS
  SELECT TOP 1 county, abbreviation, population
    FROM people, states WHERE people.state_num=?
    AND people.state_num=states.state_num
    ORDER BY population ASC;
```

In the CREATE PROCEDURE AS statement:

1. The label, in this case leastpopulated, is the name given to the stored procedure.
2. Question marks are used as placeholders for values that will be input at runtime.

In addition to creating the stored procedure, we can also specify if it is single-partitioned or not. When you partition a stored procedure, you associate it with a specific partition based on the table that it accesses. For example, the preceding query accesses the People table and, more importantly, narrows the focus to a specific value of the partitioning column, State_num.

Note that you can access more than one table in a single-partition procedure, as we do in the preceding example. However, all of the data you access must be in that partition. In other words, all the tables you access must be partitioned on the same key value or, for read-only SELECT statements, you can also include replicated tables.

So we can partition our new procedure on the People table by adding a separate PARTITION PROCEDURE statement. Or we can add the partitioning information as a clause of the CREATE PROCEDURE statement. Combining the procedure declaration and partitioning information can be more efficient and has additional benefits for complex procedures. So a combined statement is recommended:

```
CREATE PROCEDURE leastpopulated
   PARTITION ON TABLE people COLUMN state_num
AS
   SELECT TOP 1 county, abbreviation, population
     FROM people, states WHERE people.state_num=?
     AND people.state_num=states.state_num
     ORDER BY population ASC;
```

Now when we invoke the stored procedure, it is executed only in the partition where the State_num column matches the first argument to the procedure, leaving the other partitions free to process other requests.

Of course, before we can use the procedure we need to add it to the database. Modifying stored procedures can be done on the fly, like adding and removing tables. So we do not need to restart the database, just type the CREATE PARTITION statement at the sqlcmd prompt:

```
sqlcmd>
1> CREATE PROCEDURE leastpopulated
2>   PARTITION ON TABLE people COLUMN state_num
3> AS
4>    SELECT TOP 1 county, abbreviation, population
5>      FROM people, states WHERE people.state_num=?
6>      AND people.state_num=states.state_num
7>      ORDER BY population ASC;
```

Once we update the catalog, the new procedure becomes available. So we can now execute the query multiple times for different states simply by changing the argument to the procedure:

```
1> exec leastpopulated 6;
COUNTY         ABBREVIATION  POPULATION
-------------- ------------- -----------
Alpine County  CA                   1175


(1 row(s) affected)
2> exec leastpopulated 48;
COUNTY         ABBREVIATION  POPULATION
-------------- ------------- -----------
Loving County  TX                     82


(1 row(s) affected)
```

<!-- @section -->

## Writing More Powerful Stored Procedures

Simple stored procedures written purely in SQL are very handy as short cuts. However, some procedures are more complex, requiring multiple queries and additional computation based on query results. For more involved procedures, VoltDB supports writing stored procedures in Java.

It isn't necessary to be a Java programming wizard to write VoltDB stored procedures. All VoltDB stored procedures have the same basic structure. For example, the following code reproduces the simple stored procedure leastpopulated we wrote in the previous section using Java:

```
import org.voltdb.*;1

public class LeastPopulated extends VoltProcedure {2

  public final SQLStmt getLeast = new SQLStmt(
      " SELECT TOP 1 county, abbreviation, population "
    + " FROM people, states WHERE people.state_num=?"
    + " AND people.state_num=states.state_num"
    + " ORDER BY population ASC;" );3

  public VoltTable[] run(integer state_num)4
      throws VoltAbortException {

          voltQueueSQL( getLeast, state_num );5
          return voltExecuteSQL();6

      }
}
```

In this example:

1. We start by importing the necessary VoltDB classes and methods.
2. The procedure itself is defined as a Java class. The Java class name is the name we use at runtime to invoke the procedure. In this case, the procedure name is LeastPopulated.
3. At the beginning of the class, you declare the SQL queries that the stored procedure will use. Here we use the same SQL query from the simple stored procedure, including the use of a question mark as a placeholder.
4. The body of the procedure is a single run method. The arguments to the run method are the arguments that must be provided when invoking the procedure at runtime.
5. Within the run method, the procedure queues one or more queries, specifying the SQL query name, declared in step 3, and the arguments to be used for the placeholders. (Here we only have the one query with one argument, the state number.)
6. Finally, a call executes all of the queued queries and the results of those queries are returned to the calling application.

Now, writing a Java stored procedure to execute a single SQL query is overkill. But it does illustrate the basic structure of the procedure.

Java stored procedures become important when designing more complex interactions with the database. One of the most important aspects of VoltDB stored procedures is that each stored procedure is executed as a complete unit, a transaction, that either succeeds or fails as a whole. If any errors occur during the transaction, earlier queries in the transaction are rolled back before a response is returned to the calling application, or any further work is done by the partition.

One such transaction might be updating the database. It just so happens that the population data from the U.S. Census Bureau contains both actual census results and estimated population numbers for following years. If we want to update the database to replace the 2010 results with the 2011 estimated statistics (or some future estimates), we would need a procedure to:

1. Check to see if a record already exists for the specified state and county.
2. If so, use the SQL UPDATE statement to update the record.
3. If not, use an INSERT statement to create a new record.

We can do that by extending our original sample Java stored procedure. We can start be giving the Java class a descriptive name, UpdatePeople. Next we include the three SQL statements we will use (SELECT, UPDATE, and INSERT). We also need to add more arguments to the procedure to provide data for all of the columns in the People table. Finally, we add the query invocations and conditional logic needed. Note that we queue and execute the SELECT statement first, then evaluate its results (that is, whether there is at least one record or not) before queuing either the UPDATE or INSERT statement.

The following is the completed stored procedure source code.

```
import org.voltdb.*;

public class UpdatePeople extends VoltProcedure {

  public final SQLStmt findCurrent = new SQLStmt(
      " SELECT * FROM people WHERE state_num=? AND county_num=?;");
  public final SQLStmt updateExisting = new SQLStmt(
      " UPDATE people SET population=?"
    + " WHERE state_num=? AND county_num=?;");
  public final SQLStmt addNew = new SQLStmt(
      " INSERT INTO people VALUES (?,?,?,?);");

  public VoltTable[] run(byte state_num,
                         short county_num,
                         String county,
                         long population)
      throws VoltAbortException {

          voltQueueSQL( findCurrent, state_num, county_num );
          VoltTable[] results = voltExecuteSQL();

          if (results[0].getRowCount() > 0) { // found a record
             voltQueueSQL( updateExisting, population,
                                           state_num,
                                           county_num );

          } else {  // no existing record
             voltQueueSQL( addNew, state_num,
                                   county_num,
                                   county,
                                   population);

          }
          return voltExecuteSQL();
      }
}
```

<!-- @section -->

## Compiling Java Stored Procedures

Once we write the Java stored procedure, we need to load it into the database and then declare it in DDL the same way we do with simple stored procedures. But first, the Java class itself needs compiling. We use the Java compiler, javac, to compile the procedure the same way we would any other Java program.

When compiling stored procedures, the Java compiler must be able to find the VoltDB classes and methods imported at the beginning of the procedure. To do that, we must include the VoltDB libraries in the Java classpath. The libraries are in the subfolder /voltdb where you installed VoltDB. For example, if you installed VoltDB in the directory /opt/voltdb, the command to compile the UpdatePeople procedure is the following:

```
$ javac -cp "$CLASSPATH:/opt/voltdb/voltdb/*"  UpdatePeople.java
```

Once we compile the source code into a Java class, we need to package it (and any other Java stored procedures and classes the database uses) into a Jar file and load it into the database. Jar files are a standard format for compressing and packaging Java files. You use the jar command to create a Jar file, specifying the name of the Jar file to create and the files to include. For example, you can package the UpdatePeople.class file you created in the previous step into a Jar file named storedprocs.jar with the following command:

```
$ jar cvf storedprocs.jar *.class
```

Once you package the stored procedures into a Jar file, you can then load them into the database using the sqlcmd load classes directive. For example:

```
$ sqlcmd
1> load classes storedprocs.jar;
```

Finally, we can declare the stored procedure in our schema, in much the same way simple stored procedures are declared. But this time we use the CREATE PROCEDURE FROM CLASS statement, specifying the class name rather than the SQL query. We can also partition the procedure on the People table, since all of the queries are constrained to a specific value of State_num, the partitioning column. Here is the statement we add to the schema.

```
CREATE PROCEDURE
   PARTITION ON TABLE people COLUMN state_num
   FROM CLASS UpdatePeople;
```

Notice that you do not need to specify the name of the procedure after "CREATE PROCEDURE" because, unlike simple stored procedures, the CREATE PROCEDURE FROM CLASS statement takes the procedure name from the name of the class; in this case, UpdatePeople.

Go ahead and enter the CREATE PROCEDURE FROM CLASS statement at the sqlcmd prompt to bring your database up to date:

```
$ sqlcmd
1> CREATE PROCEDURE
2>   PARTITION ON TABLE people COLUMN state_num
3>   FROM CLASS UpdatePeople;
```

<!-- @section -->

## Putting it All Together

OK. Now we have a Java stored procedure and an updated schema. We are ready to try them out.

Obviously, we don't want to invoke our new procedure manually for each record in the People table. We could write a program to do it for us. Fortunately, there is a program already available that we can use.

The csvloader command normally uses the default INSERT procedures to load data into a table. However, you can specify an different procedure if you wish. So we can use csvloader to invoke our new procedure to update the database with every record in the data file.

First we must filter the data to the columns we need. We use the same shell commands we used to create the initial input file, except we switch to selecting the column with data for the 2011 estimate rather than the actual census results. We can save this file as data/people2011.txt (which is included with the source files):

```
$ grep -v "^040," data/CO-EST2011-Alldata.csv \
| cut --delimiter="," --fields=4,5,7,11 > data/people2011.txt
```

Before we update the database, let's just check to see which are the two counties with the smallest population:

```
$ sqlcmd
SQL Command :: localhost:21212
1> SELECT TOP 2 county, abbreviation, population
2> FROM people,states WHERE people.state_num=states.state_num
3> ORDER BY population ASC;
COUNTY          ABBREVIATION  POPULATION
--------------- ------------- -----------
Loving County   TX                     82
Kalawao County  HI                     90


(2 row(s) affected)
```

Now we can run csvloader to update the database, using the -p flag indicating that we are specifying a stored procedure name rather than a table name:

```
$ csvloader --skip 1 --file data/people2011.txt \
       -p UpdatePeople
```

And finally, we can check to see the results of the update by repeating our earlier query:

```
$ sqlcmd
SQL Command :: localhost:21212
1> SELECT TOP 2 county, abbreviation, population
2> FROM people,states WHERE people.state_num=states.state_num
3> ORDER BY population ASC;
COUNTY          ABBREVIATION  POPULATION
--------------- ------------- -----------
Kalawao County  HI                     90
Loving County   TX                     94


(2 row(s) affected)
```

Aha! In fact, the estimates show that Loving County, Texas is growing and is no longer the smallest!
