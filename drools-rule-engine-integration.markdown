---
layout: post
title:  Drools Rule Engine Integration
categories: SBP
parent: processing.html
weight: 100
---

{% compositionsetup %}

{% tip %}
**Summary:** {% excerpt %}This article illustrates how to integrate the Drools Rule Engine with GigaSpaces XAP{% endexcerpt %}<br/>
**Author:** Jeroen Remmerswaal<br/>
**Recently tested with GigaSpaces version**: XAP 7.1.1<br/>
**Last Update:** December 2010<br/>
**Contents:**<br/>

{% toc minLevel=1|maxLevel=2|type=flat|separator=pipe %}

{% endtip %}

# Overview

A rule engine is a component specialized in the execution of business rules. There are dedicated algorithms for such engines, such as the [RETE algorithm](http://en.wikipedia.org/wiki/Rete_algorithm), which is optimized for efficiency, concurrency and speed. As such a rule engine is capable of processing large volumes of data, and therefore, an integration with GigaSpaces would combine the best of two worlds.

![droolsRuleEngineIntegration.jpg](/attachment_files/sbp/droolsRuleEngineIntegration.jpg)

This article demonstrates how to integrate a rules engine called Drools (also known as JBoss Rules) with GigaSpaces. A working example is provided as part of this article. In the next sections a technical description is given on how the integration works. It will go into detail of all the classes used, how they interact and should be accessed from another system. You will find that the GigaSpaces data grid is used for storing the business rules, event containers for pre-compiling rulesets in an asynchronous manner, and that Remoting Services can be used for invocations of the rule-engine.

Click [here](/download_files/sbp/SpaceEnabledDrools.zip) to download a working code sample that uses [JBoss Drools 5](http://www.jboss.org/drools).

# Modules

The example project is based on a Maven project structure and has four submodules:

- RulesClient - Contains only a single class that demonstrates how a client would access the rules engine. See the 'Executing rules' section for details.
- RulesLoader - Contains code for a Processing Unit that (at startup) loads business rules into the space.
- RulesShared - A library with code that is shared between the other modules.
- RulesSpace - Contains code for a Processing Unit that has an embedded space containing the business rules data and logic to load, unload, compile and execute them.

{% inittab Modules %}

{% tabcontent Loading Rules %}
Before being able to execute any business rule, the system first needs to load a set of rules into the space. This means reading rules files from disk, converting them into Java objects and then writing them to a space. Inside this space (called rules-space) is a polling container that will pick up and precompile the new rules so they are ready to be used.

The main loader class is called GigaSpacesRulesLoader. When started, it reads its configuration from a Spring Framework context file. Next is an example snippet of such a configuration:

{% highlight xml %}
<bean id="ruleSets" class="java.util.HashMap">
   <constructor-arg>
      <map>
         <entry key="Test" value="META-INF/rules/testruleset.xml"/>
      </map>
   </constructor-arg>
</bean>
{% endhighlight %}

This piece of configuration tells the rules loader to load an XML ruleset file called testruleset.xml and place all rules defined in that set under ruleset name 'Test'.

Here is a testruleset.xml file example:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<change-set xmlns='http://drools.org/drools-5.0/change-set'
               xmlns:xs='http://www.w3.org/2001/XMLSchema-instance'
               xs:schemaLocation='http://drools.org/drools-5.0 change-set.xsd'>
   <add>
      <resource source='classpath:META-INF/rules/test/testrule.drl' type='DRL'/>
   </add>
</change-set>
{% endhighlight %}

As you can see, the file contains resources. Each resource points to an actual Drools rule file (DRL/DSLR), or a Drools DSL file. Next is an example of such a Drools rule file:

{% highlight java %}
import com.tricode.gigaspaces.rules.shared.fact.*;
import org.apache.log4j.*;

rule "validate holiday"
when
   $h1 : Holiday( `when` == "july" )
then
   Logger.getLogger("rulesspace").info($h1.getName() + ":" + $h1.getWhen());
end
{% endhighlight %}

Each file can contain multiple rules. When the rules loader reads these rules files, it converts their contents to a Java byte array (this is a small optimization because the Drools KnowledgeBuilder does not accept Strings).

The rules are converted into the following data model structure:
![componentmodel.jpg](/attachment_files/sbp/componentmodel.jpg)

The DroolsRuleset is the main entity. It contains a project name (to keep rulesets separated between projects), a ruleset name, a precompiled version of the ruleset (called a KnowledgePackage and is empty by default), a status, a collection of DSL definitions and a collection of rules. By default the status will be 'unprocessed', but this is changed when the rule is compiled by the RulesCompiler in the space.

Each one of the DroolsDslDefinition entities resembles a DSL entry from the ruleset XML file that was read by the rules loader. It contains a ruleSetId (that points to the associated ruleset), a project name (an optimization for routing of the data in the space), and the DSL content as a byte array.

Each one of the DroolsRule entities resembles a DRL entry from the ruleset XML file that was read by the rules loader. It contains a ruleSetId (that points to the associated ruleset), a project name (an optimization for routing of the data in the space), the DRL/DSLR content as a byte array and a field to determine the content type (i.e. DRL or DSLR).

The rules loader uses a GigaSpaces remoting service to load the rules into the space. This service is based on an interface called IRulesLoadingService:
![irulesloadingservice.jpg](/attachment_files/sbp/irulesloadingservice.jpg)

The interface has the following contract:

{% highlight java %}
public interface IRulesLoadingService
    void loadRuleset(@Routing("getProjectName") DroolsRuleset ruleSet);
    void unloadRuleset(@Routing("getProjectName") DroolsRuleset ruleSet);
    void unloadAllRulesets();
}
{% endhighlight %}

And on the server side the implementation is provided that will write new rule-objects into the rule space, which is called RulesLoadingServiceImpl.
{% endtabcontent %}

{% tabcontent Compiling Rules %}
Once the rule loader has written the new rule objects into the rule space, a polling container (called RulesCompiler) will pick them up. It is triggered on the creation of a new RuleSet object and automatically reads the associated DroolsRule and DroolsDslDefinition objects along with it. It then creates a new Drools KnowledgeBuilder which is capable of validating all the rules.

![stdflow.jpg](/attachment_files/sbp/stdflow.jpg)

The result of the validation is a state change of the RuleSet object, see next image for the state diagram:
![stdflow2.jpg](/attachment_files/sbp/stdflow2.jpg)

The initial state is 'unprocessed'. When there is a validation error, the state changes to 'exception'. When there is no error, the state is changed to 'compiled'.

The RuleSet object's state is updated accordingly and the object is updated in the space. When the state was set to 'compiled', the RuleSet object will now also include the compiled versions of all rules in the form of Drools KnowledgePackages.

{% endtabcontent %}

{% tabcontent Executing Rules %}
Rules execution can be done in a number of ways. The typical method is done by using Remoting Services. GigaSpaces Remoting is highly available, reliable and scalable by using automatic failover and allowing for grid computing patterns like Master/Worker and Map/Reduce.

For a remoting service the following need to be established:

- A contract, in our case an interface called IRulesExecutionService.

{% highlight java %}
interface IRulesExecutionService {
    Collection<IFact> executeRules(@Routing String projectName, String[] rulesetNames, Map<String, Object> globals, Collection<IFact> facts);
    Collection<IFact> executeRules(@Routing String projectName, String[] rulesetNames, Collection<IFact> facts);
}
{% endhighlight %}

- A client using the stub to the interface. This value is automatically injected by use of a proxy, which exists in a few flavors in GigaSpaces, for example ExecutorProxy.

{% highlight java %}
@ExecutorProxy
private IRulesExecutionService rulesExecutionService;
{% endhighlight %}

When a client invokes it, the name of the project, the name(s) of the ruleset(s) to execute, an optional map of globals and finally the facts that will be used in the rules will be specified.

- A server side implementation, in our case an implementation of IRulesExecutionService interface, called RulesExecutionServiceImpl.

Upon invocation of an execute method, this class will try to load a DroolsContext object from the space for the requested configuration. This object is a cached version of a rules execution call and is used to speed up subsequent calls for the same ruleset(s). If the cached object is not found, a new one is created and the rules excecution service loads all the requested rulesets from the space to add them to this cached object.

Finally, the cached object is used to create a new Drools stateless knowledge session. If needed, the passed globals are added to the session and then there is a call to the main execute-method with the given facts as argument.

![stdflow3.jpg](/attachment_files/sbp/stdflow3.jpg)

The result of the call is a (possibly updated) set of fact objects.

{% endtabcontent %}

{% endinittab %}

# Use Cases

Below is a short explanation on a number of typical use-cases, where the combination between GigaSpaces and a rules engine such as Drools form a great symbiosis:

{% inittab Use Cases %}

{% tabcontent Data validation %}

One of the main applications of a rules engine is to use it for validation of data and service invocations. GigaSpaces supports a good integration point that is sensible and easy to use :

Any operation of any object in the memory data grid can be intercepted using a Space Filter, and can thus be validated using the rules engine. This means the rules can be made context-aware, so that they can see under which operation the data had been changed.

Examples:

1. I want to validate field X on object Y when it is 'written'
2. Value X for object Y is required when it is 'updated'
3. When validation rule X is not valid, I want the transaction to roll back and throw an exception.

A filter to achieve this are simple to write and involves the Space-Filter framework. An example can be found below:

{% highlight java %}
@Component
public class RulesFilter {
    private final Logger log = Logger.getLogger(RulesFilter.class);

    @GigaSpaceLateContext(name="gigaSpace")
    private GigaSpace gigaSpace ;

    private IRulesExecutionService rulesExecutor;

    @BeforeWrite
    public void executeRulesBeforeWrite(final BaseEntity entry) throws Exception {
	executeRules(entry, DroolsOperationContext.WRITE) ;
    }

    @BeforeUpdate
    public void executeRulesBeforeUpdate(final BaseEntity entry) throws Exception  {
    	executeRules(entry, DroolsOperationContext.UPDATE) ;
    }

    private void executeRules(BaseEntity entry, DroolsOperationContext operationContext) throws Exception {
	try {
            synchronized (this) {
	    	if ( rulesExecutor == null ) {
	    	rulesExecutor = new ExecutorRemotingProxyConfigurer<IRulesExecutionService>(gigaSpace,
                  RulesExecutionServiceImpl.class).timeout(15000).proxy();
	        }
	    }

            log.info(String.format("Executing rules on '%s' for filter-operation '%s'",
               entry.getClass().getSimpleName(), operationContext)) ;

            // Create working memory
	    List<Serializable> facts = new ArrayList<Serializable>() ;

            // Fill working memory with facts
	    facts.add(entry);
	    facts.add(operationContext);

	    // Execute the rules engine as a remoting service
	    rulesExecutor.executeRules(
		"<your_project>",
		entry.getClass().getSimpleName(),
		facts
	    );
	} catch (Exception e) {
	    e.printStackTrace();
	    throw e;
	}
    }
}

public enum DroolsOperationContext {
	WRITE, UPDATE, TAKE;
}
{% endhighlight %}

Notes:

- In the above example the DroolsOperationContext values are passed as "context-values" to the rules engine. A single set of rules can therefore be customized for the operation under which the invocation occurred.

{% endtabcontent %}

{% tabcontent Service validation %}
GigaSpaces allows seamless integration with AOP libraries like AspectJ. Service invocations to a GigaSpaces Remoting Service can therefore be automatically wrapped in calls to the RuleExecutionService.

An example of such a generic aspect could look like the following:

{% highlight java %}
@Aspect
@Component
public class DroolsAspect {
   @ExecutorProxy(gigaSpace="gigaSpace")
   private IRulesExecutionService rulesExecutor;

   @Around(value = "(within(<your_package>.GigaSpacesRemotingService) || " +
			"within(<your_package>.RulesExecutionServiceImpl))"
			+ "&& @annotation(Droolable)")
   public Object profile(ProceedingJoinPoint pjp) throws Throwable {
      Logger logger = Logger.getLogger(pjp.getTarget().getClass());
      long timestamp = 0;
      if (logger.isDebugEnabled()) {
         timestamp = System.currentTimeMillis();
         logger.debug(String.format("Drooling method: '%s'", pjp.getSignature().getName()));
      }

      for ( int t = 0 ; t < pjp.getArgs().length ; t ++ ) {
         if ( pjp.getArgs()[t] instanceof BaseEntity ) {
            List<BaseEntity> facts = new ArrayList<BaseEntity>() ;
            facts.add((BaseEntity)pjp.getArgs()[t]);

            rulesExecutor.executeRules(
               "<your_project>",
               pjp.getArgs()[t].getClass().getSimpleName(),
               facts
            );
         }
      }

      Object o = pjp.proceed();

      if (logger.isDebugEnabled()) {
         logger.debug(String.format("Drooling method end: '%s' in %s milliseconds ",
                 pjp.getSignature().getName(), ( System.currentTimeMillis() - timestamp) ));
      }
      return o ;
   }
}
{% endhighlight %}

In the above Around-Advice it can be seen that the rules-engine will only be invoked for service-methods that have the @Droolable annotation, like this:

{% highlight java %}
@RemotingService
@Loggable
public class RulesTestService implements {
   private static final Logger logger = Logger.getLogger(RulesTestService.class);

   @Droolable
   public void testRules(Object object) {
      logger.info(String.format("Testing rules for class '%s'", object.getClass().getSimpleName()));
   }
}
{% endhighlight %}

The Droolable annotation itself is defined as follows:

{% highlight java %}
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Droolable {

}
{% endhighlight %}

{% endtabcontent %}

{% endinittab %}

# Final Notes

## Integration with other engines

Integration with other rule-engines, such as Yasu (now SAP BRMS), or Jess can be done in a similar way: many rule engines have a low memory footprint and allow themselves to be embedded in a Java Virtual Machine quite easily.

Some rule-engines like Yasu also have the ability to access a standardized persistency layer, which can be replaced to make use of the GigaSpaces datagrid. This will increase the speed of loading large sets of rules.

## Parallel execution

A RETE network which is created out of a set of rules itself is not designed for parallel execution over multiple machines. It can at best make use of multi-core processors on a single machine. If rulesets need to be defined that can be executed in a parallel way the following considerations are useful:

- Decrease and partition the data-set that the rule-engine needs to use for execution, and use Map/Reduce style operations using task-execution or Executor Driven Remoting for parallel execution.
- Break up several rule-engine executions into separate executions and use Event Containers for chaining the various calls. This will make more efficient use of your resources.
