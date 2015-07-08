# Preface

VoltDB is designed to help you achieve world-class performance in terms of throughput and scalability. At the same time, at its core, VoltDB is a relational database and can do all the things a traditional relational database can do. So before we go into the fine points of tuning a database application for maximum performance, let's just review the basics.

The following tutorial familiarizes you with the features and capabilities of VoltDB step by step, starting with its roots in SQL and walking through the individual features that help VoltDB excel in both flexibility and performance, including:

* Schema Definition
* SQL Queries
* Partitioning
* Schema Updates
* Stored Procedures

For the purposes of this tutorial, let's assume we want to learn more about the places where we live. How many cities and towns are there? Where are they? How many people live there? What other interesting facts can we find out?

Our study will take advantage of data that is freely available from government web sites. But for now, let's start with the basic structure.

## How to Use This Tutorial

Of course, you can just read the tutorial to get a feel for how VoltDB works. But we encourage you to follow along using your own copy of VoltDB if you wish.

The data files used for the tutorial are freely available from public web sites; links are provided in the text. However, the initial data is quite large. So we have created a subset of source files and pre-processed data files that is available from the VoltDB web site at the following address for those who wish to try it themselves:

http://downloads.voltdb.com/technologies/other/tutorial_files_50.zip

For each section of the tutorial, there is a subfolder containing the necessary source files, plus one subfolder, data, containing the data files. To follow along with the tutorial, do the following:

1. Create a folder to work in.
2. Unpack the tutorial files into the folder.
3. At the beginning of each section of the tutorial:
    1. Set default to your tutorial folder.
    2. Copy the sources for that current tutorial into the your working directory, like so: `$ cp -r tutorial1/* ./`

The tutorial also uses the VoltDB command line commands. Be sure to set up your environment so the commands are available to you, as described in the [installation chapter](http://docs.voltdb.com/UsingVoltDB/ChapGetStarted.php) of [Using VoltDB](http://docs.voltdb.com/UsingVoltDB/).
