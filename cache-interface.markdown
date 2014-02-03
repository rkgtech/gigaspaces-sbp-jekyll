---
layout: post
title:  Cache Interface
categories: SBP
parent: data-access-patterns.html
weight: 100
---
{% summary page %}This best practice explains how to implement a cache interface using the Map API.{% endsummary %}
 {% tip %}
 **Author**:  Shay Hassidim<br/>
 **Recently tested with GigaSpaces version**: XAP 9.6<br/>
 **Last Update:** Feb 2014<br/>

{% toc minLevel=1|maxLevel=1|type=flat|separator=pipe %}
{% endtip %}

{% compositionsetup %}


# Overview

GigaSpaces XAP .....

# Implementation

The implementation includes two classes. The 'Data' class that modles the data within the space and the 'CacheService' class that wrpas the GigaSpace API using standard Map API ('put','get' , 'remove'..). 

{% highlight java %}
import com.gigaspaces.annotation.pojo.SpaceId;
import com.gigaspaces.annotation.pojo.SpaceIndex;

public class Data {
	public Data(){}

	String key;
	Object value;
	String region;

	@SpaceIndex
	public String getRegion() {
		return region;
	}
	public void setRegion(String region) {
		this.region= region;
	}
	@SpaceId(autoGenerate=false)
	public String getKey() {
		return key;
	}
	public void setKey(String key) {
		this.key = key;
	}
	public Object getValue() {
		return value;
	}
	public void setValue(Object value) {
		this.value = value;
	}
}

{% endhighlight %}


{% highlight java %}
package org.openspaces.cache;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Map;
import java.util.Set;

import org.openspaces.core.GigaSpace;
import org.openspaces.core.GigaSpaceConfigurer;
import org.openspaces.core.space.UrlSpaceConfigurer;
import org.openspaces.core.space.cache.LocalViewSpaceConfigurer;

import com.gigaspaces.query.IdQuery;
import com.j_spaces.core.client.SQLQuery;

public class CacheService 
{
	private GigaSpace spaceView;
	private GigaSpace space;
	boolean localView;

	public CacheService(String url) throws Exception {
		init(url,false);
	}

	public CacheService(String url , boolean clientCache) throws Exception {
		init(url, clientCache);
	}
	
	public void init(String url , boolean clientCache) throws Exception {
		this.localView = clientCache;
		UrlSpaceConfigurer urlConfigurer = new UrlSpaceConfigurer(url);
		space= new GigaSpaceConfigurer(urlConfigurer).gigaSpace();
		if (localView)
		{
			LocalViewSpaceConfigurer localViewConfigurer = new LocalViewSpaceConfigurer(urlConfigurer)
				.addViewQuery(new SQLQuery<Data>(Data.class, ""));
			// Create local view:
			spaceView = new GigaSpaceConfigurer(localViewConfigurer).gigaSpace();
		}
		else
		{
			spaceView  = space;
		}
		
	}    	
	
    public void put(String region, String key, Object value) throws Exception 
    {
		Data d = new Data();
		d.setKey(key);
		d.setValue(value);
		d.setRegion(region);
		space.write(d);
    }

    public Object get(String region , String key) throws Exception 
    {
    	Data templ = new Data();
    	templ.setKey(key);
    	templ.setRegion(region);
		Data d = spaceView.read(templ);
		if (d!=null)
			return d.getValue();
		else 
			return null;
    }

	public void clear() {
		space.clear(new Data());
	}

	
	
    public Object get(String key) throws Exception 
    {
		Data d = spaceView.readById(Data.class, key);
		if (d!=null)
			return d.getValue();
		else 
			return null;
    }

    public void remove(String region, String key) throws Exception 
    {
    	Data templ = new Data();
    	templ.setKey(key);
    	templ.setRegion(region);
    	space.clear(templ);
    }

    
    public void remove(String key) throws Exception 
    {
    	IdQuery<Data> idquery = new IdQuery<Data>(Data.class, key);
    	space.clear(idquery);
    }

    
	public boolean containsKey(String key) {
    	IdQuery<Data> idquery = new IdQuery<Data>(Data.class, key);
		int count = spaceView.count(idquery);
    	return (count>0);
	}

	public int size() throws Exception 
    {
		return spaceView.count(new Data());
    }

	public int size(String region) throws Exception 
    {
    	Data templ = new Data();
    	templ.setRegion(region);
		return spaceView.count(templ);
    }

	public void putAll(String region , Map<String,Object> map) {
		Data d[] = new Data[map.size()];
		int count=0;
		Iterator<String>  keys = map.keySet().iterator();
		while (keys.hasNext())
		{
			String key = keys.next();
			Object val = map.get(key);
			d[count] = new Data();
			d[count].setKey(key);
			d[count].setValue(val);
			d[count].setRegion(region);
			count++;
		}
		space.writeMultiple(d);
	}
	
	public void putAll(Map<String,Object> map) {
		putAll(null , map);
	}
	
	public Set<String> keySet() throws Exception {
		SQLQuery<Data> query = null;
		if (localView)
			query = new SQLQuery<Data>(Data.class, "");
		else
    	 query = new SQLQuery<Data>(Data.class, "").setProjections("key");

    	Data d[] = spaceView.readMultiple(query);
    	
    	Set<String> keys = new HashSet<String>();
		
		for (int i = 0; i < d.length; i++) {
			keys.add(d[i].getKey());
		}
		return keys;
	}
}

{% endhighlight %}

{% tip %}
bla bla bla bla 
{% endtip %}

## Further reading:

- [Modeling and Accessing Your Data]({%latestjavaurl%}/modeling-and-accessing-your-data.html)
- [Deploying and Interacting with the Space]({%latestjavaurl%}/deploying-and-interacting-with-the-space.html)

