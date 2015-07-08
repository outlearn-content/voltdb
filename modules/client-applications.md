<!--
{
"name" : "client-applications",
"version" : "0.1",
"title" : "Client Applications",
"description": "Tutorial for VoltDB.",
"freshnessDate" : 2015-07-08,
"homepage" : "http://docs.voltdb.com/tutorial/",
"license" : "All Rights Reserved"
}
-->

<!-- @section -->

## Overview

We now have a working sample database with data. We even wrote a stored procedure demonstrating how to update the data. To run the stored procedure we used the pre-existing csvloader utility. However, most applications require more logic than a single stored procedure. Understanding how to integrate calls to the database into your client applications is key to producing a complete business solution, In this lesson, we explain how to interact with VoltDB from client applications.

VoltDB provides client libraries in a number of different programming languages, each with their own unique syntax, supported datatypes, and capabilities. However, the general process for calling VoltDB from client applications is the same no matter what programming language you use:

1. Create a client connection to the database.
2. Make one of more calls to stored procedures and interpret their results.
3. Close the connection when you are done.

This lesson will show you how to perform these steps in several different languages.

<!-- @section -->

## Making the Sample Application Interactive

As interesting as the population and location information is, it isn't terribly dynamic. Population does not change that quickly and locations even less so. Creating an interactive application around this data alone is difficult. However, if we add just one more layer of data things get interesting.

The United States National Weather Service (part of the Department of Commerce) issues notices describing dangerous weather conditions. These alerts are available online in XML format and include the state and county FIPS numbers of the areas affected by each weather advisory. This means it is possible to load weather advisories correlated to the same locations for which we have population and elevation data. Not only is it possible to list the weather alerts for a given state and county, we could also determine which events have the highest impact, in terms of population affected.

<!-- @section -->

## Designing the Solution

To make use of this new data, we can build a solution composed of two separate applications:

* One to load the weather advisory data
* Another to fetch the alerts for a specific location

This matches the natural break down of activities, since loading the data can be repeated periodically — every five or ten minutes say — to ensure the database has the latest information. Whereas fetching the alerts would normally be triggered by a user request.

At any given time, there are only a few hundred weather alerts and the alerts are updated only every 5-10 minutes on the NWS web site. Because it is a small data set updated infrequently, the alerts would normally be a good candidate for a replicated table. However, in this case, there can be — and usually are — multiple state/county pairs associated with each alert. Also, performance of user requests to look up alerts for a specific state and county could be critically important depending on the volume and use of that function within the business solution.

So we can normalize the data into two separate tables: nws_alert for storing general information about the alerts and local_event which correlates each alert (identified by a unique ID) to the state and county it applies to. This second table can be partitioned on the same column, state_num, as the towns and people tables. The new tables and associated indexes look like this:

```
CREATE TABLE nws_event (
   id VARCHAR(256) NOT NULL,
   type VARCHAR(128),
   severity VARCHAR(128),
   SUMMARY VARCHAR(1024),
   starttime TIMESTAMP,
   endtime TIMESTAMP,
   updated TIMESTAMP,
   PRIMARY KEY (id)
);

CREATE TABLE local_event (
    state_num TINYINT NOT NULL,
    county_num SMALLINT NOT NULL,
    id VARCHAR(256) NOT NULL
);

CREATE INDEX local_event_idx ON local_event (state_num, county_num);
CREATE INDEX nws_event_idx ON nws_event (id);

PARTITION TABLE local_event ON COLUMN state_num;
```

It is possible to add the new table declarations to the existing schema file. However, it is possible to load multiple separate schema files into the database using sqlcmd, as long as the DDL statements don't overlap or conflict. So to help organize our source files, we can create the new table declarations in a separate schema file, weather.sql. We will also need some new stored procedures, so we won't load the new DDL statements right now. But you can view the weather.sql file in your tutorial directory to see the new table declarations.

<!-- @section -->

## Designing the Stored Procedures for Data Access

Having defined the schema, we can now define the stored procedures that the client applications need. The first application, which loads the weather alerts, needs two stored procedures:

* FindAlert — to determine if a given alert already exists in the database
* LoadAlert — to insert the information into both the nws_alert and local_alert table

The first stored procedure is a simple SQL query based on the id column and can be defined in the schema. The second procedure needs to create a record in the replicated table nws_alert and then as many records in local_alert as needed. Additionally, the input file lists the state and county FIPS numbers as a string of six digit values separated by spaces rather than as separate fields. As a result, the second procedure must be written in Java so it can queue multiple queries and decipher the input values before using them as query arguments. You can find the code for this stored procedure in the file LoadAlert.java in the tutorial directory.

