<!--
{
"name" : "loading",
"version" : "0.1",
"title" : "Loading and Managing Data",
"description": "Learn more about powerful tools to handle data in VoltDB.",
"freshnessDate" : 2015-07-08,
"homepage" : "http://docs.voltdb.com/tutorial/",
"license" : "All Rights Reserved"
}
-->

<!-- @section -->

## Overview

As useful as ad hoc queries are, typing in data by hand is not very efficient. Fortunately, VoltDB provides several features to help automate this process.

When you define tables using the CREATE TABLE statement, VoltDB automatically creates stored procedures to insert records for each table. There is also a command line tool that uses these default stored procedures so you can load data files into your database with a single command. The **csvloader** command reads a data file, such as a comma-separated value (CSV) file, and writes each entry as a record in the specified database table using the default insert procedure.

It just so happens that there is data readily available for towns and other landmarks in the United States. The Geographic Names Information Service (GNIS), part of the U.S. Geological Survey, provides data files of officially named locations throughout the United States. In particular, we are interested in the data file for populated places. This data is available as a text file from their web site, http://geonames.usgs.gov/domestic/download_data.htm.

<!-- @link, "url" : "http://geonames.usgs.gov/domestic/download_data.htm", "text": "Download data from GNIS" -->

The information provided by GNIS not only includes the name, county and state, it includes each location's position (latitude and longitude) and elevation. Since we don't need all of the information provided, we can reduce the number of columns to only the information we want.

For our purposes, let's use information about the name, county, state, and elevation of any populated places. This means we can go back and edit our schema file, `towns.sql`, to add the new columns:

```
CREATE TABLE towns (
   town VARCHAR(64),
   state VARCHAR(2),
   state_num TINYINT NOT NULL,
   county VARCHAR(64),
   county_num SMALLINT NOT NULL,
   elevation INTEGER
);
```

Note that the GNIS data includes both names and numbers for identifying counties and states. We rearranged the columns to match the order in which they appear in the data file. This makes loading the data easier since, by default, the **csvloader** command assumes the data columns are in the same order as specified in the schema.

Finally, we can use some shell magic to trim out the unwanted columns and rows from the data file. The following script selects the desired columns and removes any records with empty fields:

```
$ cut --delimiter="|" --fields=2,4-7,16 POP_PLACES_20120801.txt \
  | grep -v "|$" \
  | grep -v "||" > data/towns.txt
```

To save time and space, the resulting file containing only the data we need is included with the tutorial files in a subfolder as `data/towns.txt`.

<!-- @task, "text" : "Save the data into data/towns.txt using the script provided."-->

<!-- @section -->

## Restarting the Database

Because we changed the schema and reordered the columns of the Towns table, we want to start over with an empty database and reload the schema. Alternately, if the database is still running you could do a DROP TABLE and CREATE TABLE to delete any existing data and replace the table definition.

Later we'll learn how to recover the database to its last state. But for now, we'll use the same commands you learned earlier to create an empty database in one terminal session and load the schema using sqlcmd in another. This time we will load the schema from our DDL file using the FILE directive:

```
  [terminal 1]
$ voltdb create

  [terminal 2]
$ sqlcmd
1> FILE towns.sql;
Command succeeded.
2> exit
```

<!-- @task, "text" : "Restart the database."-->

<!-- @section -->

## Loading the Data

Once the database is running and the schema loaded, we are ready to insert our new data. To do this, set default to the `/data` subfolder in your tutorial directory, and use the **csvloader** command to load the data file:

```
$ cd data
$ csvloader --separator "|"   --skip 1   \
           --file towns.txt  towns
```

In the preceding commands:

1. The `--separator` flag lets you specify the character separating the individual data entries. Since the GNIS data is not a standard CSV, we use `--separator` to identify the correct delimiter.
2. The data file includes a line with column headings. The `--skip 1` flag tells csvloader to skip the first line.
3. The `--file` flag tells csvloader what file to use as input. If you do not specify a file, csvloader uses standard input as the source for the data.
4. The argument, `towns`, tells csvloader which database table to load the data into.

The csvloader loads all of the records into the database and it generates three log files: one listing any errors that occurred, one listing any records it could not load from the data file, and a summary report including statistics on how long the loading process took and how many records were loaded.

<!-- @task, "text" : "Insert the new data."-->

