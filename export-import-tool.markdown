---
layout: post
title:  Exporting data from a Space to a file
categories: SBP
parent: data-access-patterns.html
weight: 50
---

{% summary page %}This shows explains how to export data from a space and import it.{% endsummary %}
{% tip %}
 **Author**:Shay Hasidim, John Burke<br/>
 **Recently tested with GigaSpaces version**: XAP 9.7<br/>
 **Last Update:** Apr 2014<br/>
{% endtip %}

# Introduction

With this tool we will demonstrate how to export data from a space via serializing it to a file. We can then re-import the data back into the space. The tool executes distributed tasks in 'preprocess' mode, which reads the serialization files and returns a de-duplicated list of the classes only.

{% indent %}
![xap-export-import.png](pics/xap-export-import.png)
{% endindent %}

# Getting started

### Download the Export/Import Example

You can download the example project from [here](download_files/Export_Tool.zip) and unzip into an empty folder.


### Running the Example
#####Step 1: Deploy a space and write some data.
<br/>If you do not have an already deployed space with data, you will need to deploy a new space and write some dummy data to it.

#####Step 2: Build the project<br/>
{% highlight java %}
cd <project_root>
mvn clean install
{% endhighlight %}
 
#####Step 3:	Run the tool to export the objects<br/>
{% highlight java %}
cd target
java -classpath < classpath > com.gigaspaces.support.export.SpaceClassExporterTool -e
{% endhighlight %}



In this case we are using a simple Account POJO.

{% indent %}
Note: Make sure that classes of the POJOs are set in the classpath before running.
{% endindent %}

{% highlight java %}
2014-04-06 20:34:14,029  INFO [com.gigaspaces.common] - (tid-23) : found 1 classes
2014-04-06 20:34:14,043  INFO [com.gigaspaces.common] - (tid-23) : starting export thread for xap.poc.common.Account
2014-04-06 20:34:14,043  INFO [com.gigaspaces.common] - (tid-23) : (tid-386) : reading space class : xap.poc.common.Account
2014-04-06 20:34:14,043  INFO [com.gigaspaces.common] - (tid-23) : (tid-386) : space partition contains 1239 objects
2014-04-06 20:34:14,043  INFO [com.gigaspaces.common] - (tid-23) : (tid-386) : writing to file : /Users/kemi/Documents/GigaSpaces/XAP/gigaspaces-xap-premium-9.7.0-ga/bin/xap.poc.common.Account.1.ser.gz
2014-04-06 20:34:14,044  INFO [com.gigaspaces.common] - (tid-23) : (tid-386) : read 1239 objects from space partition
2014-04-06 20:34:14,044  INFO [com.gigaspaces.common] - (tid-23) : (tid-386) : export operation took 215 millis
2014-04-06 20:34:14,044  INFO [com.gigaspaces.common] - (tid-23) : finished writing 1 classes
{% endhighlight %}

#####Step 4:	Run the tool to import the objects back into a space<br/>
{% highlight java %}
cd target
java -classpath < classpath > com.gigaspaces.support.export.SpaceClassExporterTool -e
{% endhighlight %}


{% indent %}
Note: A space read call for each class is executed before trying to perform any import.
{% endindent %}

##Advanced Options
The tool supports the following arguments:

| Argument      | Name          	| Description |
| ------------- |:-------------	| :-----|
| -e             | Export 			| Performs space class export | 
| -i             | Import    	  	| Performs space class export |
| -g             | Group		    | The names of lookup groups - comma separated |
| -l             | Locators		    | The names of lookup services hosts - comma separated |
| -s             | Space		    | The name of the space |
| -c             | Classes		    | The classes whose objects to import/export - comma separated|
| -b             | Batch		    | The batch size - default is 1000|
| -p             | Partitions	    | The partition(s) to restore - comma separated|
| -t             | Test			    | Performs a sanity check|