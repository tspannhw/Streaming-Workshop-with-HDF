# Streaming-Workshop-with-HDF

# Contents
- [Introduction](#introduction) - Workshop Introduction
- [Use case](#use-case) - Building a 360 view for customers
- [Lab 1](#lab-1) - Cluster installation
  - Create an HDF 3.2 cluster
  - Access your cluster
- [Lab 2](#lab-2) - Platform preparation (admin persona)
  - Create schemas in Schema Registry
  - Create record readers and writters in NiFi
  - Create process groups and variables in NiFi
  - Create events topics in Kafka
  - Create environment and service pool in SAM
- [Lab 3](#lab-3) - MySQL CDC data ingestion (DataEng persona)
  - Configure MySQL to enable binary logs
  - Ingest and format data in NiFi
  - Store events in ElasticSearch
  - Publish update events in Kafka
  - Version flow in NiFi Registry
- [Lab 4](#lab-4) - Logs data collection with MiNiFi(DataEng persona)
  - Design MiNiFi pipeline
  - Deploy MiNiFi agent
  - Deploy MiNiFi pipeline 
  - Design NiFi pipeline
- [Lab 5](#lab-5) - TODO Fraud detection with SAM (Dev/Business Analyst persona)
- [Lab 6](#lab-6) - TODO Realtime analytics with Druid (Dev/Business Analyst persona)

  ---------------
# Introduction

The objective of this workshop is to build an end to end streaming use case with HDF. This include edge collection, flow management and stream processing. A focus is also put on governance and best practices using tools such Schema Registry, Flow Registry and Variable Registry. At the end of the workshop, you will understand why HDF is a complete streaming platform that offers entreprise features to build, test and deploy any advanced streaming application. In addition, you will learn details on some of the new features brought by the latest HDF versions:
  - Use NiFi to ingest CDC data in real time
  - Use Record processors to benefit from improved performance and integration with schema registry
  - Route and filter data using SQL
  - Deploy and use MiNiFi agents
  - Version flow developments and propagation from dev to prod
  - Integration between NiFi and Kafka to benefit from latest Kafka improvements (transactions, message headers, etc)
  - Test mode in Stream Analytics Manager to mock a streaming application before deploying it

# Use case

In this workshop, we will build a simplified streaming use case for a retail company. We will ingest data from MySQL Database and web apps logs to build a 360 view of a customer in real-time. This data can be stored on modern databases such as HDP or ElasticSearch to offer more scalability and agility compared to legacy DBs. Based on these two data streams, we will implement a fraud detection algorithm based on business rules. For instance, if a user updates their account (for instance their address) and buys an item that's more expensive than their usual expenses, we may decide to investigate. This can be a sign that their account has been hacked and used to buy an expensive item that will be shipped to a new address. The following picture explains the high level architecture of the use case.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/use_cases.png)

# Lab 1 

## Create an HDF 3.2 cluster

For the coming labs, we will install a one-node HDF cluster with NiFi, NiFi Registry, Kafka, Storm, Schema Registry and Stream Analytics Manager. We will use the field cloud for this workshop but the instructions will work for any cloud provider (AWS for instance).

  - Connect to your OpenStack account on field cloud and create a VM with at least 16 GB of RAM (this corresponds to a m3.xlarge instance). Keep the default parameters and num_vms to 1. Note the stack name that you defined as it will be used to access to your cluster. Let's assume that your stack name is hdfcluster.
  - SSH to your cluster using the field PEM key ``` ssh -i field.pem centos@hdfcluster0.field.hortonworks.com ```
  - Launch the cluster install using the following instruction
  ```
  curl -sSL https://raw.githubusercontent.com/ahadjidj/Streaming-Workshop-with-HDF/master/scripts/install_hdf3-2_cluster.sh | sudo -E sh
  ```
This scripts installs a MySQL Database, ElasticSearch, MiNiFi, Ambari agent and server, HDF MPack and HDF services required for this workshop. Cluster installation will take about 10 minutes.

## Access your Cluster

  - When the script finishes the work, login to Ambari Web UI by opening http://{YOUR_IP}:8080 and log in with **admin/StrongPassword**
  - Open the different UIs and check that all services are running and are healthy (NiFi, NiFi Registry, SR, SAM, etc)
  - Connect to the MySQL DB using bash or tools like MySQLWorkbench. A workshop DB has been created for the lab. You have also two users:
    - **root/StrongPassword** usable from localhost only
    - **workshop/StrongPassword** usable from remote and has full privileges on the workshop DB 
    
# Lab 2 Platform preparation (admin persona)
To enforce best practices and a minimal governance, there are a few tasks that an admin should do before granting access to the platform. These tasks include:
  - Define users, roles and privileges on each tool (SAM, NiFi, Etc)
  - Define the schemas of events that we will use. This avoids having developpers using their own schemas which makes applications integration and evolution a real nightmare.
  - Define and enforce naming convention that makes it easier to manage applications lifecycle (eg. NiFi PG and processors names)
  - Define global variables that should be used to make application migration between environments simple
  - etc

In this lab, we will implement some of these best practices to set the right environment for our developpers.

## Create schemas in Schema Registry

In this workshop, we will manipulate three type of events.

### Customer events

These events are data coming from the MySQL DB through the CDC layer. Each event has different fields describing the customer (id, first name, last name, etc). To declare this schema, go to Schema Registry and add a new schema with these details:
  - Name: customers
  - Descrption: schema for CDC events
  - Type: Avro Schema Provider
  - Schema Group: Kafka
  - Compatibility: both
  - Evolve: true
 
For the schema text, use the following Avro description, also available [here](https://raw.githubusercontent.com/ahadjidj/Streaming-Workshop-with-HDF/master/schemas/customers.asvc)
 
  ```
{
  "type": "record",
  "name": "customers",
  "fields" : [
    {"name": "id", "type": "int"},
    {"name": "first_name", "type": ["null", "string"]},
    {"name": "last_name", "type": ["null", "string"]},
    {"name": "gender", "type": ["null", "string"]},
    {"name": "phone", "type": ["null", "string"]},   
    {"name": "email", "type": ["null", "string"]},    
    {"name": "countrycode", "type": ["null", "string"]},
    {"name": "country", "type": ["null", "string"]},
    {"name": "city", "type": ["null", "string"]},
    {"name": "state", "type": ["null", "string"]},
    {"name": "address", "type": ["null", "string"]},
    {"name": "zipcode", "type": ["null", "string"]},
    {"name": "ssn", "type": ["null", "string"]},
    {"name": "timezone", "type": ["null", "string"]},
    {"name": "currency", "type": ["null", "string"]},
    {"name": "averagebasket", "type": ["null", "int"]}
  ]
} 
  ```
### Logs events
These events are data coming from Web Application through the MiNiFi agents deployed on application servers. Each event, describe a customer browsing behavior on a webpage. The provided information is the customer id, the product page being viewed, session duration and if the customer bought the product at the end of the session or not. Go to Schema Registry and add a new schema with these details:
  - Name: logs
  - Descrption: schema for logs events
  - Type: Avro Schema Provider
  - Schema Group: Kafka
  - Compatibility: both
  - Evolve: true
 
 For the schema text, use the following Avro description, also available [here](https://raw.githubusercontent.com/ahadjidj/Streaming-Workshop-with-HDF/master/schemas/logs.asvc)
 
  ```
{
  "type": "record",
  "name": "logs",
  "fields" : [
    {"name": "id", "type": "int"},
    {"name": "product", "type": ["null", "string"]},
    {"name": "sessionduration", "type": ["null", "int"]},
    {"name": "buy", "type": ["null", "boolean"]},
    {"name": "price", "type": ["null", "int"]}
  ]
}
  ```
We also need another logs event (logs_view) that contain only the product browsing session information with the buy and price fields. We will see why later in the labs.

  ```
{
  "type": "record",
  "name": "logs_view",
  "fields" : [
    {"name": "id", "type": "int"},
    {"name": "product", "type": ["null", "string"]},
    {"name": "sessionduration", "type": ["null", "int"]}
  ]
}
  ```
### Alerts events
At the end of the workshop, we will use the different events to detect eventual frauds. If a fraud is detected, we will send an alert to inform an application or a supervisor. To achieve this, we need a new schema for these events. Go to Schema Registry and add a new schema with these details:
  - Name: alerts
  - Descrption: schema for alerts events
  - Type: Avro Schema Provider
  - Schema Group: Kafka
  - Compatibility: both
  - Evolve: true
 
For the schema text, use the following Avro description, also available [here](https://raw.githubusercontent.com/ahadjidj/Streaming-Workshop-with-HDF/master/schemas/alerts.asvc)
 
  ```
{
  "type": "record",
  "name": "alerts",
  "fields" : [
    {"name": "_id", "type": "int"},
    {"name": "price", "type": ["null", "int"]},
    {"name": "first_name", "type": ["null", "string"]},
    {"name": "last_name", "type": ["null", "string"]},
    {"name": "averagebasket", "type": ["null", "int"]}
  ]
}
  ```
## Create record readers and writters in NiFi

To use these schema in NiFi, we will leverage record based processors. These processors use record readers and writers to offer improved performances and to use schemas defined globally in a Schema Registry. Our sources (MySQL CDC event and Web App logs) generate data in JSON format so we will need a JSON reader to deserialise data. We will store this data in ElasticSearch and publish it to Kafka. Hence, we need JSON and Avro writers to serialize the data. To add a reader/writer accessible by all our NiFi flows, navigate to the root canvas, click on Configure, Controller service and click on "+" button.

### Add a HortonworksSchemaRegistry
Before adding any record reader/writer, we need to add a Hortonworks Schema Registry to tell NiFi where to look for schemas definitions. Add a HortonworksSchemaRegistry controller and configure it with your SR URL as shown below:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/HortonworksSchemaRegistry.png)

### Add JsonTreeReader
To deserialize JSON data, add a JsonTreeReader and configure it as shown below. Note that the **Schema Access Strategy** is set to **Use 'Schema Name' Property**. This means that flow files going through this serializer must have an attribute **schema.name** that specifies the name of the schema that should be used.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/JsonTreeReader.png)

### Add JsonRecordSetWriter
To serialize JSON data for which we have a defined schema, add a JsonRecordSetWriter and configure it as shown below.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/JsonRecordSetWriter.png)

### Add AvroRecordSetWriter
Event collected by NiFi will be published to Kafka for further consumption. In the second use case, we will use SAM to analyse data in realtime and detect potential frauds. SAM expects the events to be in Avro format with the first byte containing an encoded schema reference. To prepare data for SAM consumption, we need to add AvroRecordSetWriter and set **Schema Write Strategy** to **HWX Content-Encoded Schema Reference** as shown below:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/AvroRecordSetWriter.png)

## Create process groups and variables in NiFi
It's critical to organize your flows when you have a shared NiFi instance. NiFi flows can be organized per data sources where each Process Group defines the processing that should be applied to data coming from this source. If you have several flow developers working on different projects, you can assign roles and privileges to each one of them on those process groups. The PG organisation is also useful to declare variables for each source or project and make flow migration from one environment to another one easier. In this workshop, we will define 3 PGs as shown below. Note the naming convention (sourceID_description) that will be useful for flows migration and monitoring.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/PGS.png)

  - SRC1_CDCIngestion: ingest data from MySQL. This PG will use the below variables. For instance, we can change the variable elastic.url from localhost to the production Elastic cluster URL in a central location instead of updating it in every Elastic processor.
  
  ```
mysql.driver.location : /usr/share/java/mysql-connector-java.jar
mysql.username : root
mysql.serverid : 123
mysql.host : 127.0.0.1:3306
source.schema : customers
elastic.url : http://localhost:9200
kafka.url : hdfcluster0.field.hortonworks.com:6667
  ```  
![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/PG1.png)

  - SRC2_LogsIngestion: ingest data from Web applications. This PG will use the following variables:
  
  ```
source.schema : logs
elastic.url : http://localhost:9200
kafka.url : hdfcluster0.field.hortonworks.com:6667
  ```  
  - Agent1_LogsIngestion: the template that will be deployed in each MiNiFi agent for log ingestion. This PG don't use any variable.
 
 ## Create events topics in Kafka

 As an admin, we need to provision Kafka topics and define their access policies. Use the following instructions to create the topics that we will use. In the future, topic provisioning will be possible through SMM.

  ```
/usr/hdf/current/kafka-broker/bin/kafka-topics.sh --zookeeper hdfcluster0.field.hortonworks.com:2181 --create --topic customers --partitions 1 --replication-factor 1
/usr/hdf/current/kafka-broker/bin/kafka-topics.sh --zookeeper hdfcluster0.field.hortonworks.com:2181 --create --topic logs --partitions 1 --replication-factor 1
/usr/hdf/current/kafka-broker/bin/kafka-topics.sh --zookeeper hdfcluster0.field.hortonworks.com:2181 --create --topic alerts --partitions 1 --replication-factor 1

  ```  
 ## Create service pool and application environment in SAM  
Finally, we need to provision a service pool and an environment in SAM for our application. For the service pool, use the HDF cluster URL : http://hdfcluster0.field.hortonworks.com:8080/api/v1/clusters/hdfcluster

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/ServicePool.png)

# Lab 3
In this lab, we will use NiFi to ingest CDC data from MySQL. The MySQL DB has a table "customers" that stores information on our customers. We would like to receive each change in the table as an event (insert, update, etc) and use it with other sources to build a customer 360 view in ElasticSearch. The high level flow can be described as follows:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/UC1.png)

  - Ingest events from MySQL (SRC1_CDCMySQL)
  - Keep only Insert and Delete events and format them in a usable JSON format (SRC1_RouteSQLVerbe to SRC1_SetSchemaName)
  - Insert and update customer data in ElasticSearch where we will build the 360 view (SRC1_MergeRecord to SRC1PutElasticRecord)
  - Publish update events in Kafka to use them for fraud detection use cases with SAM (SRC1_PublishKafkaUpdate)

## Configure MySQL to enable binary logs
NiFi has a native CDC feature for MySQL databases. To use it, the MySQL DB must be configured to use binary logs. Use the following instructions to enable binary logs for the workshop DB and use ROW format CDC events.

  ```
sudo bash -c 'sudo cat <<EOF >> /etc/my.cnf
server_id = 1
log_bin = delta
binlog_format=row
binlog_do_db = workshop
EOF'

sudo systemctl restart mysqld.service
  ``` 

## Ingest and format data in NiFi
Go to SRC1_DCIngestion PG, add a CaptureChangeMySQL processor and configure it as follows:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/CDC.png)

Note that we are leveraging the variables we defined previously. The CaptureChangeMySQL needs a MapCache service to store its state (binlog position and transaction ID). Add a MapCache client and server. 

The CDC processor can be configured to listen to some events only. In our use case, we won't use Begin/Commit/DDL statements. But for teaching purposes, we will receive those events and filter them later. Add a RouteOnAttribute processor and configure it as follows:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/Route1.png)

At this level, you can generate some data to see how CDC events look like. Use the following instructions to insert 10 customers to the MySQL DB:

  ```
curl "https://raw.githubusercontent.com/ahadjidj/Streaming-Workshop-with-HDF/master/scripts/create-customers-table.sql" > "create-customers-table.sql"

mysql -h localhost -u workshop -p"StrongPassword" --database=workshop < create-customers-table.sql
  ``` 
Use the different relations to see how the data looks like for each event. Make sure that you have 10 flow files in "insert,update" relation, 20 in "unmatched" and 1 in "ddl".

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/cdcresults.png)

To get update events, you can connect to MySQL and update some customer information. You should get 11 flow files in the "insert,update" relation now.

  ```
mysql -h localhost -u root -pStrongPassword
UPDATE customers SET phone='0645341234' WHERE id=1;
  ```
For the next step, add an EvaluateJsonPath processor to extract the table name. Connect the Route processor to the EvaluateJsonProcessor with insert and update relations only. 

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/ExtractTableName.png)

As you can see, each event has a lot of additional information that are not useful for us. To keep only of interest, add a JoltTransformationProcessor with the following Jolt specification:

  ```
[
  {
    "operation": "shift",
    "spec": {
      "columns": {
        "*": {
          "@(value)": "[#1].@(1,name)"
        }
      }
    }
  }
]
  ``` 
Now that we have our data in the target Json format, let's add an attribute schema.name with the value ${source.schema} to prepare to use record-based processors and our customers schema defined in SR.

## Store events in ElasticSearch
Before storing data in ES, let's separate between Insert and Update events first. This is not required since the PutElasticSearchRecord processor supports both insert and update operations. But for other processors, this may be required. Also, some CDC tools generate different schemas for insert and update operations so routing data is required. 

Add a RouteOnAttribute processor like before and separate between inserts and updates. For each type of event, add a MergeRecord and PutElasticSearchHttpRecord configured as follows. Use the Index operation for the Insert events and Update operation for Update events.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/Merge.png)
![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/PutES.png)

Note how easy it is to use Record-based processors now that we have prepared our Schema and Reader/Writer.

Open ElasticSearch UI and check that your customer data has been indexed: http://hdfcluster0.field.hortonworks.com:9200/customers/_search?pretty

## Publish update events in Kafka
The last step for this lab is to publish "insert" events in Kafka. These events will be used by SAM to check if there's a risk of fraud. Hence, we need to use Avro record writer in the Kafka processor configuration.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/KafkaPublish1.png)

## Version flow in NiFi Registry
Now that we have our first use case implemented, let's explore the NiFi Flow Registry service. Add a "workshop" bucket in NiFi Registry. In NiFi, right click on the PG SRC1_CDCIngestion, versions and "Start version control". Give a name to your flow and click on save. Check that you flow is saved in the registry.

Unfortunately, we don't have another cluster to deploy our flow so let's deploy it in the same instance. Add a new PG to NiFi, click on import, and select your flow. Explore registry features: for instance, change variable in the second instance, make changes in the first instance, and save the new flow version. This should not impact your local variables.

# Lab 4
The objective of this lab is to ingest web applications logs with MiNiFi. Each web application generates logs on customer behaviour on the website. An event is a JSON line that describes a user behaviour on a product web page and gives information on:
  - Id: the user browsing the website. id = 0 means that the user is not connected or not known.
  - Product: the product id that the customer has looked at.
  - Sessionduration: how long the customer stayed on the product web page. A short duration means that the user is not interested by the product and is only browsing.
  - Buy: a boolean that indicates if the user bought the product or not
  - Price: the total amount of money that the customer spent

We will simulate the web apps by writing directly events to the files inside the tmp folder. The final objective will be to add browsing information to customer data in Elasticsearch. This will be the first step for the customer 360 view. 
 
## Design MiNiFi pipeline
Before working on the MiNiFi pipeline, we need to prepare an Input port to receive data from the agent. In the NiFi root Canvas, add an Input port and call it **SRC2_InputFromWebApps**. 

Now, inside the NiFi Agent1_logsIngestion process group, create the MiNiFi flow as follows:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/minifi.png)

