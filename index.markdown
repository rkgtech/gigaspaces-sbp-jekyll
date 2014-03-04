---
layout: post
title:  Solutions, Patterns & Best Practices
categories: SBP
---

This section contains GigaSpaces XAP known patterns, solutions and best practicees that we've accumulated over time based on numerous real life use cases. Each pattern is listed with its author/origin, and the GigaSpaces product version that it was tested against. 

{% note %}The patterns below are presented as is. While most of them are used in real life use cases and productions environments, and GigaSpaces is making its best effort to keep them up to date, they should not be considered as fully productized artifacts, and you should test them in your own environment before using them.{% endnote %}

# Data Access Patterns

{: .table .table-bordered}
| Pattern | Level | Description |
|:--------------|:------|:------------|
|[Kafka Integration](./kafka-integration.html) | Advanced | This pattern explains how to integrate XAP with Kafka.|
|[Cache Interface](./cache-interface.html) | Beginner | This pattern explains how to implement a cache interface using the Map API.|
|[Moving from Hibernate to Space](./moving-from-hibernate-to-space.html) | Beginner | Moving from Database Centric into In-Memory Model. Can be used when moving from **J2EE Session Bean** into XAP.|
|[Finding Partition Load](./finding-partition-load.html)| Beginner | Monitoring Data Grid partitions in real time.|
|[Even Data Distribution](./even-data-distribution.html)| Beginner | Technique to ensure even data load-balancing with partitioned IMDG.|
|[Space Object Modeling](./space-object-modeling.html)| Beginner | Migrating Relational Model into Object model.|
|[Oracle Delta Server](./oracle-delta-server.html)| Beginner |This pattern presents the Oracle Delta Server allowing the data grid to receive updates from the database conducted by applications that are not using the data grid as their system of record (Non-Aware Data-Grid Clients) |
|[Excel that Scales Solution](./excel-that-scales-solution.html)| Beginner | Leveraging GigaSpaces .Net API to scale Excel applications.|
|[Global ID Generator](./global-id-generator.html)| Beginner | Replacing Database sequencing with lightweight In-Memory ID Generator.|
|[Web Service PU](./web-service-pu.html)|Beginner | Web Service Processing Unit. Using the CXF Framework.|
|[ODBC Driver](./odbc-driver.html)| Beginner | Accessing GigaSpaces Data Grid via 3rd party ODBC Driver.|
|[IMDG with Large Backend Database Support](./imdg-with-large-backend-database-support.html)| Advanced | Leveraging IMDG running in LRU mode with large backend Database.|
|[DB2 Delta Server](./db2-delta-server.html)| Advanced |This pattern presents the DB2 Delta Server allowing the data grid to receive updates from the database conducted by applications that are not using the data grid as their system of record (Non-Aware Data-Grid Clients) |
|[Dynamic Polling Container Templates using TriggerOperationHandler](./dynamic-polling-container-templates-using-triggeroperationhandler.html)| Advanced | Polling Containers using TriggerOperationHandler |
|[Custom Matching](./custom-matching.html)| Advanced | Implementing complex customized queries for pattern matching.|
|[Custom Eviction](./custom-eviction.html)| Advanced | Implementing used defined data eviction when using large backend Database.|
|[Lowering the Space Object Footprint](./lowering-the-space-object-footprint.html)| Advanced | Implementing compaction API to reduce the object footprint when storing objects with large numeric values.|
|[Rapid Data Load](./rapid-data-load.html)| Advanced | Loading large amount of data into each partition using Map-Reduce technique.|
|[Schema Evolution](./schema-evolution.html)| Advanced | Leveraging different techniques to enhance the application data model without IMDG downtime.|
|[Distributed Bitmap](./distributed-bitmap.html) | Advanced| Use XAP to distribute a bitmap across a partitioned cluster. |
|[Modeling your data](./modeling-your-data.html) | Beginner| Moving from a centralized to a distributed data store. |



# Parallel Processing & Messaging Patterns