These procedures are not partitioned because they access the replicated table nws_alert and — in the case of the second procedure — must insert records into the partitioned table local_alert using multiple different partitioning column values.

Finally, we also need a stored procedure to retrieve the alerts associated with a specific state and county. In this case, we can partition the procedure based on the state_num field. This last procedure is called GetAlertsByLocation.

The following procedure declarations complete the weather.sql schema file:

```
CREATE PROCEDURE FindAlert AS
   SELECT id, updated FROM nws_event
   WHERE id = ?;

CREATE PROCEDURE FROM CLASS LoadAlert;

CREATE PROCEDURE GetAlertsByLocation
   PARTITION PROCEDURE ON TABLE local_event COLUMN state_num
   AS SELECT w.id, w.summary, w.type, w.severity,
             w.starttime, w.endtime
             FROM nws_event as w, local_event as l
             WHERE l.id=w.id and
                   l.state_num=? and l.county_num = ? and
                   w.endtime > TO_TIMESTAMP(MILLISECOND,?)
             ORDER BY w.endtime;
```

Now the stored procedures are written and the additional schema file created, we can compile the Java stored procedure, package both it and the UpdatePeople procedure from Part Five into a Jar file, and load both the procedures and schema into the database. Note that you must load the stored procedures first, so the database can find the class file when it processes the CREATE PROCEDURE FROM CLASS statement in the schema:

```
$ javac -cp "$CLASSPATH:/opt/voltdb/voltdb/*"  LoadAlert.java
$ jar cvf storedprocs.jar *.class
$ sqlcmd
1> load classes storedprocs.jar;
2> file weather.sql;
```

<!-- @section -->

## Creating the LoadWeather Client Application

The goal of the first client application, LoadWeather, is to read the weathers alerts from the National Weather Service and load them into the database. The basic program logic is:

1. Read and parse the NWS alerts feed.
2. For each alert, first check if it already exists in the database using the FindAlert procedure.
        * If yes, move on.
        * If no, insert the alert using the LoadAlert procedure.

Since this application will be run periodically, we should write it in a programming language that allows for easy parsing of XML and can be run from the command line. Python meets these requirements so we will use it for the example application.

The first task for the client application is to include all the libraries we need. In this case we need the VoltDB client library and standard Python libraries for input/output and parsing XML. The start of our Python program looks like this:

```
import sys
from xml.dom.minidom import parseString
from voltdbclient import *
```

The beginning of the program also contains code to read and parse the XML from standard input and define some useful functions. You can find this in the program LoadWeather.py in the tutorial directory.

More importantly, we must, as mentioned before, create a client connection. In Python this is done by creating an instance of the FastSerializer:

```
client = FastSerializer("localhost", 21212)
```

In Python, we must also declare any stored procedures we intend to use. In this case, we must declare FindAlert and LoadAlert:

```
finder = VoltProcedure( client, "FindAlert", [
FastSerializer.VOLTTYPE_STRING,
] )

loader = VoltProcedure( client, "LoadAlert", [
FastSerializer.VOLTTYPE_STRING,
FastSerializer.VOLTTYPE_STRING,
FastSerializer.VOLTTYPE_STRING,
FastSerializer.VOLTTYPE_STRING,
FastSerializer.VOLTTYPE_STRING,
FastSerializer.VOLTTYPE_STRING,
FastSerializer.VOLTTYPE_STRING,
FastSerializer.VOLTTYPE_STRING
] )
```

The bulk of the work of the application is a set of loops that walk through each alert in the XML structure checking if it already exists in the database and, if not, adding it. Again, the code for parsing the XML can be found in the tutorial directory if you are interested. But the code for calling the VoltDB stored procedures is the following:

```
# Check to see if the alert is already in the database.
response = finder.call([ id ])
if (response.tables):
    if (response.tables[0].tuples):
          # Existing alert
          cOld += 1
    else:
          # New alert
          response = loader.call([ id, wtype, severity, summary,
                                   starttime, endtime, updated, fips])
          if response.status == 1:
                cLoaded += 1
```

Note how the application uses the response from the procedures in two different ways:

* The response from FindAlert (finder) is used to check if any records were returned. If so, the alert already exists in the database.
* The response from LoadAlert (loader) is used to verify the status of the call. If the return status is one, or success, then we know the alert was successfully added to the database.

There is additional information in the procedure response besides just a status code and the data returned by the queries. But LoadWeather shows two of the most commonly used components.

The last step, once all the alerts are processed, is to close the connection to the database:

```
client.close()
```

<!-- @section -->

## Running the LoadWeather Application

Because Python is a scripting language, you do not need to compile your code before running it. However, you do need to tell Python where to find any custom libraries, such as the VoltDB client. Simply add the location of the VoltDB client library to the environment variable PYTHONPATH. For example, if VoltDB is installed in your home directory as the folder ~/voltdb, the command to use is:

```
$ export PYTHONPATH="$HOME/voltdb/lib/python/"
```

Once you define PYTHONPATH, you are ready to run LoadWeather. Of course, you will also need weather alerts data to load. A sample file of weather data is included in the tutorial files data directory:

```
$ python LoadWeather.py < data/alerts.xml
```

Or you can pipe the most recent alerts directly from the NWS web site:

```
$ curl http://alerts.weather.gov/cap/us.php?x=0 | python LoadWeather.py
```

<!-- @section -->

## Creating the GetWeather Application

Now the database contains weather data, we can write the second half of the solution — an application to retrieve all of the alerts associated with a specific location. In a real world example, the GetWeather application is relatively simple, consisting of a single call to the GetAlertsByLocation stored procedure for the user's current location. Run manually, one query at a time, this is not much of a test of VoltDB — or any database. But in practice, where there can be hundreds or thousands of users running the application simultaneously, VoltDB's performance profile excels.

To demonstrate both aspects of our hypothetical solution, we can write two versions of the GetWeather application:

* A user interface showing what it looks like to the user and how easy it is to integrate VoltDB into such applications.
* A high-performance application to emulate real world loads on the database.

<!-- @section -->

## VoltDB in User Applications

The first example adds VoltDB access to a user application. In this case, a web interface implemented in HTML and Javascript. You can find the complete application in the /MyWeather folder in the tutorial directory. Run the application by opening the file GetWeather.html in a web browser. If you use the sample alert data included in the tutorial directory, you can look up alerts for Androscoggin County in Maine to see what the warnings look like to the user.

Most of the application revolves around the user interface, including HTML. CSS, and Javascript code to display the initial form and format the results. Only a very small part of the code is related to VoltDB access.

In fact, for applications like this VoltDB has simplified programming interfaces that do not require the explicit setup and tear down of a normal database application. In this case, we can use the JSON interface, which does not require you to open and close an explicit connection. Instead, you simply call the database with your query and it returns the results in standard JavaScript Object Notation (JSON). VoltDB takes care of managing the connection, pooling queries, and so on.

So the actual database call only takes up two statements in the file GetAlerts.js;

* One to construct the URL that will be invoked, identifying the database server, the stored procedure (GetAlertsByLocation), and the parameters.
* Another to do the actual invocation and specify a callback routine.

```
  var url = "http://localhost:8080/api/1.0/" +
            "?Procedure=GetAlertsByLocation&Parameters=" +
            "[" + statenum + "," + countynum + "," + currenttime + "]";
  callJSON(url,"loadAlertsCallback");
```

Once the stored procedure completes, the callback routine is invoked. The callback routine uses the procedure response, this time in JSON format, much the same way the LoadWeather application does. First it checks the status to make sure the procedure succeeded and then it parses the results and formats them for display to the user.

```
function loadAlertsCallback(data) {

    if (data.status == 1) {
      var output = "";
      var results = data.results[0].data;
      if (results.length > 0) {
          var datarow = null;
          for (var i = 0; i < results.length; i++) {
              datarow = results[i];
              var link = datarow[0];
              var descr = datarow[1];
              var type = datarow[2];
              var severity = datarow[3];
              var starttime = datarow[4]/1000;
              var endtime = datarow[5]/1000;
              output += '<p><a href="' + link + '">' + type + '</a> '
                         + severity + '<br/>' + descr + '</p>';
          }
       }  else {
          output = "<p>No current weather alerts for this location.</p>";
       }
       var panel = document.getElementById('alerts');
       panel.innerHTML = "<h3>Current Alerts</h3>" + output;


    } else {
       alert("Failure: " + data.statusstring);
    }
}
```

<!-- @section -->

## VoltDB in High Performance Applications

The second example of GetWeather emulates many users accessing the database at the same time. It is very similar to the voter sample application that comes with the VoltDB software.

In this case we can write the application in Java. As we did before with LoadWeather, we need to import the VoltDB libraries and open a connection to the database. The code to do that in Java looks like this:

```
import org.voltdb.*;
import org.voltdb.client.*;

   [ . . . ]

        /*
         * Instantiate a client and connect to the database.
         */
        org.voltdb.client.Client client;
        client = ClientFactory.createClient();
        client.createConnection("localhost");
```

The program then does one ad hoc query. You can add ad hoc queries to your applications by calling the @AdHoc system procedure with the SQL statement you want to execute as the only argument to the call. Normally, it is best for performance to always use stored procedures since they are precompiled and can be partitioned. However, in this case, where the query is run only once at the beginning of the program to get a list of valid county numbers per state, there is little or no negative impact.

You use system procedures such as @AdHoc just as you would your own stored procedures, identifying the procedure and any arguments in the callProcedure method. Again, we use the status in the procedure response to verify that the procedure completed successfully.

```
ClientResponse response = client.callProcedure("@AdHoc",
    "Select state_num, max(county_num) from people " +
    "group by state_num order by state_num;");
if (response.getStatus() != ClientResponse.SUCCESS){
    System.err.println(response.getStatusString());
    System.exit(-1);
}
```

The bulk of the application is a program loop that randomly assigns a state and county number and looks up weather alerts for that location using the GetAlertsByLocation stored procedure. The major difference here is that rather than calling the procedure synchronously and waiting for the results, we call the procedure asynchronously and move immediately on to the next call.

```
while ( currenttime - starttime < timelimit) {

      // Pick a state and county
    int s = 1 + (int)(Math.random() * maxstate);
    int c = 1 + (int)(Math.random() * states[s]);

      // Get the alerts
    client.callProcedure(new AlertCallback(),
                         "GetAlertsByLocation",
                         s, c, new Date().getTime());

    currenttime = System.currentTimeMillis();
     if (currenttime > lastreport + reportdelta) {
        DisplayInfo(currenttime-lastreport);
        lastreport = currenttime;
     }
}
```

Asynchronous procedure calls are very useful for high velocity applications because they ensure that the database always has queries to process. If you call stored procedures synchronously, one at a time, the database can only process queries as quickly as your application can send them. Between each stored procedure call, while your application is processing the results and setting up the next procedure call, the database is idle. Essentially, all of the parallelism and partitioning of the database are wasted while your application does other work.

By calling stored procedures asynchronously, the database can queue the queries and process multiple single-partitioned in parallel, while your application sets up the next procedure invocation. In other words, both your application and the database can run at top speed. This is also a good way to emulate multiple synchronous clients accessing the database simultaneously.

Once an asynchronous procedure call completes, the application is notified by invoking the callback procedure identified in the first argument to the callProcedure method. In this case, the AlertCallback procedure. The callback procedure then processes the procedure response, which it receives as an argument, just as your application would after a synchronous call.

```
static class AlertCallback implements ProcedureCallback {
    @Override
    public void clientCallback(ClientResponse response) throws Exception {
        if (response.getStatus() == ClientResponse.SUCCESS) {
            VoltTable tuples = response.getResults()[0];
              // Could do something with the results.
              // For now we throw them away since we are
              // demonstrating load on the database
            tuples.resetRowPosition();
            while (tuples.advanceRow()) {
               String id = tuples.getString(0);
               String summary = tuples.getString(1);
               String type = tuples.getString(2);
               String severity = tuples.getString(3);
               long starttime = tuples.getTimestampAsLong(4);
               long endtime = tuples.getTimestampAsLong(5);
            }
         }
        txns++;
        if ( (txns % 50000) == 0) System.out.print(".");
    }
}
```

Finally, once the application has run for the predefined time period (by default, five minutes) it prints out one final report and closes the connection.

```
    // one final report
if (txns > 0 && currenttime > lastreport)
   DisplayInfo(currenttime - lastreport);

client.close();
```

<!-- @section -->

## Running the GetWeather Application

How you compile and run your client applications depends on the programming language they are written in. For Java programs, like the sample GetWeather application, you need to include the VoltDB JAR file in the class path. If you installed VoltDB as ~/voltdb, a subdirectory of your home directory, you can add the VoltDB and associated JAR files and your current working directory to the Java classpath like so:

```
$ export CLASSPATH="$CLASSPATH:$HOME/voltdb/voltdb/*:$HOME/voltdb/lib/*:./"
```

You can then compile and run the application using standard Java commands:

```
$ javac GetWeather.java
$ java GetWeather
Emulating read queries of weather alerts by location...
............................
1403551 Transactions in 30 seconds (46783 TPS)
...............................
1550652 Transactions in 30 seconds (51674 TPS)
```

As the application runs, it periodically shows metrics on the number of transactions processed. These values will vary based on the type of server, the configuration of the VoltDB cluster (sites per host, notes in the cluster, etc) and other environmental factors. But the numbers give you a rough feel for the performance of your database under load.

<!-- @section -->

## In Conclusion

How your database schema is structured, how tables and procedures are partitioned, and how your client application is designed, will all impact performance. When writing client applications, although the specifics vary for each programming language, the basics are:

* Create a connection to the database.
* Call stored procedures and interpret the results. Use asynchronous calls where possible to maximize throughput.
* Close the connection when done.

Much more information about how to design your client application and how to tune it and the database for maximum performance can be found online in the Using VoltDB manual and VoltDB Performance Guide.