As you can see, it's a very simple pipeline that tails all web-appXXX.log files inside /tmp and send them to our NiFi via S2S. You can enrich this pipeline with more steps such as compression or filtering on session duration later if you like. The tail fail configuration is described below:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/tail.png)

Save the flow as a template and download the associated XML file.

## Deploy MiNiFi agent
MiNiFi is part of the NiFi ecosystem but should be deployed separately. Currently, the deployment should be automated by the user. In the near future, we will build a Command & Control tool (C2) that can be used to deploy, monitor and manage a number of MiNiFi agents from a central location. Run the following instructions to install MiNiFi in /usr/hdf/current/minifi

  ``` 
sudo mkdir /usr/hdf/current/minifi
sudo mkdir /usr/hdf/current/minifi/toolkit
wget http://apache.claz.org/nifi/minifi/0.5.0/minifi-0.5.0-bin.tar.gz
tar -xvf minifi-0.5.0-bin.tar.gz
sudo cp -R minifi-0.5.0/. /usr/hdf/current/minifi
  ``` 

In addition to NiFi, we will need the MiNiFi toolkit to convert our XML template file into YAML file understandable by MiNiFi.

  ``` 
wget http://apache.claz.org/nifi/minifi/0.5.0/minifi-toolkit-0.5.0-bin.tar.gz
tar -xvf minifi-toolkit-0.5.0-bin.tar.gz
sudo cp -R minifi-toolkit-0.5.0/. /usr/hdf/current/minifi/toolkit
  ``` 

