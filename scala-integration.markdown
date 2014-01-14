---
layout: post
title:  Scala Integration
categories: SBP
parent: data-access-patterns.html
weight: 1500
---



{% tip %}
**Summary:** {% excerpt %}This article presents common Scala integration scenarios on top of XAP.{% endexcerpt %}<br/>
**Author**: Shravan (Sean) Kumar, Solutions Architect, GigaSpaces<br/>
**Recently tested with GigaSpaces version**: XAP 8.0.3<br/>

{% toc minLevel=1|maxLevel=1|type=flat|separator=pipe %}

{% endtip %}

# Overview

Scala is a general purpose programming language designed to express common programming patterns in a concise, elegant, and type-safe way. It smoothly integrates features of object-oriented and functional languages, enabling Java and other programmers to be more productive.

Scala programs run on the Java VM, are byte code compatible with Java so you can make full use of existing Java libraries or existing application code. You can call Scala from Java and you can call Java from Scala, the integration is seamless. Because of this, Scala applications can use of GigaSpaces libraries and API like any other Java library or API.

# Scala Helloworld examples

There are many possible permutations for integrating Java and Scala applications, you may have mixture of Java and Scala code or pure Scala code that you want to run on top of GigaSpaces.

{%comment%}
[Hello World in Scala](http://www.openspaces.org/display/SCL/Hello+World+in+Scala) OpenSpaces project ports the GigaSpaces XAP standard helloworld example application for common Scala integration scenarios. Use this as a reference for how to integrate your Scala applications with GigaSpaces.
{%endcomment%}

When building your application in Scala, the configuration and packaging will still be like Java application.

Find below code and configuration for the Helloworld application written completely in Scala.

{% inittab Scala Helloworld Example %}

{% tabcontent Scala Data Model %}

{% highlight java %}
package org.openspaces.example.helloworld.common

import com.gigaspaces.annotation.pojo.SpaceRouting
import scala.reflect.BeanProperty
import java.lang.Integer

case class Message(

    @BeanProperty @SpaceRouting
    var id: Integer,

    @BeanProperty
    var info: String) {

  def this() = this(null, null)

}
{% endhighlight %}

{% endtabcontent %}

{% tabcontent Scala Polling Container Bean %}

{% highlight java %}
package org.openspaces.example.helloworld.processor

import org.openspaces.events.adapter.SpaceDataEvent
import org.openspaces.example.helloworld.common.Message

import java.util.logging.Logger

class Processor {

	val logger = Logger.getLogger(this.getClass().getName())
	logger.info("Processor instantiated, waiting for messages feed...")

	@SpaceDataEvent
	def processMessage(msg: Message): Message = {
		logger.info("Processor PROCESSING: " + msg)
		msg.info = msg.info + "World !!"
		msg
	}

}
{% endhighlight %}

{% endtabcontent %}

{% tabcontent Scala Feeder %}

{% highlight java %}
package org.openspaces.example.helloworld.feeder

import org.openspaces.core.GigaSpace
import org.openspaces.core.GigaSpaceConfigurer
import org.openspaces.core.space.UrlSpaceConfigurer
import org.openspaces.example.helloworld.common._

import com.j_spaces.core.IJSpace;

import java.util.logging.Logger;

object Feeder {

	val logger = Logger.getLogger(this.getClass().getName())
	val gigaSpace = {
		val space = new UrlSpaceConfigurer(System.getProperty("helloworld.space.url")).space()
		new GigaSpaceConfigurer(space).gigaSpace()
	}

	def main(args: Array[String]) {
		Feeder.feed(1000)
		Feeder.readResults()
	}

	def feed(numberOfMessages: Int) {
		(0 until numberOfMessages).foreach { counter =>
			gigaSpace.write(Message(counter, "Hello "))
		}

		logger.info("Feeder WROTE " + numberOfMessages + " messages")
	}

	def readResults() {
		var template = Message(null, "Hello World !!")
		logger.info("Here is one of them printed out: " + gigaSpace.read(template))

		Thread.sleep(3000)

		val numInSpace = gigaSpace.count(template)
		logger.info("There are " + numInSpace + " Message objects in the space now")

	}

}
{% endhighlight %}

{% endtabcontent %}

{% tabcontent Scala Helloworld pu.xml %}

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:os-core="http://www.openspaces.org/schema/core"
       xmlns:os-events="http://www.openspaces.org/schema/events"
       xmlns:os-remoting="http://www.openspaces.org/schema/remoting"
       xmlns:os-sla="http://www.openspaces.org/schema/sla"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.openspaces.org/schema/core http://www.openspaces.org/schema/{%latestxaprelease%}/core/openspaces-core.xsd
       http://www.openspaces.org/schema/events http://www.openspaces.org/schema/{%latestxaprelease%}/events/openspaces-events.xsd
       http://www.openspaces.org/schema/remoting http://www.openspaces.org/schema/{%latestxaprelease%}/remoting/openspaces-remoting.xsd
       http://www.openspaces.org/schema/sla http://www.openspaces.org/schema/{%latestxaprelease%}/sla/openspaces-sla.xsd">

    <os-core:space id="space" url="/./processorSpace" />
    <os-core:distributed-tx-manager id="transactionManager"/>
    <os-core:giga-space id="gigaSpace" space="space" tx-manager="transactionManager"/>
    <bean id="helloProcessor" class="org.openspaces.example.helloworld.processor.Processor"/>
    <os-events:polling-container id="helloProcessorPollingEventContainer" giga-space="gigaSpace">
    <os-events:tx-support tx-manager="transactionManager"/>
		<os-core:template>
			<bean class="org.openspaces.example.helloworld.common.Message">
				<property name="info" value="Hello "/>
			</bean>
		</os-core:template>
		<os-events:listener>
			<os-events:annotation-adapter>
				<os-events:delegate ref="helloProcessor"/>
			</os-events:annotation-adapter>
		</os-events:listener>
	</os-events:polling-container>

	<os-sla:sla
				cluster-schema="partitioned-sync2backup"
				number-of-instances="2"
				number-of-backups="1"
				max-instances-per-vm="1">
	</os-sla:sla>
</beans>
{% endhighlight %}

{% endtabcontent %}

{% endinittab %}

# Using Scala Interpreter

Scala also comes with an interpreter ([REPL](http://www.scala-lang.org/node/2097)) which can be handy for development and testing. It is an interactive "shell" for writing Scala expressions and programs.

Interestingly, REPL can also be embedded in your application which is discussed in detail by Josh Suereth  [here](http://suereth.blogspot.com/2009/04/embedding-scala-interpreter.html) and by Vassil Dichev [here](http://speaking-my-language.blogspot.com/2009/11/embedded-scala-interpreter.html). Running REPL in your application is a useful trick which is theoretically possible but the ramifications of doing this on top of GigaSpaces are unknown. It is not recommended to run Scala on top of GigaSpaces using this approach.