{: .table .table-bordered}
| Pattern | Level | Description |
|:--------------|:------|:------------|
|[Map-Reduce Pattern - Executors Example](./map-reduce-pattern---executors-example.html)| Beginner | Implementing Task Executors and Service Executors to perform parallel queries or parallel processing across multiple IMDG partitions. Can be used when moving from **J2EE EJB**/RMI/IIOP into XAP.|
|[Master-Worker Pattern](./master-worker-pattern.html)| Beginner | Grid computing pattern. Implementing distributed processing across multiple workers deployed into the Grid. |
|[Event Driven Remoting Example](./event-driven-remoting-example.html)| Beginner | Request-Response pattern. Implementing asynchronous remove service invocation. Can be used when moving from **J2EE MDB** into XAP.|
|[Mule ESB Example](./mule-esb-example.html)| Advanced | Scaling ESB based application. Using the IMDG as the ESB state repository. Deploying the ESB into the GigaSpaces grid allowing it to scale in an elastic manner.|
|[Priority Based Queue](./priority-based-queue.html)| Advanced | Messaging based pattern. Can be used when moving from **J2EE JMS** Quality of Service into XAP.|
|[Parallel Queue Pattern](./parallel-queue-pattern.html)| Advanced |Messaging based pattern. Can be used when moving from **J2EE JMS Service Activator Aggregator Strategy/MDB** into XAP.|
|[Unit Of Work](./unit-of-work.html)| Advanced |Messaging based pattern. Can be used when moving from **J2EE JMS** Unit of Order into XAP.|
|[Drools Rule Engine Integration](./drools-rule-engine-integration.html)| Advanced | Scaling business rule management system based application.|
|[Spring Batch PU](./spring-batch-pu.html)| Advanced | Complex Batch Processing using [Spring Batch](http://static.springsource.org/spring-batch).|
|[JTA-XA Example](./jta-xa-example.html) | Advanced | Integrating GigaSpaces with an external JMS Server using XA. Atomikos is used as the JTA Transaction provider. Apache ActiveMQ is used as a the JMS provider.|

# Setup Production Environment

{: .table .table-bordered}
| Pattern | Level | Description |
|:--------------|:------|:------------|
|[Embedding Tomcat as a Web Container](./tomcat-pu.html)| Beginner | This article presents embedding Tomcat 7 within the XAP data grid|
|[Embedding XAP for OEMs](./embedding-xap-for-oems.html)| Beginner | A quick and simple example of how an OEM might embed GigaSpaces XAP for customer use.|
|[Universal Deployer](./universal-deployer.html) |Beginner | Allows deploying composite applications. Support multiple processing unit dependency based deployment.|
|[Data-Collocation Deployment Topology](./data-collocation-deployment-topology.html)| Beginner | Considerations which topology to choose when deploying both business logic and data into the Grid.|
|[Mirror Monitor](./mirror-monitor.html)| Beginner | Monitoring asynchronous persistency behavior in real time using JMX.|
|[Safe Grid Shutdown](./safe-grid-shutdown.html)| Beginner | Cleanly shutting down the grid after all async replication operations are committed.|
|[Hyperic integration](./hyperic-integration.html)| Beginner | Hyperic monitoring integration example.|
|[Primary-Backup Zone Controller](./primary-backup-zone-controller.html)|Beginner | Using Zones to control Data-Grid primary/backup instances location.|
|[RESTful Admin API](http://www.openspaces.org/display/RES/Project+Documentation)| Beginner | Using web services to monitor and administrate GigaSpaces.|
|[JMX Space Statistics](./jmx-space-statistics.html)| Beginner | Using JMX to expose space statistics.|
|[Space Dump and Reload](./space-dump-and-reload.html)| Beginner | Using the administration API to dump the space data into a file and reload it back.|
|[Scaling Agent](./scaling-agent.html)| Beginner | Using the administration API to scale web applications. Can be used when moving **J2EE Web applications** into XAP elastic Web Container.|
|[Web Load Balancer Agent PU](./web-load-balancer-agent-pu.html)| Advanced | Using the administration API to build customized HTTP load-balancer agent. Can be used when moving **J2EE Web applications** into XAP elastic Web Container.|
|[Moving into Production Checklist](./moving-into-production-checklist.html)| Advanced | All what you need to review before moving your system into production.|
|[Capacity Planning](./capacity-planning.html)| Advanced | Considerations for sizing your system before moving into production.|
|[Refreshable Business Logic Example](./refreshable-business-logic-example.html)| Advanced | Using the administration API to reload new application code (hot deploy) while the application is running.|
|[Recipes For XAP Cloud Deployment](./automated-xap-deployment-with-cloudify.html)| Advanced | Recipes For XAP Cloud Deployment.|
|[Puppet to install and configure XAP](./puppet-xap-module.html)| Advanced | This pattern explains how to use Puppet to install and configure XAP.|





# WAN Based Deployment

{: .table .table-bordered}
| Pattern | Level | Description |
|:--------------|:------|:------------|
|[Clusters Over WAN](./wan-replication-gateway.html)| Beginner | Example for WAN replication implementation.|
|[WAN Gateway Pass Through Replication](./wan-gateway-pass-through-replication.html)| Beginner | Example for implementing a pass through WAN replication topology.|
|[WAN Gateway Master Slave Replication](./wan-gateway-master-slave-replication.html)| Beginner | Example for implementing a single-master/multi-slave replication topology.|
|[Observable WAN](./observable-wan.html)|Advanced|Monitor and measure the replication performance of a multi-site deployment.|


# Solutions

{: .table .table-bordered}
| Pattern | Level | Description |
|:--------------|:------|:------------|
|[Elastic Distributed Calculation Engine](./elastic-distributed-calculation-engine.html)|Advanced| Elastic Distributed Calculation Engine implementation using Map-Reduce approach.|
|[Trading Settlement](./trading-settlement.html) |Advanced| A trading settlement system where the entire tier-based architecture is built on GigaSpaces.|
|[Mainframe Integration](./mainframe-integration.html) |Advanced| GigaSpaces XAP can simplify the migration effort from mainframe based systems and reduce the cost of the legacy applications. GigaSpaces XAP act as a front-end layer for mainframe based systems may boost the system performance and improve the overall system response time on peak load.|
|[Global Http Session Sharing](./global-http-session-sharing.html)|Advanced| Global HTTP Session Sharing allows transparent session replication between remote sites and sharing between different application servers in real-time.|
