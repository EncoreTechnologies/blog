+++
author = "John Schoewe"
author_url = "https://github.com/jschoewe"
categories = ["John Schoewe", "Encore"]
date = "2019-01-29"
description = "How Encore uses Grafana and InfluxDB to Gather Information About Our PDUs and Visuallize Their Power Consumption"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Using Grafana and InfluxDB to Visualize Power Consumption"
type = "post"

+++

This blog explains how we configured and use Grafana to graph and visualize PDU data stored in InfluxDB.

<br />
# Background
InfluxDB is a time series database that we use to store dozens of metrics from all of our power distribution units (PDUs). This database is updated periodically with a python script that parses snmp data from our PDUs and stores it in the correct table and row. After getting the data into the db, we used Grafana to create graphs of some of the power consumption metrics so that we could easily see how much power was being used by each PDU. All pictures are from Grafana version 4.6.1.
<br />
<br />
<br />
# Storing Data in InfluxDB
The reasons why we chose InfluxDB for this project are because it is designed to store a large volume of time-series data, it uses an SQL-like query language to interact with, and Grafana has a built-in plugin for it that makes queryinig easier. After installing InfluxDB, we made separate databases for pdu, breaker, and panel info and separate tables for each measurement that we wanted to track from these objects. Each of these measurements are also stored with various tags (similar to columns in an SQL db) that identify what the value is used for.

Pictures of how we query these databases can be seen below.
<br />
<br />
<br />

# Configuring Grafana to Use InfluxDB
After setting up our InfluxDB database and providing it with some metrics from our PDUs, all we had to do was configure Grafana to use our database and write queries to visualize the data. In order to do that, we logged into the Grafana server on port 3000 navigated to Configuraion -> Data Sources and clicked on "Add data source". InfluxDB is a built-in option to choose from, so we just had to provide a server and database name and credentials to connect to it. Once the data sources were added, we wrote queries to create graphs from them.

<br />
<br />
# Create Graphs of the Data
The picture below shows how Grafana uses InfluxDB's SQL-like query language to create lines in a graph
<br />
![image](/img/2019/01/grafana_table_query.png)

1. Log into the Grafana server via web browser on port 3000
2. Click on the graph button to create a new row
<br />
![image](/img/2019/01/grafana_new_dashboard.png)
3. Click on the new row and click "Edit" in the small box that pops up
<br />
![image](/img/2019/01/grafana_row_edit.png)
4. Click the drop-down next to "Data Source" and select the datasource that we added in the last section
5. Edit this query information and tell it which tables and data to graph from your database
<br />
![image](/img/2019/01/grafana_table_query.png)
6. Add one or more queries to the graph to compare your data. Below you can see an example of the queries we use including the measurement name and different tags:
<br />
![image](/img/2019/01/grafana_line_graph3.png)