<!-- @section -->

## Querying the Database

Now that we have real data, we can perform more interesting queries. For example, which towns are at the highest elevation, or how many locations in the United States have the same name?

```
$ sqlcmd
1> SELECT town,state,elevation from towns order by elevation desc limit 5;
TOWN                      STATE  ELEVATION
------------------------- ------ ----------
Corona (historical)       CO           3573
Quartzville (historical)  CO           3529
Logtown (historical)      CO           3524
Tomboy (historical)       CO           3508
Rexford (historical)      CO           3484


(5 row(s) affected)
2> select town, count(town) as duplicates from towns
3> group by town order by duplicates desc limit 5;
TOWN            DUPLICATES
--------------- -----------
Midway                  215
Fairview                213
Oak Grove               167
Five Points             150
Riverside               130


(5 row(s) affected)
```

As we can see, the five highest towns are all what appear to be abandoned mining towns in the Rocky Mountains. And Springfield, as common as it is, doesn't make it into the top five named places in the United States.

We can make even more interesting discoveries when we combine data. We already have information about locations and elevation. The US Census Bureau can also provide us with information about population density. Population data for individual towns and counties in the United States can be downloaded from their web site, http://www.census.gov/popest/data/index.html.

To add the new data, we must add a new table to the database. So let's edit our schema to add a table for population that we will call people. While we are at it, we can create indexes for both tables, using the columns that will be used most frequently for searching and sorting, state_num and county_num.

```
CREATE TABLE towns (
   town VARCHAR(64),
   state VARCHAR(2),
   state_num TINYINT NOT NULL,
   county VARCHAR(64),
   county_num SMALLINT NOT NULL,
   elevation INTEGER
);
CREATE TABLE people (
  state_num TINYINT NOT NULL,
  county_num SMALLINT NOT NULL,
  state VARCHAR(20),
  county VARCHAR(64),
  population INTEGER
);
CREATE INDEX town_idx ON towns (state_num, county_num);
CREATE INDEX people_idx ON people (state_num, county_num);
```

Once again, we put the columns in the same order as they appear in the data file. We also need to trim the data file to remove extraneous columns. The census bureau data includes both measured and estimated values. For the tutorial, we will focus on one population number.

The shell command to trim the data file is the following. (Again, the resulting data file is available as part of the downloadable tutorial package.)

```
$ grep -v "^040," CO-EST2011-Alldata.csv \
     | cut --delimiter="," --fields=4-8  > people.txt
```

Once we have the data and the new DDL statements, we can update the database schema. We could stop and restart the database, load the new schema from our text file and reload the data. But we don't have to. Since we are not changing the Towns table or adding a unique index, we can make our changes to the running database by simply cutting and pasting the new DDL statements into sqlcmd:

```
$ sqlcmd
1> CREATE TABLE people (
2>  state_num TINYINT NOT NULL,
3>  county_num SMALLINT NOT NULL,
4>  state VARCHAR(20),
5>  county VARCHAR(64),
6>  population INTEGER
7> );
8> CREATE INDEX town_idx ON towns (state_num, county_num);
9> CREATE INDEX people_idx ON people (state_num, county_num);
```

Once we create the new table and indexes, we can load the accompanying data file:

```
$ cd data
$ csvloader --file people.txt --skip 1 people
```

At this point we now have two tables loaded with data. Now we can join the tables to look for correlations between elevation and population, like so:

```
$ sqlcmd
1> select top 5 min(t.elevation) as height,
2> t.state,t.county, max(p.population)
3> from towns as t, people as p
4> where t.state_num=p.state_num and t.county_num=p.county_num
5> group by t.state, t.county order by height desc;
HEIGHT  STATE  COUNTY    C4
------- ------ --------- ------
   2754 CO     Lake        7310
   2640 CO     Hinsdale     843
   2609 CO     Mineral      712
   2523 CO     San Juan     699
   2454 CO     Summit     27994

(5 row(s) affected)
```

It turns out that, even discounting ghost towns that have no population, the five inhabited counties with highest elevation are all in Colorado. In fact, if we reverse the select expression to find the lowest inhabited counties (by changing the sort order from descending to ascending), the lowest is Inyo county in California — the home of Death Valley!

<!-- @task, "text" : "Run some more queries about the elevation and population of counties."-->

This ends Part Two of the tutorial.
