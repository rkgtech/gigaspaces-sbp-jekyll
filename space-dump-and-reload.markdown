---
layout: post
title:  Space Dump and Reload
categories: SBP
parent: production.html
weight: 1400
---

{% compositionsetup %}

{% tip %}
**Summary:** {% excerpt %}Space Dump and Reload{% endexcerpt %}<br/>
**Author**: Shay Hassidim, Deputy CTO, GigaSpaces<br/>
**Recently tested with GigaSpaces version**: XAP 7.1<br/>
**Last Update:** November 2010<br/>

{% toc minLevel=1|maxLevel=1|type=flat|separator=pipe %}

{% endtip %}

# Space Dump and Reload
When running a system using an in-memory data grid (IMDG), you may need to dump the data stored within the IMDG into a file and later reload it back. This might happen when you would like to copy the IMDG data from one system to another or when you would like to shutdown the system to perform some maintenance procedures or when you would like to upgrade the GigaSpaces release.

{% tip %}
If you would like to perform hardware maintenance activities without shutting down the system you can use a [rolling upgrade technique]({%latestjavaurl%}/deploying-onto-the-service-grid.html#HotDeploy).
{% endtip %}

The [Space Dump Utility](/attachment_files/sbp/spacedump.zip) copies the data currently stored within the IMDG and saves it into an embedded DB file used by a temporary space. Later, once you would like to reload the data back into the IMDG, the utility performs the procedure in a reverse manner, by reading the data from the file and copy it back into the IMDG.

{% indent %}
![spaceDumpReload.jpg](/attachment_files/sbp/spaceDumpReload.jpg)
{% endindent %}

The Space Dump utility uses a temporary persistent space approach with the [space copy API](http://www.gigaspaces.com/docs/JavaDoc{%latestxaprelease%}/com/j_spaces/core/admin/IRemoteJSpaceAdmin.html#spaceCopy). This allows the utility to consume all the data from every IMDG partition and push it into a file. To reload the data from the file, the temporary space is started, loading the data from the file, and then the data is copied back into the relevant IMDG partition. If the IMDG is running backup spaces, these are restarted to allow them to recover their data from their relevant primary instance.

# The Space Dump Utility
To run the Space Dump Utility:

1. Download the [Space Dump Utility](/attachment_files/sbp/spacedump.zip).
2. Run the utility - The Space Dump Utility accept the following arguments:

{% highlight java %}
java com.gigaspaces.util.spacedump.SpaceDumpMain <lookup locator> <Operation [dump | reload] <spaceName>
{% endhighlight %}

Example - Dump space data into a file:

{% highlight java %}
java com.gigaspaces.util.spacedump.SpaceDumpMain myhostName dump mySpace
{% endhighlight %}

Example - Reload space data from a file:

{% highlight java %}
java com.gigaspaces.util.spacedump.SpaceDumpMain myhostName reload mySpace
{% endhighlight %}

{% tip %}
Make sure the Space Dump utility has the `/gigaspaces-xap-root/lib/platform/jdbc/h2.jar` as part of its classpath.
{% endtip %}

