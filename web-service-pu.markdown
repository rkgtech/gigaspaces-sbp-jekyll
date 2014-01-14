---
layout: post
title:  Web Service PU
categories: SBP
parent: data-access-patterns.html
weight: 1800
---

{% compositionsetup %}

{% tip %}
**Summary:** {% excerpt %}Web Service Processing Unit{% endexcerpt %}<br/>
**Author**: Shay Hassidim, Deputy CTO, GigaSpaces<br/>
**Recently tested with GigaSpaces version**: XAP 8.0.<br/>
**Last Update:** May 2011<br/>

{% toc minLevel=1|maxLevel=1|type=flat|separator=pipe %}

{% endtip %}

# Overview
This example illustrates a simple web service packaged as a WAR using [Apache CXF](http://cxf.apache.org) 2.4. Web service is also interacting with a space and performing read and write operations on behalf of the client.

{% indent %}
![web_service_pu.jpg](/attachment_files/sbp/web_service_pu.jpg)
{% endindent %}

For simplicity:

- Space is collocated with the web service.
- Example does not include HTTP Load balancer and Load balancer Agent.

The example is based on the [sample](http://cxf.apache.org/docs/sample-projects.html) found within the CXF package under the `\apache-cxf-2.4.0\samples\wsdl_first` folder. This sample shows how to build and call a web service using a given WSDL (also called Contract First).

# What the Sample Web Service is Doing?
The web service performs the following space operations:

- Write a Customer object into the space - See the `com.example.customerservice.server.CustomerServiceImpl.updateCustomer (Customer customer)`.
- Read Customer objects from the space - See the `com.example.customerservice.server.CustomerServiceImpl.getCustomersByName (String name)`.

# Running the Web Service

Step 1. Download the [Web Service WAR file](/attachment_files/sbp/CustomerServicePort.war).
Step 2. Download the [Web Service project](/attachment_files/sbp/WebServicePU.zip). This includes the entire source code for the client and the web service.
Step 3. Setup the application project class path libraries. See the [Client Application Project jars](#Client Application Project jars) for details.
Step 4. Start GigaSpaces agent.

{% highlight java %}
gs-agent
{% endhighlight %}

Step 5. Deploy the WAR file. You may use the GS-UI or the CLI.

{% highlight java %}
gs deploy CustomerServicePort.war
{% endhighlight %}

Step 6. Run the Client application.

{% highlight java %}
java com.example.customerservice.client.CustomerServiceClient
{% endhighlight %}

See the [The Client Application](#The Client Application) for details.

# Web Service Configuration
The WAR file includes the web service configuration. It is placed within `\CustomerServicePort\WEB-INF\cxf-servlet.xml`.
The configuration includes:

- The Space bean.
- The GigaSpace bean. It is injected into the Service implementation automatically via the `@GigaSpaceContext` annotation.
- The Service implementation.
- The endpoint Configuration.

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      	xmlns:jaxws="http://cxf.apache.org/jaxws"
      	xmlns:soap="http://cxf.apache.org/bindings/soap"
      	xmlns:os-core="http://www.openspaces.org/schema/core"
      	xmlns:os-events="http://www.openspaces.org/schema/events"
      	xmlns:os-remoting="http://www.openspaces.org/schema/remoting"
      	xsi:schemaLocation="
      	http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://cxf.apache.org/bindings/soap http://cxf.apache.org/schemas/configuration/soap.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
       	http://www.openspaces.org/schema/core http://www.openspaces.org/schema/{%latestxaprelease%}/core/openspaces-core.xsd
       	http://www.openspaces.org/schema/events http://www.openspaces.org/schema/{%latestxaprelease%}/events/openspaces-events.xsd
       	http://www.openspaces.org/schema/remoting http://www.openspaces.org/schema/{%latestxaprelease%}/remoting/openspaces-remoting.xsd
       	http://cxf.apache.org/jaxws http://cxf.apache.org/schemas/jaxws.xsd">

	<bean id="service" class="com.example.customerservice.server.CustomerServiceImpl" />

	<os-core:giga-space-context/>
  	<os-core:space id="space" url="/./webServiceSpace">
  	</os-core:space>
  	<os-core:giga-space id="gigaSpace" space="space"/>

  	<jaxws:endpoint xmlns:customer="http://customerservice.example.com/"
            id="CustomerServiceHTTP"
            address="http://localhost:8080/CustomerServicePort/services/CustomerServicePort"
            serviceName="customer:CustomerServiceService"
            endpointName="customer:CustomerServiceEndpoint"
            implementor="#service"
            implementorClass="com.example.customerservice.server.CustomerServiceImpl">
    </jaxws:endpoint>
</beans>
{% endhighlight %}

{% note %}
There is no `pu.xml` used with this war. cxf-servlet.xml has all the required configuration.
{% endnote %}

# Space Proxy Injection into the Service
The space proxy injected into the `com.example.customerservice.server.CustomerServiceImpl` using `@GigaSpaceContext` annotation:

{% highlight java %}
public class CustomerServiceImpl implements CustomerService , InitializingBean{
....
    @GigaSpaceContext
    GigaSpace space;
{% endhighlight %}

# WSDL Contract
The WSDL used by the web service includes following:

{% highlight xml %}
<wsdl:service name="CustomerServiceService">
	<wsdl:port name="CustomerServicePort" binding="tns:CustomerServiceServiceSoapBinding">
		<soap:address location="http:/localhost:8080/CustomerServicePort/services/CustomerServicePort"/>
	</wsdl:port>
</wsdl:service>
{% endhighlight %}

WSDL is packaged in the CustomerServicePort.war file as `\WEB-INF\CustomerService.wsdl`.

# The Client Application
The client application:

- Call the `CustomerServiceImpl.updateCustomer (Customer customer)`.
- Call the `CustomerServiceImpl.getCustomersByName (String name)`.

## Running the Client
You may test the web service using a client application running as a standalone Java application, Spring Application or via [SOAP UI](http://www.soapui.org).

### Standalone Java Application
The `com.example.customerservice.client.CustomerServiceClient` can be used as a standalone Java application to invoke the service. Run the `main` to run the client:

{% highlight java %}
java com.example.customerservice.client.CustomerServiceClient
{% endhighlight %}

### Spring Application
The `com.example.customerservice.client.CustomerServiceSpringClient` can be used as a Spring client application to invoke the service. The `\resources\client-applicationContext.xml` used as the client application context. Make sure you have the `resources` folder as part of the client classpath when running the client.
Run the `main` to run the client:

{% highlight java %}
com.example.customerservice.client.CustomerServiceSpringClient
{% endhighlight %}

## Expected Output
When running the client application you should get the following output:

{% highlight java %}
Updating Customer info named:Smith
Reading customer named:Smith
Found 1 customers with Name:Smith
All calls were successful
{% endhighlight %}

### SOAP UI
You can use the [SOAP UI](http://www.soapui.org) to test the web service.
Writing a Custmer object:

{% indent %}
![soap_ui1.jpg](/attachment_files/sbp/soap_ui1.jpg)
{% endindent %}

Reading a Custmer object:

{% indent %}
![soap_ui2.jpg](/attachment_files/sbp/soap_ui2.jpg)
{% endindent %}

The Server will have the following output:

{% highlight java %}
>>>>>>> update request was received
>>>>>>> Customer written into the space
found 1 Customers matching the name:Smith
{% endhighlight %}

## Checking the Data within the Space
When the client application running sucessfuly you will be able to see the Customer data within the space using the GS-UI.

{% indent %}
![web_serv_ui1.jpg](/attachment_files/sbp/web_serv_ui1.jpg)
{% endindent %}

Click the Query button to view the Customer Data:

{% indent %}
![web_serv_ui2.jpg](/attachment_files/sbp/web_serv_ui2.jpg)
{% endindent %}

# WAR file jars
The CustomerServicePort.war file includes the following jars within its `\CustomerServicePort\WEB-INF\lib` folder.
Spring and Jetty jars should not be included since they are loaded by GigaSpaces:

{% section %}

{% column %}

{% highlight java %}
aopalliance-1.0.jar
asm-3.3.jar
cxf-api-2.4.0.jar
cxf-common-utilities-2.4.0.jar
cxf-rt-bindings-soap-2.4.0.jar
cxf-rt-bindings-xml-2.4.0.jar
cxf-rt-core-2.4.0.jar
cxf-rt-databinding-jaxb-2.4.0.jar
cxf-rt-frontend-jaxws-2.4.0.jar
{% endhighlight %}

{% endcolumn %}

{% column %}

{% highlight java %}
cxf-rt-frontend-simple-2.4.0.jar
cxf-rt-transports-common-2.4.0.jar
cxf-rt-transports-http-2.4.0.jar
cxf-rt-transports-http-jetty-2.4.0.jar
cxf-rt-ws-addr-2.4.0.jar
cxf-tools-common-2.4.0.jar
geronimo-javamail_1.4_spec-1.7.1.jar
geronimo-servlet_3.0_spec-1.0.jar
jaxb-impl-2.1.13.jar
{% endhighlight %}

{% endcolumn %}

{% column %}

{% highlight java %}
junit-4.7.jar
neethi-3.0.0.jar
slf4j-api-1.6.1.jar
slf4j-jdk14-1.6.1.jar
stax2-api-3.1.1.jar
woodstox-core-asl-4.1.1.jar
wsdl4j-1.6.2.jar
xml-resolver-1.2.jar
xmlschema-core-2.0.jar
{% endhighlight %}

{% endcolumn %}

{% endsection %}

# Client Application Project jars
The Client project should include the following libraries:

- All jars within the `GigaSpaces root/lib/platform/jetty` folder.
- All jars within the `GigaSpaces root/lib/required` folder.
- All jars within the `GigaSpaces root/lib/optional/spring` folder.
- All jars within the CustomerServicePort.war `CustomerServicePort/WEB-INF/lib` folder
