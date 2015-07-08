<!--
{
"name" : "partitioning",
"version" : "0.1",
"title" : "Partitioning",
"description": "Tutorial for VoltDB.",
"freshnessDate" : 2015-07-08,
"homepage" : "http://docs.voltdb.com/tutorial/",
"license" : "All Rights Reserved"
}
-->

<!-- @section -->

## Overview

Now you have the hang of the basic features of VoltDB as a relational database, it's time to start looking at what makes VoltDB unique. One of the most important features of VoltDB is partitioning.

*Partitioning* organizes the contents of a database table into separate autonomous units. Similar to sharding, VoltDB partitioning is unique because:

* VoltDB partitions the database tables automatically, based on a partitioning column you specify. You do not have to manually manage the partitions.
* You can have multiple partitions, or sites, on a single server. In other words, partitioning is not just for scaling the data volume, it helps performance as well.
* VoltDB partitions both the data and the processing that accesses that data, which is how VoltDB leverages the throughput improvements parallelism provides.

<!-- @section -->

## Partitioned Tables

You partition a table by specifying the partitioning column as part of your schema. If a table is partitioned, each time you insert a row into that table, VoltDB decides which partition the row goes into based on the value of the partitioning column. So, for example, if you partition the Towns table on the column Name, the records for all towns with the same name end up in the same partition.

However, although partitioning by name may be reasonable in terms of evenly distributing the records, the goal of partitioning is to distribute both the data and the processing. We don't often compare information about towns with the same name. Whereas, comparing towns within a given geographic region is very common. So let's partition the records by state so we can quickly do things like finding the largest or highest town within a given state.

Both the Towns and the People tables have columns for the state name. However, they are slightly different; one uses the state abbreviation and one uses the full name. To be consistent, we can use the State_num column instead, which is common to both tables.

To partition the tables, we simply add a PARTITION TABLE statement to the database schema. Here are the statements we can add to our schema to partition both tables by the State_num column:

```
PARTITION TABLE towns ON COLUMN state_num;
PARTITION TABLE people ON COLUMN state_num;
```

Having added partitioning information, we can stop the database, restart and reload the schema and data. This time, rather than using CTRL-C to kill the database process, we can use the voltadmin shutdown command. The voltadmin commands perform administrative functions for a database cluster and shutdown performs an orderly shutdown of the database whether a single node or a 15 node cluster. So go to the second terminal session and use voltadmin shutdown to stop the database:

```
$ voltadmin shutdown
```

Then you can restart the database and load the new schema and data files:

```
  [terminal 1]
$ voltdb create

  [terminal 2]
$ sqlcmd
1> FILE towns.sql;
Command succeeded.
2> exit
$ cd data
$ csvloader --separator "|"  --skip 1   \
             --file towns.txt  towns
$ csvloader --file people.txt --skip 1 people
```

The first thing you might notice, without doing any other queries, is that loading the data files is faster. In fact, when csvloader runs, it creates three log files summarizing the results of the loading process. One of these files, `csvloader_TABLE-NAME_insert_report.log`, describes how long the process took and the average transactions per second (TPS). Comparing the load times before and after adding partitioning shows that adding partitioning increases the ingestion rate for the Towns table from approximately 5,000 to 16,000 TPS — more than three times as fast! This performance improvement is a result of parallelizing the stored procedure calls across eight sites per host. Increasing the number of sites per host can provide additional improvements, assuming the server has the core processors necessary to manage the additional threads.

<!-- @section -->

## Replicated Tables

As mentioned earlier, the two tables Towns and People both have a VARCHAR column for the state name, but its use is not consistent. Instead we use the State_num column to do partitioning and joining of the two tables.

The State_num column contains the FIPS number. That is, a federal standardized identifier assigned to each state. The FIPS number ensures unique and consistent identification of the state. However, as useful as the FIPS number is for computation, most people think of their location by name, not number. So it would be useful to have a consistent name to go along with the number.

Instead of attempting to modify the fields in the individual tables, we can normalize our schema and create a separate table that provides an authoritative state name for each state number. Again, the federal government makes this information freely available from the U.S. Environmental Protection Agency web site, http://www.epa.gov/enviro/html/codes/state.html. Although it is not directly downloadable as a data file, a copy of the FIPS numbers and names for all of the states is included in the tutorial files in the data subfolder as `data/state.txt`.

So let's go and add a table definition for this data to our schema:

```
CREATE TABLE states (
   abbreviation VARCHAR(20),
   state_num TINYINT,
   name VARCHAR(20),
   PRIMARY KEY (state_num)
);
```

This sort of lookup table is very common in relational databases. They reduce redundancy and ensure data consistency. Two of the most common attributes of lookup tables are that they are relatively small in size and they are static. That is, they are primarily read-only.

It would be possible to partition the States table on the State_num column, like we do the Towns and People tables. However, when a table is relatively small and not updated frequently, it is better to replicate it to all partitions. This way, even if another table is partitioned (such as a customer table partitioned on last name), stored procedures can join the two tables, no matter what partition the procedure executes in.

Tables where all the records appear in all the partitions are called replicated tables. Note that tables are replicated by default. So to make the States table a replicated table, we simply include the CREATE TABLE statement without an accompanying PARTITION TABLE statement.

One last caveat concerning replicated tables: the benefits of having the data replicated in all partitions is that it can be read from any individual partition. However, the deficit is that any updates or inserts to a replicated table must be executed in all partitions at once. This sort of multi-partition procedure reduces the benefits of parallel processing and impacts throughput. Which is why you should not replicate tables that are frequently updated.

This ends Part Three of the tutorial.