## Deploy MiNiFi pipeline 
SCP the template you downloaded from your NiFi node to your HDF cluster. You can Curl mine and change it to add your NiFi URL in the S2S section.

  ``` 
sudo curl -sSL -o /usr/hdf/current/minifi/conf/minifi.xml https://raw.githubusercontent.com/ahadjidj/Streaming-Workshop-with-HDF/master/scripts/minifi.xml
  ``` 
Use the toolkit to convert your XML file to YAML format:

  ``` 
sudo /usr/hdf/current/minifi/toolkit/bin/config.sh transform /usr/hdf/current/minifi/conf/minifi.xml /usr/hdf/current/minifi/conf/config.yml
  ``` 

Now start the MiNiFi agent and look to the logs:

  ``` 
sudo /usr/hdf/current/minifi/bin/minifi.sh start
tail -f /usr/hdf/current/minifi/logs/minifi-app.log
  ``` 
## Design NiFi pipeline
Inside the SRC2_LogIngestion PG, create the NiFi pipeline that will process data coming from our agent. The general flow is:
  - Receive data through S2S
  - Filter events based on the sessionduration. We consider that a customer who spends less than 20 seconds on a product page is not interested. We will filter these events and ignore them.
  - Filter unknown users browsing events (id=0). These events can be browsing activity from a non-logged-in customer or a customer who is not yet logged in. We can store these events in HDFS for other use cases such as product recommendations. In a real life scenario, a browsing session will have an ID and can be used to link browsing history to a user once logged in.
  - For the other events, we will do two things:
    - Update customer data in Elasticsearch to include the products that the customer has looked at. For the sake of simplicity, we will store only the last item. If you want to keep a complete list, you can use ElasticSearch API with scripts feature (eg. "source": "ctx.source.products.add(params.product)")
    - Convert logs event to Avro and publish them to Kafka. These events will be used by SAM in the fraud detection use case. The final flow looks like the below:
    
