---
layout: post
title:  XAP Integration with Kafka
categories: SBP
parent: data-access-patterns.html
weight: 50
---

{% summary page %}This shows explains how to export data from a space and import it.{% endsummary %}
 {% tip %}
 **Author**:<br/>
 **Recently tested with GigaSpaces version**: XAP 9.7<br/>
 **Last Update:** Apr 2014<br/>

{% endtip %}

# Introduction

This tool demonstrates how to export data from a space via serializing it to a file. You can then use this to import the data back into a another space. GigaSpace-Kafka protocol is simple and represents the data and its IMDG operation. The message consists of the IMDG operation type (Write, Update , remove, etc.) and the actual data object. The Data object itself could be represented either as a single object or as a Space Document with key/values pairs (`SpaceDocument`).
Since a Kafka message is sent over the wire, it should be serialized to bytes in some way.
The default encoder utilizes Java serialization mechanism which implies Space classes (domain model) to be `Serializable`.

By default Kafka messages are uniformly distributed across Kafka partitions. Please note, even though IMDG operations appear ordered in `SpaceSynchronizationEndpoint`, it doesn't imply correct ordering of data processing in Kafka consumers. See below diagram:

{% indent %}
![xap-export-import.png](../pics/xap-export-import.png)
{% endindent %}

# Getting started

## Download the Export/Import Example

You can download the example code from [here](/download_files/sbp/kafka-integration.tar).
Unzip into an empty folder.

The example located under `<project_root>/example`. It demonstrates how to configure Kafka persistence and implements a simple Kafka consumer pulling data from Kafka and store in HsqlDB.

## Running the Example
In order to run an example, please follow the instruction below: