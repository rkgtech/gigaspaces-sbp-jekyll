---
layout: post
title:  Global HTTP Session Sharing
categories: SBP
parent: solutions.html
weight: 200
---

{% tip %}
**Summary:** {% excerpt %}This pattern presents the HTTP Session sharing best pratice for IIS servers and .NET apps{% endexcerpt %}<br/>
**Author**:  Ronnie Guha<br/>
**Recently tested with GigaSpaces version**: XAP.NET 9.7.0 x86<br/>
**Last Update:** May 2014<br/>

{% endtip %}


# Global HTTP Session Sharing

It’s becoming increasingly important for organizations to share HTTP session data across multiple data centers, multiple web server instances or different types of web servers. Here are few scenarios where HTTP session sharing is required:
•	Multiple different Web servers running your web application - You may be porting your application from one web server to another and there will be a period of time when both types of servers need to be active in production.
•	Web Application is broken into multiple modules - When applications are modularized such that different functionalities are deployed across multiple server instances. For example, you may have login, basic info, check-in and shopping functionalities split into separate modules and deployed individually across different servers for manageability or scalability. In order for the user to be presented with a single, seamless, and transparent application, session data needs to be shared between all the servers.
•	Reduce Web application memory footprint - The web application storing all session within the web application process heap, consuming large amount of memory. Having the session stored within a remote process will reduce web application utilization avoiding garbage collocation and long pauses.
•	Multiple Data-Center deployment - You may need to deploy your application across multiple data centers for high-availability, scalability or flexibility, so session data will need to be replicated.

The GigaSpaces Global HTTP Session Sharing architecture allows users to deploy their web application across multiple data centers where the session is shared in real-time and in a transparent manner. The HTTP session is also backed by a data grid cluster within each data center for fault tolerance and high-availability.
With this solution, there is no need to deploy a database to store the session, so you avoid the use of expensive database replication across multiple sites. Setting up GigaSpaces for session sharing between each site is simple and does not involve any code changes to the application.

# Overview

GigaSpaces Global HTTP Session Management for .NET is designed to deliver the application maximum performance with ZERO application code changes. It features the following: - Reduced App/Web server memory footprint storing the session within a remote JVM on XAP.
•	No code changes required to share the session with other remote Web/App servers - Support Serialized and Non-Serialized Session attributes. Your attributes do not need to implement Serializable or Externalizable interface.
•	Application elasticity - Support session replication across different App/Web applications located within the same or different data-centers/clouds allowing the application to scale dynamically without any downtime.
•	Unlimited number of sessions and concurrent users support - Sub-millisecond session data access by using GigaSpaces In-Memory-Data-Grid.
•	Session replication over the WAN support - Utilizing GigaSpaces Multi-Site Replication over the WAN technology.
•	HTTP Session data access scalability - Session data can utilize any of the supported In-Memory-Data-Grid topologies ; replicated , partitioned , with and without local cache.
•	Transparent App/Web Failover - Allow app server restart without any session data loss.
•	Any session data type attribute support - Primitive and Non-Primitive (collections, user defined types) attributes supported.
•	Sticky session and Non-sticky session support - Your requests can move across multiple instances of web application seamlessly.
•	Atomic HTTP request session access support - multiple requests for the session attributes within the same HTTP request will be served without performing any remote calls. Master session copy will be updated when the HTTP request will be completed.


# Scenario
{%section%}
{%column width=80% %}
Before proceeding with the following sections, please find some more information on various session state modes here (http://msdn.microsoft.com/en-us/library/vstudio/ms178586(v=vs.100).aspx). As you might guess, there are pros and cons of these various approaches.
1)	Inproc mode has the best performance metrics but it only works best when you have a single server hosting a web application and web applications in which the end user is guaranteed to be re-directed to the one and only, and therefore, correct server. This mode is used when session data is not very critical as it can be in some other ways reconstructed. 
2)	Out of process/StateServer mode is the only way that allows you to really distribute your applications inside and outside your datacenters. This is the mode that one chooses when they cannot guarantee which web server would be serving the end user’s request. Herewith you can get the performance of reading from memory, reliability (and associated concerns) of having sessions managed by a separate process for all the web servers involved.
3)	SQL Server mode is used when very high levels of reliability are required. In other words, a session and its attributes have to be guaranteed re-construction on the fly and the application absolutely requires this kind of data reliability. Keep in mind that this mode has the worst performance of all (as quite easily understood) and trades off reliability for performance. This also allows you to globally distribute your applications across datacenters but relies heavily on clustering SQL Server databases across these data centers globally.

We will be concentrating our discussions on using an out of process, custom storage provider for achieving true global, lightweight distribution of your application that provides similar if not better reliability that other modes discussed above.

{%endcolumn%}
{%endsection%}

We will show you how you can implement this scenario with XAP.NET. A fully functional example is available [here](/download_files/GigaSpacesHTTPSessionSharing.zip).


# Various Scenarios

Below are various HTTP Session sharing scenarios:

## Scenario 1:
---------------
Consider a simple scenario where you have a single IIS server serving your web application in a non-clustered fashion. For most non-critical applications, this could very well be your deployment architecture. As seen in the following diagram, you could use an IMDG in scenarios as simple as the following or for as complex a distributed architecture as described in Scenario 4. In this particular scenario, your IIS server uses custom mode, specifies GigaSpaces as the storage provider, reads sessions from GigaSpaces space/cluster and serves your end users’ sessions reliably.
As you follow the instructions later on in this document to run the demo application, you would be using the same demo app and your web application, IIS and GigaSpaces to support the various scenarios. 
For scenario 1, you would not create a cluster in IIS. In other words, there is no need to add an extra server to your IIS server configuration. You would configure your GigaSpaces space as documented below or start the same from C:\GigaSpaces\XAP.NET 9.7.0 x86\NET v4.0.30319\Bin directory. Either way starts an instance with Url: /./mySpace


![](/pics/iis-pic1.png)


## Scenario 2:
---------------
Consider a little more complex scenario where you have a cluster of IIS servers serving your web application. For many semi-critical applications, this could very well be your deployment architecture. As seen in the following diagram, you could use an IMDG in almost the same way as you did in scenario 1. In this particular configuration, your IIS servers use custom mode, specify GigaSpaces as the storage provider, read sessions from GigaSpaces space/cluster and serve your end users’ sessions reliably. The session created on either one of the IIS servers is stored in GigaSpaces reliably and is retrieved from the web application even if a completely different IIS server fulfills the next HTTP request from the end user.
For scenario 2, you would create a cluster in IIS. In other words, you would need to add an extra IIS server to your IIS cluster configuration. You would then configure your GigaSpaces space as documented below or start the same from C:\GigaSpaces\XAP.NET 9.7.0 x86\NET v4.0.30319\Bin directory. Either way starts an instance with Url: /./mySpace

![](/pics/iis-pic2.png)

## Scenario 3:
Scenario 3:
---------------
Consider the next step in a more complex scenario where you have a cluster of IIS servers serving multiple web applications. For varied reasons, multiple modules of applications or multiple separate applications are required to share session data reliably between each other. For a large multi-module application, the following could very well be your deployment architecture and so would the case be for a multi-application deployment wherein you require session sharing (e.g. imagine a single sign-on requirement). As seen in the following diagram, you could use an IMDG in almost the same way as you did in scenario 1 and 2. In this particular configuration, your IIS servers use custom mode, specify GigaSpaces as the storage provider, read sessions from GigaSpaces space/cluster and serve your end users’ sessions reliably. The session created on either one of the IIS servers is stored in GigaSpaces reliably and is retrieved from the web application even if a completely different IIS server fulfills the next HTTP request from the end user. What’s more is that since multiple web applications can interact with the GigaSpaces IMDG (that’s running as a separate process), store and retrieve session data based on SessionID. 

For scenario 3, you would create a cluster in IIS or reuse the cluster you created in scenario 2. You could very well emulate a multiple application scenario by deploying another website in your IIS configuration and copying the demo application to another location to serve this separate web application. You would then configure your GigaSpaces space as documented below.

![](/pics/iis-pic3.png)

