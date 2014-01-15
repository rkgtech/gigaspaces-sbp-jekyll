---
layout: post
title:  Moving into Production Checklist
categories: SBP
parent: production.html
weight: 800
---


{% summary %}Moving into Production Checklist{% endsummary %}

**Author**: Shay Hassidim, Deputy CTO, GigaSpaces<br/>
Date: December  2009<br/>
Latest Date: Feb 2011<br/>

The following list should provide you with the main activities to be done prior moving your system into production. Reviewing this list and executing the relevant recommendations should result in a stable environment with a low probability of unexpected behavior or failures that are result of a GigaSpaces environment misconfiguration.

# Sharing Grid Management Services Infrastructure
There are numerous ways allowing different systems/applications/groups to share the same pool of servers (in development or production environment) on the network.  A non-exhaustive list of some of the options is delineated below:

1. Outside of GigaSpaces:  Dedicated hardware for each group, each set of servers runs an independent GigaSpaces runtime environment (aka Service Grid) without sharing the same server between different groups.  This naïve approach is good for simple or temporary scenarios. In this case each GigaSpaces runtime environment is isolated from each other using [different LOOKUPLOCATORS](#Running Multiple Locators) or [different LOOKUPGROUPS](#Running Multiple Groups) value.

2. [Using Multiple Zones](#Running Multiple Zones):  A single GigaSpaces runtime environment spans all servers, where each group of GigaSpaces containers (across several machines) are labeled with a specific Zone.  You may have multiple Zones used by different containers on the same server. For example, have on server A two containers labeled with zoneX and four containers labeled with zoneY and on server B two containers labeled with zoneX and four containers labeled with zoneY.
At deployment time, application services (aka processing Unit) are deployed using a specific Zone. This instructs the system to provision the services into the corresponding containers.  Use of multiple Zones breaks logically the runtime environment into different segments.

3. [Using Multiple Lookup Groups (multicast lookup discovery)](#Running Multiple Groups):  All servers running multiple GigaSpaces runtime environments, where each GigaSpaces container using a specific Lookup Group when registering with the Lookup Service.  At deployment time, application services (aka processing Unit) are deployed using a specific lookup group. Use of multiple lookup group breaks logically the Infrastructure into different segments. The Lookup Group value controlled via the `LOOKUPGROUPS` environment variable. When using this option you should make sure multicast is enabled on all machines.

4. [Using Multiple Lookup Locators (unicast lookup discovery)](#Running Multiple Locators): All servers running multiple GigaSpaces runtime environments, where each GigaSpaces container using a specific Lookup locator when registering with the Lookup Service.  At deployment time, application services (aka processing Unit) are deployed using a specific lookup locator. Use of multiple lookup locators breaks logically the Infrastructure into different segments. If you have multiple lookup services running on the same server, each will use a different listening port. You may control this port using the `com.sun.jini.reggie.initialUnicastDiscoveryPort` system property. The Lookup Locators value controlled via the `LOOKUPLOCATORS` environment variable.

5. Using a shared GigaSpaces runtime environment: A single GigaSpaces runtime environment spans all servers, with no use of Zones or Lookup Groups/Locators.   Application services share the servers and allocation done in a random manner without using any pre-defined segmentation.

For any of the above options, GigaSpaces exposes the ability to control a deployed application service in run-time, such that new application service instances can be created or existing instances can be relocated.   This tight operational control enables even more creative resource sharing possibilities.

Devising the appropriate resource sharing strategy for your system should consider the breadth of operational requirements and application services' characteristics. For example, two applications with variable load may run into trouble running on a fixed-size shared environment if peak loads coincide.
GigaSpaces provides consultancy services for the environement planning stage that addresses the above as well as other considerations impacting your environment. For more information see [GigaPro Services](http://www.gigaspaces.com/content/gigapro-full-services-offering-xap-customers)

# Binding the Process into a Machine IP Address
In many cases, the machines that are running GigaSpaces (i.e., a GSA, GSM, or GSC), or running GigaSpaces client applications (e.g., web servers or standalone JVM/.Net/CPP processes) have multiple network cards with multiple IP addresses. To make sure that the GigaSpaces processes or the GigaSpaces client application processes bind themselves to the correct IP addresses - accessible from another machines - you should use the `NIC_ADDR` environment variable, or the [java.rmi.server.hostname](http://docs.oracle.com/javase/7/docs/api/java/rmi/server/package-summary.html) system property. Both should be set to the IP of the machine (one of them in case of a machine with multiple IP addresses). Without having this environment/property specified, in some cases, a client process is not able to be notified of events generated by the GigaSpaces runtime environment or the space.

Examples:

{% highlight java %}
export NIC_ADDR=10.10.10.100
./gs-agent.sh &
{% endhighlight %}

{% highlight java %}
 java -Djava.rmi.server.hostname=10.10.10.100 MyApplication
{% endhighlight %}

{% tip %}
With the above approach, you can leverage multiple network cards within the same machine to provide a higher level of hardware resiliency, and utilize the network bandwidth in an optimal manner, by binding different JVM processes running on the same physical machine to different IP addresses. One example of this would be four GSCs running on the same machine, where two of the them are using IP_1 and the other two are using IP_2.
{% endtip %}

{% exclamation %} For more information, see [How to Configure an Environment With Multiple Network-Cards (Multi-NIC)]({%latestjavaurl%}/how-to-configure-an-environment-with-multiple-network-cards-(multi-nic).html)

# Ports
GigaSpaces uses TCP/IP for most of its remote operations. The following components within GigaSpaces require open ports:

{: .table .table-bordered}
| Service | Description | Configuration Property | Default value |Comment|
|:--------|:------------|:-----------------------|:--------------|:------|
|[Lookup Service listening port]({%latestjavaurl%}/the-lookup-service.html) |Used as part of the lookup discovery protocol.|`com.sun.jini.reggie.initialUnicastDiscoveryPort` System property|XAP 6: **4162**{% wbr %}XAP 7: **4164**{% wbr %}XAP 8: **4166**| |
|[LRMI listening port]({%latestjavaurl%}/communication-protocol.html)|Used with client-space and space-space communication. |`com.gs.transport_protocol.lrmi.bind-port` System property. |variable, random| |
|RMI registry listening port |Used as an alternative directory service.| `com.gigaspaces.system.registryPort` System property|10098 and above.| |
|Webster listening port|Internal web service used as part of the application deployment process. |`com.gigaspaces.start.httpPort` System property|9813| |
|[Web UI Agent]({%latestjavaurl%}/web-management-console.html)|GigaSpaces Dashboard Web Application. | `com.gs.webui.port` System property|8099| |

Here are examples of how to set different LRMI listening ports for the GS-UI, and another set of ports for the GSA/GSC/GSM/Lookup Service:

{% highlight java %}
 export EXT_JAVA_OPTIONS=-Dcom.gs.transport_protocol.lrmi.bind-port=7000-7500
{% endhighlight %}

{% highlight java %}
 export EXT_JAVA_OPTIONS=-Dcom.gs.transport_protocol.lrmi.bind-port=8000-8100
{% endhighlight %}

A running GSC tries to use the first free port that is not used out of the port range specified. The same port might be used for several connections (via a multiplexed protocol). If all of the port range is exhausted, an error is displayed.

{% tip %}
When there are several GSCs running on the same machine, or several servers running on the same machine, it is recommended that you set a different LRMI port range for each JVM.  Having 100 as a port range for the GSCs supports a large number of clients (a few thousand).
{% endtip %}

# Client LRMI Connection Pool and Server LRMI Connection Thread Pool

The [GigaSpaces LRMI]({%latestjavaurl%}/communication-protocol.html) uses two independent resource pools working collaboratively allowing a client to communicate with a server in a scalable manner. The client connection pool is configured via the `com.gs.transport_protocol.lrmi.max-conn-pool` and a server connection thread pool is configured via the `com.gs.transport_protocol.lrmi.max-threads`, both should be configured on the server side as system properties. You may configure these two pools' sizes and their resource timeouts to provide maximum throughput and low latency when a client communicates with a server. The default LRMI behavior will open a different connection at the client side and start a connection thread at the server side, once a multithreaded client accesses a server component. All client connections may be shared between all the client threads when communicating with the server. All server side connection threads may be shared between all client connections.

{% indent %}
[<img src="/attachment_files/sbp/lrmi_archi2.jpg" width="120" height="80">](/attachment_files/sbp/lrmi_archi2.jpg)
{% endindent %}

## Client LRMI Connection Pool
The client LRMI connection pool is maintained per server component - i.e. by each space partition. For each space partition a client maintains a dedicated connection pool shared between all client threads accessing a specific partition. When having multiple partitions (N) hosted within the same GSC, a client may open maximum of `N * com.gs.transport_protocol.lrmi.max-conn-pool` connections against the GSC JVM process.

{% tip %}
You may need to change the `com.gs.transport_protocol.lrmi.max-conn-pool` value (1024) to have a smaller number. The default value might be high for application with multiple partitions.

{% highlight java %}
Client total # of open connections = com.gs.transport_protocol.lrmi.max-conn-pool * # of partitions
{% endhighlight %}

This may result very large amount of connections started at the client side resulting "Too many open files" error. You should increase the OS' max file descriptors amount by calling the following before running the client application (on UNIX):

{% highlight java %}
ulimit -n 65536
{% endhighlight %}

or by lowering the `com.gs.transport_protocol.lrmi.max-conn-pool` value.
{% endtip %}

## Server LRMI Connection Thread Pool
The LRMI connection thread pool is a server side component. It is in charge of executing the incoming LRMI invocations. It is a single thread pool within the JVM that executes all the invocations, from all the clients and all the replication targets.

{% tip %}
In some cases you might need to increase the LRMI Connection thread pool maximum size. Without this tuning activity, the system might hang in case there would be large amount of concurrent access. See the [LRMI Configuration]({%latestjavaurl%}/communication-protocol.html#LRMI+Configuration) for details about the GigaSpaces Communication Protocol options. Using a value as **1024** for the LRMI Connection Thread Pool should be sufficient for most large scale systems.
{% endtip %}

# Lookup Locators and Groups
A space (or any other service, such as a GSC or GSM) publishes (or registers/exports) itself within the [Lookup Service]({%latestjavaurl%}/the-lookup-service.html). The lookup service acts as the system directory service. The lookup service (aka service proxy) keeps information about each service, such as its location and its exposed remote methods. Every client or service needs to discover a lookup service as part of its bootstrap process.

There are 2 main options for how to discover a lookup service:

- **Via locator(s)** - Unicast Discovery mode. With this option a specific IP (or hostname) used indicating the machine running the lookup service. This option can be used when multicast communication is disabled on the network, or when you want to avoid the overhead involved with the multicast discovery.
- **Via group(s)** - Multicast Discovery mode. relevant **only when the network supports multicast**. This is a "tag" you assign to the lookup.  Clients that want to register with this lookup service, or search for a service proxy, need to use this specific group when discovering the lookup service.

To configure the GigaSpaces runtime components (GSA,GSC,GSM,LUS) to use unicast discovery you should set the `LOOKUPLOCATORS` variable:

{% highlight java %}
export LOOKUPLOCATORS=MachineA,MachineB
./gs-agent.sh &
{% endhighlight %}

To configure the GigaSpaces runtime components (GSA,GSC,GSM,LUS) to use multicast discovery you should set the `LOOKUPGROUPS` variable:

{% highlight java %}
export LOOKUPGROUPS=Group1,Group2
./gs-agent.sh &
{% endhighlight %}

When running multiple systems on the same network infrastructure, you should isolate these by having a dedicated set of lookup services (and  GSC/GSM) for each system. Each should have different locators/groups settings.

{% comment %}

{% tip %}
By default every GigaSpaces component (Client, Lookup Service, GSC, GSM) look for a lookup service using the multicast discovery protocol. This has some overhead. If you would like to avoid this overhead (both on the client side and Lookup Service, GSC, GSM), you should disable the multicast discovery using the following system property:
`com.gs.multicast.enabled=false`

In such a case, make sure clients have their lookup locators set correctly to have the lookup service machine names or IP listed.
{% endtip %}

{% endcomment %}

## Space URL Examples
See below for examples of [Space URL]({%latestjavaurl%}/space-url.html)s you should be familiar with:

- "jini://localhost/*/space" - this space URL means that the client is trying to discover the lookup service on the localhost, together with discovering it on the network via multicast (enabled by default).

- "jini://localhost/*/space?locators=host,host2" - this space URL means that together with searching for the lookup service on the localhost or the network, we are looking for it on host1 and host2. We call this unicast lookup discovery.

- "jini://localhost/*/space?groups=A,B" - this space URL means that together with searching for the lookup service on the localhost, we are using multicast discovery to search for all the lookup services associated with group A or B.

- "jini://**/**/space" - this space URL means that searching for the lookup service is done only via multicast discovery.

- "/./space?groups=A,B" - this space URL means that the started space registers itself with group A and B. To access such a space via a remote client, it needs to use the following space URL:

{% highlight java %}
"jini://*/*/space?groups=A"
or
"jini://*/*/space?groups=B"
{% endhighlight %}

## Space Configuration with Unit Tests
When running unit tests, you might want these set up so that no remote client can access the space they are running. This includes regular clients or the GS-UI.

{% tip %}
When running a space running in embedded mode and not deployed into a GSC (standalone space), it starts a lookup service automatically. This allows you to access it from the GS-UI.
If it is running within the GSC, it finds the lookup via the `LOOKUPLOCATORS` or `LOOKUPGROUPS` settings.
{% endtip %}
.
Here is a simple confguration you should place within your pu.xml to disable the lookup service startup, disable the space registration with the lookup service, and disable the space registration with the Rmi registry, when the space starts as a PU or running as a standalone:

{% highlight java %}
    <os-core:space id="space" url="/./myspace" >
        <os-core:properties>
            <props>
                <prop key="com.j_spaces.core.container.directory_services.jini_lus.start-embedded-lus">false</prop>
                <prop key="com.j_spaces.core.container.directory_services.jini_lus.enabled">false</prop>
                <prop key="com.j_spaces.core.container.directory_services.jndi.enabled">false</prop>
                <prop key="com.j_spaces.core.container.embedded-services.httpd.enabled">false</prop>
            </props>
        </os-core:properties>
     </os-core:space>
{% endhighlight %}

# The Runtime Environment - GSA, LUS, GSM and GSCs
In a dynamic environment where you want to start [GSCs]({%latestjavaurl%}/service-grid.html#gsc) and [GSM]({%latestjavaurl%}/service-grid.html#gsm) remotely, manually or dynamically, the [GSA]({%latestjavaurl%}/service-grid.html#gsa) is the only component you should have running on the machine that is hosting the [GigaSpaces runtime environment]({%latestjavaurl%}/the-runtime-environment.html). This lightweight service acts as an agent and starts a GSC/GSM/LUS when needed.

You should plan the initial number of GSCs and GSMs based on the application memory footprint, and the amount of processing you might need. The most basic deployment should include 2 GSMs (running on different machines), 2 Lookup services (running on different machines), and 2 GSCs (running on each machine). These host your Data-Grid or any other application components (services, web servers, Mirror) that you deploy.

In general, the total amount of GSCs you are running across the machines that host the system depends on:

- The amount of data you want to store in memory.
- The JVM maximum heap size.
- The processing requirements.
- The number of users the system needs to serve.
- The total number of CPU cores the machine is running.

{% tip %}
A good number for the amount of GSCs a machine should host would be **half of the amount** of total CPU cores, each having no more than a 10G maximum heap size.
{% endtip %}

## Configuring the Runtime Environment

![newin71-star.jpg](/attachment_files/sbp/newin71-star.jpg)
JVM parameters (system properties, heap settings etc.) that are shared between all components are best set using the `EXT_JAVA_OPTIONS` environment variable. However, starting from 7.1.1, specific GSA JVM parameters can be easily passed using `GSA_JAVA_OPTIONS` that will be appended to `EXT_JAVA_OPTIONS`. If `GSA_JAVA_OPTIONS` is not defined, the system will behave as in 7.1.0. As a good practice, one can add all components' environment variables ( `GSA_JAVA_OPTIONS`, `GSM_JAVA_OPTIONS`, `GSC_JAVA_OPTIONS`, `LUS_JAVA_OPTIONS`) within the GSA script, or in a wrapper script and the values will be passed to corresponding components.

{% inittab %}

{% tabcontent Linux %}

{% highlight java %}
#Wrapper Script
export GSA_JAVA_OPTIONS='-Xmx256m'
export GSC_JAVA_OPTIONS='-Xmx2048m'
export GSM_JAVA_OPTIONS='-Xmx1024m'
export LUS_JAVA_OPTIONS='-Xmx1024m'

#call gs-agent.sh
. ./gs-agent.sh
{% endhighlight %}

{% endtabcontent %}

{% tabcontent Windows %}

{% highlight java %}
@rem Wrapper Script
@set GSA_JAVA_OPTIONS=-Xmx256m
@set GSC_JAVA_OPTIONS=-Xmx2048m
@set GSM_JAVA_OPTIONS=-Xmx1024m
@set LUS_JAVA_OPTIONS=-Xmx1024m

@rem call gs-agent.bat
call gs-agent.bat
{% endhighlight %}

{% endtabcontent %}

{% endinittab %}

## Running Multiple Groups
You may have a set of LUS/GSM managing GSCs associated to a specific group. Let's assume you would like to "break" your network into 2 groups. Here is how you should start the GigaSpaces runtime environment:

1. Run gs-agent starting LUS/GSM with GroupX:

{% highlight java %}
export LOOKUPGROUPS=GroupX
gs-agent.sh gsa.global.lus 0 gsa.lus 1 gsa.global.gsm 0 gsa.gsm 1 gsa.gsc 0
{% endhighlight %}

2. Run gs-agent that will start GSCs with GroupX (4 GGCs with this example):

{% highlight java %}
export LOOKUPGROUPS=GroupX
gs-agent.sh gsa.global.lus 0 gsa.lus 0 gsa.global.gsm 0 gsa.gsm 0 gsa.gsc 4
{% endhighlight %}

3. Run gs-agent starting LUS/GSM with GroupY:

{% highlight java %}
export LOOKUPGROUPS=GroupX
gs-agent.sh gsa.global.lus 0 gsa.lus 1 gsa.global.gsm 0 gsa.gsm 1 gsa.gsc 0
{% endhighlight %}

4. Run gs-agent that will start GSCs with GroupY (2 GGCs with this example):

{% highlight java %}
export LOOKUPGROUPS=GroupY
gs-agent.sh gsa.global.lus 0 gsa.lus 0 gsa.global.gsm 0 gsa.gsm 0 gsa.gsc 2
{% endhighlight %}

5. Deploy a space into GroupX GSCs

{% highlight java %}
export LOOKUPGROUPS=GroupX
gs deploy-space -cluster schema=partitioned total_members=4 spaceX
{% endhighlight %}

6. Deploy a space into GroupY GSCs

{% highlight java %}
export LOOKUPGROUPS=GroupY
gs deploy-space -cluster schema=partitioned total_members=2 spaceY
{% endhighlight %}

## Running Multiple Locators
You may have a set of LUS/GSM managing GSCs associated to a specific locaator. Let's assume you would like to "break" your network into 2 groups using different lookup locators. Here is how you should start the GigaSpaces runtime environment:

1. Run gs-agent starting LUS/GSM with a lookup service listening on port 8888:
{% highlight java %}
export LUS_JAVA_OPTIONS=-Dcom.sun.jini.reggie.initialUnicastDiscoveryPort=8888
export LOOKUPLOCATORS=127.0.0.1:8888
export EXT_JAVA_OPTIONS=-Dcom.gs.multicast.enabled=false
gs-agent.sh gsa.global.lus 0 gsa.lus 1 gsa.global.gsm 0 gsa.gsm 1 gsa.gsc 0
{% endhighlight %}
2. Run gs-agent that will start GSCs using the lookup listening on port 8888 (4 GGCs with this example):
{% highlight java %}
export LOOKUPLOCATORS=127.0.0.1:8888
export EXT_JAVA_OPTIONS=-Dcom.gs.multicast.enabled=false
gs-agent.sh gsa.global.lus 0 gsa.lus 0 gsa.global.gsm 0 gsa.gsm 0 gsa.gsc 4
{% endhighlight %}
3. Run gs-agent starting LUS/GSM with a lookup service listening on port 9999:
{% highlight java %}
export LUS_JAVA_OPTIONS=-Dcom.sun.jini.reggie.initialUnicastDiscoveryPort=9999
export LOOKUPLOCATORS=127.0.0.1:8888
export EXT_JAVA_OPTIONS=-Dcom.gs.multicast.enabled=false
gs-agent.sh gsa.global.lus 0 gsa.lus 1 gsa.global.gsm 0 gsa.gsm 1 gsa.gsc 0
{% endhighlight %}
4. Run gs-agent that will start GSCs using the lookup listening on port 9999 (2 GGCs with this example):
{% highlight java %}
export LOOKUPLOCATORS=127.0.0.1:9999
export EXT_JAVA_OPTIONS=-Dcom.gs.multicast.enabled=false
gs-agent.sh gsa.global.lus 0 gsa.lus 0 gsa.global.gsm 0 gsa.gsm 0 gsa.gsc 2
{% endhighlight %}
5. Deploy a space using lookup listening on port 8888
{% highlight java %}
export LOOKUPLOCATORS=127.0.0.1:8888
gs deploy-space -cluster schema=partitioned total_members=4 spaceX
{% endhighlight %}
6. Deploy a space using lookup listening on port 9999

{% highlight java %}
export LOOKUPLOCATORS=127.0.0.1:9999
gs deploy-space -cluster schema=partitioned total_members=2 spaceY
{% endhighlight %}

{% tip %}
On top of the Lookup service, there is also an alternative way to export the space proxy - it is done via the RMI registry (JNDI). It is started by default within any JVM running a GSC/GSM. By default, the port used is 10098 and above. This option should be used only in special cases where somehow there is no way to use the default lookup service. Since this is the usual RMI registry, it suffers from known problems, such as being non-distributed, non-highly-available, etc.
{% endtip %}

The lookup service runs by default as a standalone JVM process started by the GSA. You can also embed it to run together with the GSM. In general, you should run 2 lookup services per system. Running more than 2 lookup services may cause an overhead, due to the chatting and heartbeat mechanism performed between the services and the lookup service, to signal the existence of the service.

# Zones
The [GigaSpaces Zone]({%latestjavaurl%}/configuring-the-processing-unit-sla.html) allows you to "label" a running GSC(s) before starting it. The GigaSpaces **Zone** should be used to isolate applications and a Data-Grid running on the same network. It has been designed to allow users to deploy a processing unit into specific set of GSCs where all these **sharing the same set of LUSs and GSMs**.

The **Zone** property can be used for example to deploy your Data-Grid into a specific GSC(s) labeled with specific zone(s). The zone is specified prior to the GSC startup, and cannot be changed once the GSC has been started.
![zones.jpg](/attachment_files/sbp/zones.jpg)

{% tip %}
You should make sure you have an adequate number of GSCs running, prior to deploying an application whose SLA specified a specific zone.
{% endtip %}

To use Zones when deploying your PU you should:

1. Start the GSC using the `com.gs.zones` system property. Example:

{% highlight java %}
export EXT_JAVA_OPTIONS=-Dcom.gs.zones=webZone ${EXT_JAVA_OPTIONS}
gs-agent gsa.gsc 2
{% endhighlight %}

2. Deploy the PU using the `-zones` option. Example:

{% highlight java %}
gs deploy -zones webZone myWar.war
{% endhighlight %}

## Running Multiple Zones
You may have a set of LUS/GSM managing multiple zones (recommended) or have a separate LUS/GSM set per zone. In such a case (set of LUS/GSM managing multiple zones) you should run these in the following manner:

1. Run gs-agent on the machines you want to have the LUS/GSM:
{% highlight java %}
gs-agent.sh gsa.global.lus 0 gsa.lus 1 gsa.global.gsm 0 gsa.gsm 1 gsa.gsc 0
{% endhighlight %}
2. Run gs-agent that will start GSCs with zoneX (4 GGCs with this example):
{% highlight java %}
export EXT_JAVA_OPTIONS=-Dcom.gs.zones=zoneX ${EXT_JAVA_OPTIONS}
gs-agent.sh gsa.global.lus 0 gsa.lus 0 gsa.global.gsm 0 gsa.gsm 0 gsa.gsc 4
{% endhighlight %}
3. Run gs-agent that will start GSCs with zoneY (2 GGCs with this example):
{% highlight java %}
export EXT_JAVA_OPTIONS=-Dcom.gs.zones=zoneY ${EXT_JAVA_OPTIONS}
gs-agent.sh gsa.global.lus 0 gsa.lus 0 gsa.global.gsm 0 gsa.gsm 0 gsa.gsc 2
{% endhighlight %}

Note that with XAP 7.1.1 new variables provided that allows you to set different JVM arguments for GSC,GSM,LUS,GSA separately (GSA_JAVA_OPTIONS , GSC_JAVA_OPTIONS , GSM_JAVA_OPTIONS , LUS_JAVA_OPTIONS).

# Capacity Planning
In order to estimate the amount of total RAM and CPU required for your application, you should take the following into consideration:

- The Object Footprint within the space.
- Active Clients vs. Cores vs. Heap Size.
- The number of space partitions and backups.

The [Capacity Planning](./capacity-planning.html) section provides a detailed explanation how to estimate the resources required.

# PU Packaging and CLASSPATH

## User PU Application Libraries
A [Processing Unit]({%latestjavaurl%}/the-processing-unit-structure-and-configuration.html) JAR file, or a [Web Application]({%latestjavaurl%}/web-jetty-processing-unit-container.html) WAR file should include within its lib folder, all the necessary JARs required for the application. Resource files should be placed within one of the JAR files within the PU JAR, located under the lib folder. In addition, the PU JAR should include the pu.xml within the `META-INF\spring` folder.
In order to close LRMI threads when closing application,please use:LRMIManager.shutdown().

## Data-Grid PU Libraries
When deploying a Data-Grid PU, it is recommended that you include all space classes and their dependency classes as part a PU JAR file. This PU JAR file should include a pu.xml within the META-INF\spring, to include the space declarations and relevant tuning parameters.

## GS-UI Libraries
It is recommended that you include all space classes and their dependency classes as part of the GS-UI CLASSAPTH . This makes sure that you can query the data via the GS-UI. To set the GS-UI classpath, set the `POST_CLASSPATH` variable prior to calling the GS-UI script to have the application JARs locations.

{% tip %}
To avoid the need to load the same library into each PU instance classloader running within the GSC, you should place common libraries (such as JDBC driver, logging libraries, Hibernate libraries and their dependencies) at the `<gigaspaces-xap root>\lib\optional\pu-common` folder. You may specify the location of this folder using the `com.gs.pu-common` system property.
{% endtip %}

# JVM Tuning
In most cases, the applications using GigaSpaces are leveraging machines with very fast CPUs, where the amount of temporary objects created is relatively large for the JVM garbage collector to handle with its default settings. This means careful tuning of the JVM is very important to ensure stable and flawless behavior of the application.

See below examples of JVM settings recommended for applications that might generate large number of temporary objects. In such situations you afford long pauses due to garbage collection activity.

{% tip %}
 In case your JVM is throwing an OutOfMemoryException, the JVM process should be restarted. you need to add this property to your JVM setting:
SUN -XX:+HeapDumpOnOutOfMemoryError -XX:OnOutOfMemoryError="kill -9 %p"
JROCKIT -XXexitOnOutOfMemory
{% endtip %}

These settings are good for cases where you are running a IMDG or when the business logic and the IMDG are collocated. For example IMDG with collocated Polling /Notify containers, Task executors or Service remoting:

JDK 1.6 - CMS mode - good for low latency scnearios:

{% highlight java %}
-server -Xms8g -Xmx8g -Xmn300m -XX:+UseConcMarkSweepGC -XX:+UseParNewGC
-XX:CMSInitiatingOccupancyFraction=60 -XX:+UseCMSInitiatingOccupancyOnly
-XX:MaxPermSize=256m -XX:+ExplicitGCInvokesConcurrent -XX:+UseCompressedOops
-XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled
{% endhighlight %}

JDK 1.7 - g1 mode - good for low latency scnearios:
{% highlight java %}
-server -Xms8g -Xmx8g -XX:+UseG1GC -XX:MaxGCPauseMillis=500 -XX:InitiatingHeapOccupancyPercent=50 -XX:+UseCompressedOops
{% endhighlight %}

Advanced options for JDK 1.7 with suggested values provided:
{% highlight java %}
-XX:MaxTenuringThreshold=25 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=8
-XX:G1ReservePercent=10 -XX:G1HeapRegionSize=32m
{% endhighlight %}

Some useful references:
http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html
http://stackoverflow.com/questions/8111310/java-7-jdk-7-garbage-collection-and-documentation
http://blog.sematext.com/2013/06/24/g1-cms-java-garbage-collector/
http://stackoverflow.com/questions/8262674/java7-g1-options
http://mpouttuclarke.wordpress.com/2013/03/13/large-java-heap-with-g1-collector-part-1/


{% tip %}
It is highly recommended that you use the latest JDK release when using these options.
{% endtip %}

{% tip %}
To capture the detailed information about garbage collection and how it is performing, you can add following parameters to JVM settings,

{% highlight java %}
-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:/path/to/log/directory/gc-log-file.log
{% endhighlight %}

Modify the path and file names appropriately. You will need to use a different file name for each invocation in order to not overwrite the files from multiple processes.
{% endtip %}

{% include /COM/jconsolejmapwarning.markdown %}

## Soft References LRU Policy
In the attempt to provide the highest level of performance possible, GigaSpaces takes advantage of features in the Java language that allows for effective caching in the face of memory demands. In particular, the [SoftReference](http://docs.oracle.com/javase/6/docs/api/java/lang/ref/SoftReference.html) class is used to store data up until there is a need for explicit garbage collection, at which point the data stored in soft references will be collected if possible. The system default is 1000, which represents the amount of time (in milliseconds) they will survive past their last reference. `-XX:SoftRefLRUPolicyMSPerMB` is the parameter that allows you to determine how much data is cached by allowing the JVM to control how long it endures; A recommended setting this value to **500** in active, dynamic systems:

{% highlight java %}
-XX:SoftRefLRUPolicyMSPerMB=500
{% endhighlight %}

The above means that softly reachable objects will remain alive for 500 milliseconds after the last time they were referenced.

## Permanent Generation Space
For applications that are using relatively large amount of third party libraries (PU using large amount of jars) the default permanent generation space size may not be adequate. In such a case, you should increase the permanent generation space size and please also refer to the suggested parameters above that should be used together with the other CMS parameters (-XX:+CMSClassUnloadingEnabled). Here are a suggested values:

{% highlight java %}
-XX:PermSize=512m -XX:MaxPermSize=512m
{% endhighlight %}

{% exclamation %} GigaSpaces is a Java-based product. .Net and C++ applications using GigaSpaces should also be aware the usage of the JVM libraries as part of the .Net and C++ client libraries.

See the [Tuning Java Virtual Machines]({%latestjavaurl%}/tuning-java-virtual-machines.html) section and the [Java SE 6 HotSpot Virtual Machine Garbage Collection Tuning](http://java.sun.com/javase/technologies/hotspot/gc/gc_tuning_6.html) for detailed JVM tuning recommendations.

# Space Memory Management
The Space supports two [Memory Management]({%latestjavaurl%}/memory-management-facilities.html) modes:

- `ALL_IN_CACHE` - this assumes all application data is stored within the space.
- `LRU` - this assumes some of the application data is stored within the space, and all the rest is stored in some external data source.

When running with `ALL_IN_CACHE`, the memory management:

- Stops clients from writing data into the space once the JVM utilized memory crosses the WRITE threshold (percentage of the heap max size).
- Throws a `MemoryShortageExecption` back to the client once the JVM utilized memory crosses the `high_watermark_percentage` threshold.

When running with `ALL_IN_CACHE`, you should make sure the default memory management parameters are tuned according the JVM heap size. A large heap size (over 2 G RAM) requires special attention. Here is an example of memory manager settings for a **10 G heap size**:

{% highlight java %}
<os-core:space id="space" url="/./mySpace" >
    <os-core:properties>
        <props>
            <prop key="space-config.engine.memory_usage.high_watermark_percentage">95</prop>
            <prop key="space-config.engine.memory_usage.write_only_block_percentage">94</prop>
            <prop key="space-config.engine.memory_usage.write_only_check_percentage">93</prop>
            <prop key="space-config.engine.memory_usage.low_watermark_percentage">92</prop>
        </props>
    </os-core:properties>
</os-core:space>
{% endhighlight %}

# Local Cache
The [local cache]({%latestjavaurl%}/local-cache.html) is used as a client side cache that stores objects the client application reads from the space. It speeds up repeated read operations of the same object. The `readById`/`readByIds` operation has a special optimization with a local cache that speeds up the retrieval time of the object from the local cache, in the case that it is already cached. The local cache evicts objects once a threshold is met. When there is a client application with a large heap size, you might want to configure the local cache eviction parameters to control the eviction behavior:

{% highlight java %}
<os-core:space id="space" url="jini://*/*/mySpace" />

<os-core:local-cache id="localCacheSpace" space="space" update-mode="PULL" >
    <os-core:properties>
        <props>
            <prop key="space-config.engine.cache_size">5000000</prop>
            <prop key="space-config.engine.memory_usage.high_watermark_percentage">75</prop>
            <prop key="space-config.engine.memory_usage.write_only_block_percentage">73</prop>
            <prop key="space-config.engine.memory_usage.write_only_check_percentage">71</prop>
            <prop key="space-config.engine.memory_usage.low_watermark_percentage">45</prop>
            <prop key=" space-config.engine.memory_usage.eviction_batch_size">1000</prop>
            <prop key="space-config.engine.memory_usage.retry_yield_time">100</prop>
            <prop key="space-config.engine.memory_usage.retry_count">20</prop>
        </props>
    </os-core:properties>
</os-core:local-cache>
<os-core:giga-space id="gigaSpace" space="localCacheSpace"/>
{% endhighlight %}

- With the above parameters, the local cache is evicted once the client JVM memory utilization crosses the 75% threshold (or there are more than 5,000,000 objects within the local cache).
- Data is evicted in batches of 1,000 objects, trying to lower the memory utilization to 45%.
- If the eviction mechanism does not manage to lower the utilization to 45%, it has another 20 tries and stops.
- After each eviction activity, and before measuring the memory utilization, a pause of 100 ms happens, to allow the JVM to release the evicted objects.

{% tip %}
The `space-config.engine.cache_size` is set to a large value, to instruct the local cache to evict, based on the available free memory, and not based on the total number of objects within the local cache.
{% endtip %}

# Primaries Space Distribution
By default, when running GSCs on multiple machines and deploying a space with backups, GigaSpaces tries to provision primary spaces to all available GSCs across all the machines.
The `max-instances-per-vm` and the `max-instances-per-machine` deploy parameters should be set when deploying your Data-Grid, to determine how the deployed Processing Unit (e.g. space) is provisioned into the different running GSCs.

Without setting the `max-instances-per-vm` and the `max-instances-per-machine`, GigaSpaces might provision a primary and a backup instance of the same partition into GSCs running on the same physical machine. To avoid this behavior, you should set the `max-instances-per-vm=1` and the `max-instances-per-machine=1`. This makes sure that the primary and backup instances of the same partition are provisioned into different GSCs running on different machines. If there is one machine running GSCs and `max-instances-per-machine=1`, **backup instances are not provisioned**.

Here is an example of how you should deploy a Data-Grid with 4 partitions, with a backup per partition (total of 8 spaces), where you have 2 spaces per GSC, and the primary and backup are not running on the same box (even when you have other GSCs running):

{% highlight java %}
gs deploy-space -cluster schema=partitioned-sync2backup total_members=4,1
	-max-instances-per-vm 2  -max-instances-per-machine 1 MySpace
{% endhighlight %}

{% tip %}
To limit the amount of instances a GSC may host use the `com.gigaspaces.grid.gsc.serviceLimit` system property for the GSC JVM. This is very useful when you would like to have a single instance per GSC and avoid a situation the GSM might provision multiple instances into the same GSC after a failure event.
{% endtip %}

{% tip %}
After a machine startup (where GSCs are started), when the ESM is not used to deploy the IMDG, spaces do not "rebalance" across all the machines to have an even number of primaries per machine. You may have machines running more (or all) primaries, and another machine running only backups.
{% endtip %}

# Rebalancing - Dynamic Data Redistribution

## Automatic Rebalancing
GigaSpaces supports automatic discovery, rebalancing (aka Dynamic Redistribution of Data) and expansion/contraction of the IMDG **while the application is running**. When deploying an IMDG, the system partitions the data (and the collocated business logic) into logical partitions. You may choose the number of logical partitions or let GigaSpaces calculate this number.

The logical partitions may initially run on certain containers, and later get relocated to other containers (started after the data grid has been deployed) on other machines, thus allowing the system to expand and increase its memory and CPU capacity while the application is still running. The number of logical partitions and replicas per partition should be determined at deployment time.  The number of containers hosting the IMDG instances may be changed at runtime.

![rebalance_util.jpg](/attachment_files/sbp/rebalance_util.jpg)

The component that is responsible to scale the IMDG at runtime is called the Elastic Service Manager (ESM) and it is used with the [Elastic Processing Unit]({%latestjavaurl%}/elastic-processing-unit.html):
![flow.gif](/attachment_files/sbp/flow.gif)

{% tip %}
When using the [Elastic Processing Unit]({%latestjavaurl%}/elastic-processing-unit.html), instances will be continuously rebalanced across all available machines.
{% endtip %}

{% comment %}

## Manual Rebalancing
Production machines may be restarted every few days, may fail abnormally and then restarted, or new machines may be started and added to the grid. To allow even distribution of primary IMDG instances or to simply stretch the running instances across all available machines manually, you may use the [Rebalancing utility](/attachment_files/sbp/Rebalance.zip). The utility spreads primary and backup IMDG instances evenly across all the machines running GSCs. See full details how to run this utility, within the readme at the [Rebalance.zip](/attachment_files/sbp/Rebalance.zip).

## How GigaSpaces rebalancing works?
GigaSpaces runtime environments differentiates between a Container (GSC) or **Grid Node** that is running within a single JVM instance and an **IMDG Node**, also called a logical partition. A partition has one primary instance and zero or more backup instances.

A grid node hosts zero or more IMDG nodes, typically both primary and backup instances that belong to different partitions. IMDG nodes (or any other deployed PU instance) may relocate between grid nodes at runtime. A GigaSpaces IMDG may expand its capacity or shrink its capacity at runtime by adding or removing grid nodes and relocating existing logical partitions to newly started containers.

If the system selects to relocate a primary instance, it will first switch its activity mode into a backup and the existing backup instance of the same logical partition will be switched into a primary mode. Once the new backup will be relocated, it will recover its data from the existing primary. This how the IMDG expands itself without disruption to the client application.

GigaSpaces rebalancing allows you to have total control on which logical partition is moving, where it is moving to and whether to move a primary to a backup instance. You can increase the capacity or rebalance the IMDG automatically or manually. This is a very important capability in production environments. Without such control, the system might choose to move partitions into containers that are fully consumed or move too many instances into the same container. This will obviously crash the system.

GigaSpaces can adjust the high-availability SLA dynamically to cope with the current system memory resources. This means that if there is not sufficient memory capacity to instantiate all the backup instances, GigaSpaces will relax the SLA in runtime to allow the system to continue and run. Once the system identifies that there are enough resources to accommodate all the backups, it will start the missing backups automatically.

The number of logical partitions is determined at deploy time, but the amount of hosting containers may be modified in runtime by adding or removing containers, automatically (based on SLA) or manually. When the ESM is used to deploy the IMDG, GigaSpaces calculates the number of logical partitions during the deploy time based on the IMDG SLA. When the ESM is not being used to deploy the IMDG, GigaSpaces does not provide a default value for the number of logical partitions, so you should provide such. You can pick any number that you may find relevant for your system. Usually, it will be a number that will match the amount of initial containers you have multiply with a scaling factor - a number that determines how much the IMDG might need to expand its capacity without any downtime. This allows the IMDG to scale while the client application is running.

{% endcomment %}

# Storage Type - Controling Serialization

When a client application accessing a remote space (using a clustered topology or non-clustered) the data is serialized and sent over the network to the relevant JVM hosting the target space partition. The serialization involves some overhead. The [Storage Type]({%latestjavaurl%}/storage-types---controlling-serialization.html) decoration allows you to control the serialization behavior when non-primitive fields used with your space class.

{% inittab %}

{% tabcontent Object Mode %}
![storage-type-object.jpg](/attachment_files/sbp/storage-type-object.jpg)
{% endtabcontent %}

{% tabcontent Binary Mode %}
![storage-type-binary.jpg](/attachment_files/sbp/storage-type-binary.jpg)
{% endtabcontent %}

{% tabcontent Compressed Mode %}
![storage-type-compressed.jpg](/attachment_files/sbp/storage-type-compressed.jpg)
{% endtabcontent %}

{% endinittab %}

## OBJECT Storage Type
The `OBJECT` (default) serialization mode (called also native) performs serialization of all non-primitive fields at the client side, and then de-serialize these at the space side before stored within the space.

This mode is optimized for scenarios when there is a **business logic colocated with the space** (e.g. notify/polling container) or when having business logic that is sent to be executed within the space (e.g. Task Executor). The colocated business logic access non-primitive space object fields without going through any serialization. This speeds up any activity performed by the colocated business logic. The downside with this mode, is the relative overhead associated with the remote client due-to the serialization/de-serialization involved with non-primitive space object fields.

## BINARY Storage Type
When having space objects that embed large collections (e.g. List, Map data types) where there is no colocated business logic running with the space (e.g. polling/notify container colocated with the space), you should use the `BINARY` Storage Type.

When running with this mode, the collections within the space object are serialized at the client side but are **not de-serialized** at the space side before stored within the space; these are stored in their binary form. When reading the space object back into the client side, these collections are sent back into the client application without going through any serialization at the space side (as they are already stored in their binary serialized form), and de-serialized at the client side.  Due-to this optimization, this mode speeds up write and read performance when the space object involves collections with relatively large amount of elements.

You may control the Storage type at the space level, class level or field level.

See the [Controlling Serialization]({%latestjavaurl%}/storage-types---controlling-serialization.html) for more details.

# Runtime Files Location
GigaSpaces generates some files while the system is running. You should change the location of the generated files location using the following system properties. See below how:

{: .table .table-bordered}
| System Property | Description | Default |
|:----------------|:------------|:--------|
|`com.gigaspaces.logger.RollingFileHandler.filename-pattern`|The location of log files and their file pattern.| `<gigaspaces-xap root>\logs`|
|`com.gs.deploy`|The location of the deploy directory of the GSM. |`<gigaspaces-xap root>\deploy`|
|`com.gs.work`|The location of the work directory of the GSM and GSC. Due to the fact that this directory is critical to the system proper function, it should be set to a local storage in order to avoid failure in case of network failure when a remote storage is used.|`<gigaspaces-xap root>\work`|
|`user.home`|The location of system defaults config. Used by the GS-UI, and runtime system components.| |
|`com.gigaspaces.lib.platform.ext` | PUs shared classloader libraries folder. PU jars located within this folder loaded once into the **JVM system classloader** and shared between all the PU instnaces classloaders within the GSC. In most cases this is a better option than the `com.gs.pu-common` for JDBC drivers and other 3rd party libraries. This is useful option when you  want multiple processing units to share the same 3rd party jar files and do not want to repackage the processing unit jar whenever one of these 3rd party jars changes.| `<gigaspaces-xap root>\lib\platform\ext`|
|`com.gs.pu-common`|The location of common classes used across multiple processing units. The libraries located within this folder **loaded into each PU instance classloader** (and not into the system classloader as with the `com.gigaspaces.lib.platform.ext`. |`<gigaspaces-xap root>\lib\optional\pu-common`|
|`com.gigaspaces.grid.gsa.config-directory`|The location of the GSA configuration files. [The GigaSpaces Agent]({%latestjavaurl%}/service-grid.html#gsa) (GSA) manages different process types. Each process type is defined within this folder in an xml file that identifies the process type by its name. |`<gigaspaces-xap root>\config\gsa`|
|`java.util.logging.config.file`| It indicates file path to the Java logging file location. Use it to enable finest logging troubleshooting of various GigaSpaces Services. You may control this setting via the `GS_LOGGING_CONFIG_FILE_PROP` environment variable.| `<gigaspaces-xap root>\config\gs_logging.properties`|

{% exclamation %} The `com.gigaspaces.lib.platform.ext` and the `com.gs.pu-common` are useful to decrease the deployment time in case your processing unit **contains a lot of 3rd party jars files**. In such case, each GSC will download the processing unit jar file (along with all the jars it depends on) to its local working directory from the GSM, and in case of large deployments spanning tens or hundreds of GSCs this can be quite time consuming. In such cases you should consider **placing the jars on which your processing unit depends on** in a shared location on your network, and then point the `com.gs.pu-common` or `com.gigaspaces.lib.platform.ext` directory to this location.

# Log Files
GigaSpaces generates log files for each running component . This includes GSA, GSC, GSM, Lookup service and client side. By default, these are created within the `<gigaspaces-xap-root>\logs` folder. After some time you might end up with a large number of files that are hard to maintain and search. You should backup old log files or delete these. You can use the [logging backup-policy]({%latestjavaurl%}/backing-up-files-with-a-custom-policy.html) to manage your log files.

# Hardware Selection
The general rule when selecting the HW to run GigaSpaces would be: The faster the better. Multi-core machines with large amount of memory would be most cost effective since these will allow GigaSpaces to provide ultimate performance leveraging large JVM heap size handling simultaneous requests with minimal thread context switch overhead.

Running production systems with 30G-50G heap size is doable with some JVM tuning when leveraging multi-core machines. The recommended HW is [Intel® Xeon® Processor 5600 Series](http://ark.intel.com/ProductCollection.aspx?series=47915). Here is an example for [recommended server configuration](http://www.cisco.com/en/US/products/ps10280/prod_models_comparison.html):

{: .table .table-bordered}
|Model|Cisco UCS B200 M2 Blade Server|Cisco UCS B250 M2 Extended Memory Blade Server|
|:----|:----------------------------:|:--------------------------------------------:|
|Processor Sockets|2|2|
|Processors Supported|Intel Xeon processor 5600 Series|Intel Xeon processor 5600 Series|
|Memory Capacity|12 DIMMs; up to 192 GB|48 DIMMs; up to 384 GB|
|Memory Size and Speed|4, 8, and 16 GB DDR3; 1066 MHz and 1333 MHz|4 and 8 GB DDR3; 1066 MHz and 1333 MHz|
|Internal Disk Drive|2x 2.5-in. SFF SAS or 15mm SATA SSD|2x 2.5-in. SFF SAS or 15mm SATA SSD|
|Integrated Raid|0,1|0,1|
|Mezzanine I/O Adapter Slots|1|2|
|I/O Throughput|Up to 20 Gbps|Up to 40 Gbps|
|Form Factor|Half width|Full width|
|Max. Servers per Chassis|8|4|

## CPU
Since most of the application activities are conducted in-memory, the CPU speed impacts your application performance fairly drastically. You might have a machine with plenty of CPU cores, but a slow CPU clock speed, which eventually slows down the application or the Data-Grid response time. So as a basic rule, pick the fastest CPU you can find. Since the Data-Grid itself and its container are highly multi-threaded components, it is important to use machines with more than a single core to host the GSC to run your Data-Grid or application. A good number for the amount of GSCs per machine is half of the total number of cores.

## Disk
Prior to XAP 7.1, GigaSpaces Data-Grid did not overflow to a disk, and does not require a large disk space to operate.  Still, log files are generated, and for these you need at least 100M of free disk size per machine running GSC(s). Make sure you delete old log files or move them to some backup location. XAP Data-Grid may overflow data to disk when there is a long replication disconnection or delay, the location of the work directory should be on a local storage at each node in order to make this replication back log data always available to the node, this storage should have enough space to store the replication back log as explained in [Controlling the Replication Redo Log]({%latestjavaurl%}/controlling-the-replication-redo-log.html) page.

# OS Considerations
In general, GigaSpaces runs on every OS supporting the JVM technology (Windows, Linux, Solaris, AIX, HP, etc). No special OS tuning is required for most of the applications. See below for OS tuning recommendations that most of the applications running on GigaSpaces might need.

## File Descriptors
The GigaSpaces LRMI layer opens network connections dynamically. With large scale applications or with clients that are running a large number of threads accessing the Data-Grid, you might end up having a large number of file descriptors used.

You might have multiple JVMs running on the machine. This might need to increase the default max user processes value.

The Linux OS by default has a relatively small number of file descriptors available and max user processes (1024). You should make sure that your standalone clients, or GSA/GSM/GSC/LUS using a user account which have its **maximum open file descriptors ** (open files) and **max user processes** configured to a high value. A good value is 32768.

Setting the **max open file descriptors** and **max user processes** is done via the following call:

{% highlight java %}
ulimit -n 32768 -u 32768
{% endhighlight %}

Alternatively, you should have the following files updated:

{% highlight java %}
/etc/security/limits.conf

- soft    nofile          32768
- hard    nofile          32768

/etc/security/limits.d/90-nproc.conf

- soft nproc 32768
{% endhighlight %}

