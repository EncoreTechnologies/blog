+++
author = "John Schoewe"
author_url = "https://github.com/jschoewe"
categories = ["John Schoewe", "Encore"]
date = "2017-11-15"
description = "How Encore uses Grafana and InfluxDB to Gather Information About Our PDUs and Monitor Their Power Consumption"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Using Grafana and InfluxDB to Monitor Power Consumption"
type = "post"

+++

This blog explains how we configured and use Grafana to monitor an InfluxDB database that contains PDU power supply metrics.

<br />
# Background
InfluxDB is a time series database that we use to store dozens of metrics from all of our power distribution units (PDUs). This database is updated every 15 minutes with a python script that parses snmp data from our PDUs and stores it in the correct table and row. After getting the data into the db, we used Grafana to create graphs of some of the power consumption metrics so that we could easily see how much power was being used by each PDU. All pictures and tutorials are from Grafana version 4.6.1 and InfluxDB version 1.2.4.


<br />
# Getting Started
First, you will need to install InfluxDB, Apache, and Grafana 

<b>Install InfluxDB</b>

Check https://portal.influxdata.com/downloads for the latest version
<br />
<br />
Install InfluxDB, start the service, and create a user and databse

```bash
cd
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.2.4.x86_64.rpm
sudo yum -y localinstall influxdb-1.2.4.x86_64.rpm
service influxdb start
influx
CREATE USER grafana WITH PASSWORD ‘password’
CREATE DATABASE pdu
USE pdu
GRANT READ ON pdu TO grafana
```

<b>Install Apache and Grafana</b>

```bash
yum -y install httpd
chkconfig --levels 235 httpd on
```

Check https://grafana.com/grafana/download for the latest version
```bash
cd
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-4.6.1.linux-x64.tar.gz 
sudo yum -y localinstall grafana-4.4.1-1.x86_64.rpm 
systemctl daemon-reload
systemctl start grafana-server
sudo systemctl enable grafana-server.service
```
NOTE: Grafana listens for incoming connections on port 3000
<br />
NOTE: Default username/password = admin/admin
<br />
<br />

After setting up our InfluxDB database, we wrote a script to parse SNMP data from our PDUs. This script is being run as a cron job to update the DB with new metrics every 15 minutes. Once that was finished, all we had to do was configure Grafana to use our database.
<br />
<br />
<br />

# Configure Grafana to Use InfluxDB
1. Log into the Grafana server via web browser on port 3000
2. Default username and password are both "admin"
3. Click on "Add Datasource" from the Home dashboard
<br />
![image](/img/2017/11/grafana_add_ds_button.png)
4. Fill out the required information so that it resembles the picture below
<br />
![image](/img/2017/11/grafana_add_ds_page.png)
<br />
The name of the data source doesn't matter, but the type needs to be "InfluxDB" If you installed Grafana and InfluxDB on the same server, then you can use http://localhost:8086 otherwise use the hostname from the InfluxDB server that was configured above Under "Basic Auth Details" enter the credentials to log into the InfluxDB server Under "InfluxDB Details" enter the InfluxDB user and password that was created in the "Install InfluxDB" section 
5. Click on "Add" in the bottom left corner and a success message should appear
<br />
![image](/img/2017/11/grafana_add_ds_success.png)

<br />
<br />
# Create Graphs of the Data
1. Log into the Grafana server via web browser on port 3000
2. Click on the graph button to create a new row
<br />
![image](/img/2017/11/grafana_new_dashboard.png)
3. Click on the new row and click "Edit" in the small box that pops up
<br />
![image](/img/2017/11/grafana_row_edit.png)
4. Click the drop-down next to "Data Source" and select the datasource that we added in the last section
5. Edit this query information and tell it which tables and data to graph from your database
<br />
![image](/img/2017/11/grafana_table_query.png)
6. Add one or more queries to the graph to compare your data
<br />
![image](/img/2017/11/grafana_line_graph1.png)
<br />
![image](/img/2017/11/grafana_line_graph2.png)