## Scenario 4:
Consider the following most complex deployment scenario where you have a cluster of IIS servers serving multiple web applications that are distributed across data centers over LAN/MAN/WANs.  Just like the requirements in Scenario 3, there are mission critical applications that need to be available globally (in a follow-the-sun approach). As seen in the following diagram, you could use an IMDG in almost the same way as you did in scenario 1, 2 and 3. The only difference would be in deploying GigaSpaces in a WAN distributed fashion (i.e. use the WAN gateway feature). In this particular configuration, your IIS servers use custom mode, specify GigaSpaces as the storage provider, read sessions from local/near-local GigaSpaces space/cluster and serve your end users’ sessions reliably. The session created on any one of the IIS servers is stored in GigaSpaces reliably, replicated globally and is retrieved from the web application even if a completely different IIS server in a completely different WAN/MAN fulfills the next HTTP request from the end user. This, of course, is also governed by how load balancers are spraying requests across various clusters of web applications. Similar to scenario 3, multiple web applications can interact with the GigaSpaces IMDG (that’s running as a separate processes), store and retrieve session data based on SessionID.

For scenario 4, you would create a cluster in IIS or reuse the cluster you created in scenario 2. You could very well emulate a multiple application scenario by deploying another website in your IIS configuration and copying the demo application to another location to serve this separate web application. In this scenario, you would configure your GigaSpaces space as a WAN gateway enabled, clustered deployment. The steps to do so is also documented below.


![](/pics/iis-pic4.png)

  

# Global HTTP Session Sharing with your ASP.NET Application:
The following instructions assumes that you have downloaded/installed the prerequisites:
•	Download the sample ASP.NET web application here….After downloading the file, extract it to a location on your drive.
•	Download Apache httpd server as it is used for load balancing in our example.
•	XAP.NET latest version. For our example, we are using x86 version of .NET

## Configure Apache, XAP.NET and multiple IIS servers serving this sample application:

1)	Install IIS by going to control panel->add remove programs->add/remove windows features-> IIS. After installation, create and deploy a website using IIS Manager. 

![](/pics/iis-pic5.png)

2)	Run IIS Manager and ensure that you configure the following correctly
a.	Configure your .NET version correctly for your machine on IIS Manager. 

![](/pics/iis-pic6.png)

b.	Application Pool – dedicate one to your sample website. Make sure this is set up correctly thus (if you are using 32 bit then ensure that “Enable 32-bit applications is set to true as well).

![](/pics/iis-pic7.png)

3)	Deploy your website and ensure it is configured correctly. 

![](/pics/iis-pic8.png)

a.	Deploy the application on multiple server instances – in other words, bind your site to two available HTTP ports. In this example, we are binding it to ports 70 and 90.

![](/pics/iis-pic9.png)

b.	Ensure that you generate/use a machine key for your site/application. Keep these keys handy as you will be using them in the web.config of your ASP.NET application.

![](/pics/iis-pic10.png)


c.	Ensure the Session State of your application is set to Custom
![](/pics/iis-pic11.png)

4)	Configure a load balancer/cluster manager to front all HTTP requests that will be ultimately routed to the IIS servers. Here’s an example of how Apache can be configured for this purpose. It assumes that you already have Apache httpd setup and running on a machine. Make changes in the httpd.conf file that include the following
a.	------------- add these lines in your httpd.conf file ------------------------------

{%highlight bash%}

<Proxy balancer://mycluster>
BalancerMember http://127.0.0.1:70 (the same port that you configured in IIS site bindings and the IP address where this IIS server resides)
BalancerMember http://127.0.0.1:90  (the same port that you configured in IIS site bindings and the IP address where this IIS server resides)
</Proxy>
ProxyPass /GigaSpacesHttpSessionDemo balancer://mycluster (this is a sample of a URL that will be used for the application)
ProxyPassReverse /GigaSpacesHttpSessionDemo balancer://mycluster (this is a sample of a URL that will be used for the application)
{%endhighlight%}


b.	------------------ uncomment the following lines in your httpd.conf file ------
{%highlight bash%}

LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule proxy_connect_module modules/mod_proxy_connect.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule status_module modules/mod_status.so
Listen 9000 (sample port that Apache will bind to and listen for requests on). You will be accessing the application thus: http://localhost:9000/GigaSpacesHttpSessionDemo/ (<IP address of the apache server>:<port apache is listening on>/<proxy url of the app you created before>

{%endhighlight%}

c.	Restart your httpd (Apache) server and ensure that your requests are being routed to both the severs

5)	Open up the GigaSpacesHTTPSessionSharing.sln in Visual Studio and edit your web.config file. It should have the entry like this

{%highlight xml%}

<system.web>
    <machineKey validationKey="E83700E53CC20595182307B502E45F0AE4C2ADBEB6CE5BBD2E88B2EE5AB8A6FDC633FBF69309A6D6D06B4B6DBF22E2DA96F0315ED97F65136DDCEFDB2342ASDADS"
                decryptionKey="06C735053A392342K2J424HJ59C51F50673"
                validation="SHA1" decryption="AES " />
    (these are the same machine key and decryptionKeys that you had generated in IIS)
    <sessionState mode="Custom"
    customProvider="GigaSpaceSessionProvider"
    cookieless="false"
    timeout="5"
    regenerateExpiredSessionId="true">

{%endhighlight%}

After this step, please make sure that you re-build the solution.
….

Note: 
If there are issues with ASP.NET (e.g. it is not installed or not configured properly) 
Run as administrator (command prompt)

{%highlight bash%}

C:\WINDOWS>C:\WINDOWS\Microsoft.NET\Framework\v4.0.30319\aspnet_regiis.exe –i

Microsoft (R) ASP.NET RegIIS version 4.0.30319.18408
Administration utility to install and uninstall ASP.NET on the local machine.
Copyright (C) Microsoft Corporation.  All rights reserved.
Start installing ASP.NET (4.0.30319.18408).
.........
Finished installing ASP.NET (4.0.30319.18408). 

{%endhighlight%}

If the above doesn’t work and you still find that ASP.NET state service is not installed run the following:

{%highlight bash%}

C:\WINDOWS>C:\WINDOWS\Microsoft.NET\Framework\v4.0.30319\aspnet_regiis.exe –u
C:\WINDOWS>C:\WINDOWS\Microsoft.NET\Framework\v4.0.30319\aspnet_regiis.exe -i
C:\WINDOWS>C:\WINDOWS\Microsoft.NET\Framework\v4.0.30319\aspnet_regiis.exe –c

{%endhighlight%}

6)	Ensure that you have the correct connectionstring defined in your web.config file. E.g. 

{%highlight xml%}

<add name="SpaceSessionProviderURL" connectionString="jini://*/*/mySpace?groups=XAP-9.7.0-ga-NET-4.0.30319-x86"/>

{%endhighlight}

-- mySpace would be the name of the space you would be deploying in the steps below. 

7)	Ensure that you have the correct connectionstring defined in Default.aspx.cs file line 14. 

{%highlight xml%}
String spaceUrl = "jini://*/*/mySpace?groups=XAP-9.7.0-ga-NET-4.0.30319-x86";
{%endhighlight%}


8)	For this example, we are using the x86 version of XAP.NET. 
a.	Make sure you have run gs-agent and gs-webgui from XAP.NET 9.7.0 x86\NET v4.0.30319\Bin directory. 
b.	Then deploy a space thus 
{%highlight bash%}

C:\GigaSpaces\XAP.NET 9.7.0 x86\NET v4.0.30319\Bin>gs-cli deploy-space –cluster total_members=1,1 mySpace

{%endhighlight%}

9)	Ensure that the application is running on both ports. You can point your browser to http://localhost:9000/GigaSpacesHttpSessionDemo/ and on two requests, you should see the port change. (e.g. 90 vs 70 as shown below)

![](/pics/iis-pic12.png)


And upon next request (since the load balancer is “round-robin”ing the requests)


![](/pics/iis-pic13.png)

10)	Now you should be able to enter session attributes and their values in one URL/session and retrieve the same from another.. e.g.
Open one instance of Chrome and enter some data into the form as below
 
![](/pics/iis-pic14.png)

And then, open another instance of that browser (or even a tab) and see the following:

![](/pics/iis-pic15.png)


Note that even though you were routed to a different IIS server (on port 70 vs the initial request that was routed to port 90), you are still able to read the same Session values. These values are coming from storage in XAP.NET. You can verify the same by looking for the session and the associated data in XAP.NET GUI

