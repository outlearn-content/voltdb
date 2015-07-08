<!--
{
"name" : "schema-updates",
"version" : "0.1",
"title" : "Schema Updates and Durability",
"description": "Tutorial for VoltDB.",
"freshnessDate" : 2015-07-08,
"homepage" : "http://docs.voltdb.com/tutorial/",
"license" : "All Rights Reserved"
}
-->

<!-- @section -->

## Overview

Thus far in the tutorial we have restarted the database from scratch and reloaded the schema and data manually each time we changed the schema. This is sometimes the easiest way to make changes when you are first developing your application and making frequent changes. However, as your application — and the data it uses — becomes more complex, you want to maintain your database state across sessions.

You may have noticed that in the previous section of the tutorial we defined the States table but did not add it to the running database yet. That is because we want to demonstrate ways of modifying the database without having to start from scratch each time.

<!-- @section -->

## Preserving the Database

First let's talk about durability. VoltDB is an in-memory database. Each time you start the database with the create action, it creates a fresh, empty copy of the database. Obviously, in most real business situations you want the data to persist. VoltDB has several features that preserve the database contents across sessions. We will start by looking at snapshots.

> **Note:** The following examples make use of functionality available in the VoltDB Enterprise Edition only; specifically, the voltadmin save and restore commands and command logging. If you are using the VoltDB Community Edition you will need to reload the schema and data each time you restart the database process.

The easiest way to preserve the database is to use command logging, which is enabled by default for the VoltDB Enterprise Edition. Command logging logs all of the database activity, including schema and data changes, to disk. If the database ever stops, you can recover the command log simply by starting with the recover action rather than the create action.

If you are using the Enterprise Edition, try it now. Stop the database process with voltadmin shutdown, then use voltdb recover to restore the database to its previous state:

```
$ voltadmin shutdown
$ voltdb recover
```

Command logging makes saving and restoring your database easy and automatic. You can also save and restore your database manually, if you wish, using snapshots.

Snapshots are a complete disk-based representation of a VoltDB database, including everything needed to reproduce the database after a shutdown. You can create a snapshot of a running VoltDB database at anytime using the voltadmin save command. For example, from our tutorial directory, we can save the data to disk in a snapshot called "townsandpeople".

```
$ HERE=$(pwd)
$ voltadmin save $HERE/voltdbroot/snapshots/ "townsandpeople"
```

The arguments to the voltadmin save command are the directory where the snapshot files will be created and the name for the snapshot. Note that the save command requires an absolute path for the directory. In the preceding example, we assign the current working directory to a variable so the snapshot can be saved in the subfolder `./voltdbroot/snapshots`. VoltDB creates this folder by default when you start the database. We will learn more about it shortly.

Now that you have a copy of the database contents, we can stop and restart the database. Since we saved the snapshot into the `./voltdbroot/snapshots` folder, we can restore it automatically using the voltdb recover command, just as we did when using the command logs:

```
$ voltadmin shutdown
$ voltdb recover
```

When you specify recover as the startup action, VoltDB looks for and restores the most recent snapshot and/or command logs. Since the snapshot contains both the schema and the data, you do not need to reload the schema manually. With snapshots, you can also restore a previous known state of the database contents (rather than the most recent state) using the voltadmin restore command, as described in the Using VoltDB manual.

We can verify that the database was restored by doing some simple SQL queries in our other terminal session:

```
$ sqlcmd
SQL Command :: localhost:21212
1> select count(*) from towns;
C1
-------
 193297


(1 row(s) affected)
2> select count(*) from people;
C1
------
 81691


(1 row(s) affected)
```

<!-- @section -->

## Adding and Removing Tables

Now that we know how to save and restore the database, we can add the States table we defined in Part Three. Adding and dropping tables can be done "on the fly", while the database is running, using the sqlcmd utility. To add tables, you simply use the CREATE TABLE statement, like we did before. When modifying existing tables you can use the ALTER TABLE statement. Alternately, if you are not concerned with preserving existing data in the table, you can do a DROP TABLE followed by CREATE TABLE to replace the table definition.

In the case of the States table we are adding a new table so we can simply type (or copy & paste) the CREATE TABLE statement into the sqlcmd prompt. We can also use the show tables directive to verify that our new table has been added.

```
$ sqlcmd
SQL Command :: localhost:21212
1> CREATE TABLE states (
2>    abbreviation VARCHAR(20),
3>    state_num TINYINT,
4>    name VARCHAR(20),
5>    PRIMARY KEY (state_num)
6> );
Command successful
7> show tables;

--- User Tables --------------------------------------------
PEOPLE
STATES
TOWNS

--- User Views --------------------------------------------

--- User Export Streams --------------------------------------------

8> exit
```

Next we can load the state information from the data file. Finally, we can use the voltadmin save command to save a complete copy of the database.

```
$ csvloader --skip 1 -f data/states.csv states
$ HERE=$(pwd)
$ voltadmin save $HERE/voltdbroot/snapshots/  "states"
```

<!-- @section -->

## Updating Existing Tables

Now that we have a definitive lookup table for information about the states, we no longer need the redundant columns in the Towns and People tables. We want to keep the FIPS column, State_num, but can remove the State column from each table. Our updated schema for the two tables looks like this:

```
CREATE TABLE towns (
   town VARCHAR(64),
--   state VARCHAR(2),
   state_num TINYINT NOT NULL,
   county VARCHAR(64),
   county_num SMALLINT NOT NULL,
   elevation INTEGER
);
CREATE TABLE people (
  state_num TINYINT NOT NULL,
  county_num SMALLINT NOT NULL,
--  state VARCHAR(20),
  town VARCHAR(64),
  population INTEGER
);
```

It is good to have the complete schema on file in case we want to restart from scratch. (Or if we want to recreate the database on another server.) However, to modify an existing schema under development it is often easier just to use ALTER TABLE statements. So to modify the running database to match our new schema, we can use ALTER TABLE with the DROP COLUMN clause from the sqlcmd prompt:

```
$ sqlcmd
SQL Command :: localhost:21212
1> ALTER TABLE towns DROP COLUMN state;
Command successful
2> ALTER TABLE people DROP COLUMN state;
Command successful
```

Many schema changes, including adding, removing, and modifying tables, columns, and indexes can be done on the fly. However, there are a few limitations. For example, you cannot add new unique constraints to a table that already has data in it. In this case, you can DROP and CREATE the table with the new constraints if you do not need to save the data. Or, if you need to preserve the data and you know that it will not violate the new constraint, you can save the data to a snapshot, restart and reload the new schema, then restore the data from the snapshot into the updated schema using voltadmin restore.

This ends Part Four of the tutorial.