![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/UC3.png)

Start by adding an Input port followed by an update attribute that adds an attribute schema.name with the value ${source.schema}.

To route events based on the different business rules, we will use an interesting Record processor that leverage Calcite to do SQL on flow files. Add a query record processor and configure it as shown below:

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/query.png)

As you can see, this processor will create two relations (unknown and validsessions) and route data according to the SQL query. Note that a subset of fields can be selected also (ex. SELECT id, sessionduration from FLOWFILE).

Route data coming from the unknown relation to HDFS.

Route data coming from validsessions to Kafka. In the PublishKafkaRecord, use the AvroRecordSetWriter as Record Writer to publish data in Avro format. Remember that we set **Schema Write Strategy** of the Avro Writer to **HWX Content-Encoded Schema Reference**. This means that each Kafka message will have the schema reference encoded in the first byte of the message (required by SAM).

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/avro.png)

Now let's update our Elasticsearch index to add data on customer browsing behaviors. To learn how to do schema conversion with record based processors, let's consider that we want to add the browsed product ID and sessionduration as opposed to the information on whether the customer bought the product or not. To implement this, we need a ConvertRecord processor.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/convert.png)

As you can see, I had to create a new JsonSetWritter to specify the write Schema which is different from the read schema referenced by the attribute **schema.name**. The **Views JsonRecordSetWriter** should be configured as below. Note that the Schema Name field is set to logs_view that we have already defined in our schema registry. We can avoid fixing the schema directly in the record writer by creating global Read/Write controller and use two attributes : schema.name.input and schema.name.output.

![Image](https://github.com/ahadjidj/Streaming-Workshop-with-HDF/raw/master/images/views.png)

Add a PutElasticSearchHTTPRecord and configure it to update your customer data.

Now let's test the end-to-end flow by creating a file in /tmp with some logs events.

  ``` 
cat <<EOF >> /tmp/web-app.log
{"id":2,"product":"12321","sessionduration":60,"buy":"false"}
{"id":0,"product":"24234","sessionduration":120,"buy":"false"}
{"id":10,"product":"233","sessionduration":5,"buy":"true","price":2000}
{"id":1,"product":"98547","sessionduration":30,"buy":"true","price":1000}
EOF
  ``` 
You should see data coming from the MiNiFi agent to NiFi through S2S. Data will be filtered, routed and stored in Kafka and Elasticsearch. Check data in ElasticSearch and confirm that browsing information has been added to customer 1 and 2. Check also that you have one event that go through the unknown connection.