![](/pics/iis-pic16.png)


Verify the sessionId by querying the space

![](/pics/iis-pic17.png)

Verify the read/writes by looking at the statistics window on XAP.NET GUI


![](/pics/iis-pic18.png)

9)  The next step would be to close the browser instance and verify that the session still persists. After entering a session attribute and its value, close the browser instance and then restart it. You should see the session values persist.

![](/pics/iis-pic19.png)


10) The next step would be to stop IIS server and restart them to verify that the session still persists. Go to IIS manager and restart the website. Then go back to the same URL and confirm that the session still persists.

![](/pics/iis-pic20.png)

11) Validate that the same session variables are available from a totally different browser. E.g. if you had tested the previous steps using Chrome, now open up a Firefox instance and point to the same URL (please do so within the session timeout period).
Open up http://localhost:9000/GigaSpacesHttpSessionDemo/ in Chrome and enter some values


![](/pics/iis-pic21.png)


Now, copy the sessionId from here and open up firefox instance and point to this URL
http://localhost:9000/GigaSpacesHttpSessionDemo/?SessionId=2fcjxujrf0hmygfapfltukxi

![](/pics/iis-pic22.png)

Validate that even though this browser created a new session for itself, you could read the same session details based on the sessionId in the URL, from XAP.NET

12) Now, shutdown the gs-agent and also the gs-webgui. For the next demo, please follow the steps described in this document to deploy a WAN (http://docs.gigaspaces.com/sbp/wan-based-deployment.html)

13) The next step would be to start a multi cluster on a XAP.NET WAN Gateway and validate that the session data is being replicated across the clusters. Change the connection strings in the web.config and Default.aspx.cs files thus: 

a)	Ensure that you turn on wan in appsettings thus: 
{%highlight xml%}
<appSettings>
    <add key="wan" value="true"/>
</appSettings>
{%endhighlight%}


b)	Ensure that you have the correct connectionstring defined in your web.config file. 

{%highlight xml%}
<add name="SpaceSessionProviderURL" connectionString="jini://*/*/wanSpaceDE?groups=DE"/>
    <add name="SpaceSessionProviderURL-2" connectionString="jini://*/*/wanSpaceUS?groups=US"/>
    <add name="SpaceSessionProviderURL-3" connectionString="jini://*/*/wanSpaceRU?groups=RU"/>
{%endhighlight%}

 -- wanSpaceUS, wanSpaceDE, wanSpaceRU would be the names of the WAN spaces you would be deploying in the steps below. 

c)	Build the solution.

14) After the WAN based deployment is run successfully, you should be able to see the various PUs, spaces and gateways deployed (use gs-ui to see the Gigaspaces Management Center).

15) Go to the same URL as before: http://localhost:9000/GigaSpacesHttpSessionDemo and enter some values you should see the updates occurring on that session as below:

![](/pics/iis-pic23.png)

16) Now, open up another tab or browser and point to the following url : http://localhost:9000/GigaSpacesHttpSessionDemo/?sessionId=uvijcieybhqlselu4jblfu1m  Note that the sessionId is the ID from the session that you had just created in the previous step. You should be able to see the following:

![](/pics/iis-pic24.png)

And progressively as the object is clustered across the WAN

![](/pics/iis-pic25.png)

This shows that the value has been picked up from the wanSpaceRU, DE and US. 
17) Now, try stopping one of the startAgent* that you had run in the previous steps. This would shutdown one of the clusters. Important – please do not shut down the agent designated in the 

{%highlight xml%}

<add name="SpaceSessionProviderURL" connectionString="jini://*/*/wanSpaceDE?groups=DE"/> 
{%endhighlight%}

entry. You can shut down the spaces designated in the other entries in web.config – SpaceSessionProviderURL-2 and SpaceSessionProviderURL-3. 

Then point to the same URL as above http://localhost:9000/GigaSpacesHttpSessionDemo/?sessionId=uvijcieybhqlselu4jblfu1m 
You should find that the WAN gateway accomodates such situations as well (In our example, we shutdown startAgent-RU.bat).

![](/pics/iis-pic26.png)
