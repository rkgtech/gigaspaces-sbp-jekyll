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

GigaSpaces XAP provides a powerful IMDG with advanced data access options. Many times a simple key/value interface required to access the IMDG. The most popular one is the Map API. GigaSpaces main data access API is the `GigaSpace` interface. A simple wrapper around it described here exposing Map API.


# Cache Interface Implementation

The implementation includes two classes. The `Data` class that modles the data within the space and the `CacheService` class that wraps the `GigaSpace` API using standard `Map` API (`put`,`get` , `remove`..). 

The `CacheService` support inserting data using a key/value and also via region/key/value. A region allows you to mark entries with a "tag" that group these for better management. You may also indicate if you would like to have the data cached also at the client side. In such a case , the client will also have a copy of the data. Once the client application constructs the `CacheService` all cached data will be pre-loaded into the client side.

## The Data Space Class

The Data Space Class holds the key , value and region data within a simple POJO proprties. The `@SpaceId` annotation indicats the key field specified as the Space class ID field. 

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

### Optimzing Value Object Serialization and Storage
The `Data` class can be modified to support better serialization for the value object. You may store the value object in binary format or compressed format.

To store the value object in a binary format the `getValue` should have the following:
{% highlight java %}
@SpaceStorageType(storageType=StorageType.BINARY)
public Object getValue() {
	return value;
}
{% endhighlight %}

To store the value object in a compressed format the `getValue` should have the following:
{% highlight java %}
@SpaceStorageType(storageType=StorageType.COMPRESSED)
public Object getValue() {
	return value;
}
{% endhighlight %}

## The CacheService
The `CacheService` leveraging the 'GigaSpace' interface implementing the standrad `put`,`get`,`remove`,etc. methods:
{% highlight java %}
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

# Further reading:

- [Modeling and Accessing Your Data]({%latestjavaurl%}/modeling-and-accessing-your-data.html)
- [Deploying and Interacting with the Space]({%latestjavaurl%}/deploying-and-interacting-with-the-space.html)

