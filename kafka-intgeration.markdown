---
layout: post
title:  GigaSpaces XAP Kafka integration
categories: SBP
parent: processing.html
weight: 100
---
{% summary page %}This best practice explains how to use use Kafka with XAP.{% endsummary %}
 {% tip %}
 **Author**:  Oleksiy Dyagilev<br/>
 **Recently tested with GigaSpaces version**: XAP 9.6<br/>
 **Last Update:** Feb 2014<br/>

{% endtip %}

# Introduction 

Apache Kafka is a distributed publish-subscribe messaging system. It is designed to support persistent messaging with O(1) disk structures that provide constant time performance even with many TB of stored messages.
Apache Kafka provides High-throughput - even with very modest hardware, Kafka can support hundreds of thousands of messages per second. Apache Kafka support partitioning the messages over Kafka servers and distributing consumption over a cluster of consumer machines while maintaining per-partition ordering semantics. Apache Kafka used many tmes to perform data parallel load into Hadoop.

This best practive is aimed to integrate GigaSpaces with Apache Kafka. GigaSpaces write-behind the IMDG operations to Kafka making it available for the subscribors. Such could be Hadoop or other data warehousing systems using the data for reporting and processing.

# XAP Kafka Integration Architecture
 
The XAP Kafka integration implementated via the `SpaceSynchronizationEndpoint` interface deployed as a Mirror service PU. It consume a batch of IMDG operations, converts them to a custom Kafka messages and sends these to Kafka server using the Kafka Producer API.

GigaSpace-Kafka protocol is simple and represents the data and its IMDG operation. The message consists of the IMDG operation type (Write, Update , remove, etc.) and the actual data object. The Data object itself could be represented either as a single object or as a Document with key/values pairs (`SpaceDocument`). Since a kafka message should be sent over the wire, it should be serialized to bytes in some way. The default encoder utilizes Java serialization mechanism which implies Space classes (domain model) to be `Serializable`. 

By default Kafka messages are uniformly distributed across Kafka partitions. Please note, even though IMDG operations appear ordered in `SpaceSynchronizationEndpoint`, it doesn’t imply correct ordering of data processing in Kafka consumers. See below diagram:

# Getting started 
## Download the Kafka Example
The example application located in `<project_root>/example`. It demonstrates how to configure Kafka persistence and implement a simple Kafka consumer to pull data from Kafka and store in HsqlDB.

## Running the Example
In order to run an example, please follow the instruction below:
1.	Install Kafka
2.	Start Zookeeper and Kafka server
bin/zookeeper-server-start.sh config/zookeeper.properties
bin/kafka-list-topic.sh --zookeeper localhost:2181
3.	Build project
{% highlight java%}
cd <project_root>
mvn clean install
{% endhighlight %}
4.	Deploy example to GigaSpaces
{% highlight java%}
cd example
mvn os:deploy
{% endhighlight %}
5.	Run HsqlDB client and make sure data is populated into ‘data’ table.
{% highlight java%}
mvn os:hsql-ui
select * from PERSON;
{% endhighlight %}

# Configuration

## Library Dependency

The following maven dependency needs to be included to your project in order to use Kafka persistence. This artifact is built from `<project_rood>/kafka-persistence` source directory.

{% highlight xml %}
<dependency>
	<groupId>com.epam</groupId>
	<artifactId>kafka-persistence</artifactId>
	<version>1.0-SNAPSHOT</version>
</dependency>
{% endhighlight %}


## Mirror service 

Here is an example of Kafka Space Synchronization Endpoint configuration:

{% highlight xml %}
<bean id="kafkaSpaceSynchronizationEndpoint" class="com.epam.openspaces.persistency.kafka.KafkaSpaceSynchronizationEndpointFactoryBean">
	<property name="producerProperties">
		<props>
			<prop key="metadata.broker.list"> localhost:9092</prop>
			<prop key="request.required.acks">1</prop>
		</props>
	</property>
</bean>

<!--
	The mirror space. Uses the Kafka external data source. Persists changes done on the Space that
	connects to this mirror space into the Kafka.
-->
<os-core:mirror id="mirror" url="/./mirror-service" space-sync-endpoint="kafkaSpaceSynchronizationEndpoint" operation-grouping="group-by-replication-bulk">
	<os-core:source-space name="space" partitions="2" backups="1"/>
</os-core:mirror>
{% endhighlight %}

Please consult Kafka documentation for the full list of available producer properties.
The default properties applied to Kafka producer are the following:

Property	Default value	Description
key.serializer.class	com.epam.openspaces.persistency.kafka.protocol.impl.serializer. KafkaMessageKeyEncoder	Message key serializer of default Gigaspace-Kafka protocol
serializer.class	com.epam.openspaces.persistency.kafka.protocol.impl.serializer. KafkaMessageEncoder	Message serializer of default Gigaspace-Kafka protocol

These default properties could be overridden if there is a need to customize GigaSpace-Kafka protocol. See Customization section below for details.

## Space class 

In order to associate Kafka topic with domain model class, class should be annotated with @KafkaTopic annotation and marked as Serializable. Here is an example

{% highlight java%}
@KafkaTopic("user_activity")
@SpaceClass
public class UserActivity implements Serializable {
    ...
}
{% endhighlight %}

## Space Documents

To configure Kafka topic for SpaceDocuments or Extended SpaceDocument, the property KafkaPersistenceConstants.SPACE_DOCUMENT_KAFKA_TOPIC_PROPERTY_NAME should be added to document. Here is an example

public class Product extends SpaceDocument {
  
{% highlight java%}
public Product() {
	super("Product");
	super.setProperty(SPACE_DOCUMENT_KAFKA_TOPIC_PROPERTY_NAME, "product");
}
{% endhighlight %}

It’s also possible to configure name of the property which defines the Kafka topic for SpaceDocuments. Set spaceDocumentKafkaTopicName to the desired value as shown below.

{% highlight xml %}
<bean id="kafkaSpaceSynchronizationEndpoint" class="com.epam.openspaces.persistency.kafka.KafkaSpaceSynchrspaceDocumentKafkaTopicNameonizationEndpointFactoryBean">
	...
	<property name="spaceDocumentKafkaTopicName" value="topic_name" />
</bean>
{% endhighlight %}

## Kafka consumers

Kafka persistence library provides a wrapper around native Kafka Consumer API to preset configuration responsible for GigaSpace-Kafka protocol serialization. Please see com.epam.openspaces.persistency.kafka.consumer.KafkaConsumer, example of how to use it could be found in <project_root>/example module.

##Customization

- Kafka persistence was designed to be extensible and customizable. 
- If you need to create a custom protocol between GigaSpace and Kafka, provide an implementation of AbstractKafkaMessage, AbstractKafkaMessageKey, AbstractKafkaMessageFactory.
- If you would like to customize how data sync operations are sent to Kafka or how Kafka topic is chosen for given entity, provide an implement of AbstractKafkaSpaceSynchronizationEndpoint.
- If you want to create a custom serializer, look at KafkaMessageDecoder and KafkaMessageKeyDecoder.
Kafka Producer client (which is used under the hood) could be configured with a number of settings, see Kafka documentation.
