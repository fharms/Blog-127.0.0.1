---
layout: post
title: Take JPA to another level in Apache Camel
categories : [Apache Camel]
tags: 
 - Java
 - Apache Camel
 - JPA
 - Apache
 - Spring
 - Hibernate
 - Proxy
post_author: Flemming Harms
---

Camel comes with an impressive number of modules, and one of them is the ability to use JPA in Camel routes to 
consume/produce with JPA. 

There are many examples and guides showing how you can set up and work with basic Camel JPA. 
But I found little help using JPA in beans with camel routes. 

This post will give you a little background information on how @PersistenceContext and transaction work with 
Camel and introduce you to the Camel Entity Manager Post Processor.
<!--more-->

## <a name="thegoal">[./](#thegoal) In this post 

In this post, I’m explaining what the challenge is with using JPA directly in beans and how the 
Camel Entity Manager Post Processor works and finally show how easily you can inject an EntityManager 
working with the Camel JPA component.  

## <a name="whatstheproblem"></a>[./](#whatstheproblem) So what is the challenge?

The challenge using JPA from a bean is it does not know of the EntityManager the route is using or the transaction manager. 
The same goes the other way if you are injecting @PersistenceContext into a bean Camel route that does not know of it.

It’s simply two scopes from the EntityManager and the transaction point of view. Confused? Here is a short example

```java
public void configure() {
       from("jpa:com.github.Joke")
	     .transacted()
           .bean(MyJpaBean.class, “findSubscribers”); 
   }
```
MyJpaBean does not know about the Camel Entity Manager or the active transaction. Second the injection of @PersistenceContext behaves differently, depending on which scenario you choose.

>With simple Camel beans, you must inject the EntityManager and join the transaction or extract the Camel EntityManager from the Exchange.

or

>For spring beans annotated with @Transactional and injected EntityManager, it will automatically join the transaction.

Luckily, you don’t have to think about that if you are using the Camel Entity Manager Post Processor. 


## <a name="solution"></a>[./](#solution) The Camel Entity Manager Post processor

The Camel Entity Manager Post Processor wraps the current EntityManager proxy with a new proxy, with the only purpose to control when it should use the Camel EntityManager or join the current transaction.

The post processor [scans](https://github.com/fharms/camel-jpa-entitymanager/blob/blog/src/main/java/com/github/fharms/camel/entitymanager/CamelEntityManagerHandler.java#L144) for beans with the @PersisteneContext annotation and create a proxy. Second, it creates a [MethodInterceptor](https://github.com/fharms/camel-jpa-entitymanager/blob/blog/src/main/java/com/github/fharms/camel/entitymanager/CamelEntityManagerHandler.java#L106) for intercepting method calls in the bean.

It might be obvious why it creates a proxy around the EntityManager, but the MethodInterceptor might not, 
but we want to intercept method calls with the parameter of type exchange; this makes it possible to retrieve the 
Camel EntityManager from the [exchange header](https://github.com/fharms/camel-jpa-entitymanager/blob/blog/src/main/java/com/github/fharms/camel/entitymanager/CamelEntityManagerHandler.java#L123), using this instead.


You can always retrieve Entity Manager manual as shown below

```java
exchange.getIn().getHeader(CAMEL_ENTITY_MANAGER, EntityManager.class)
```

The trick here is we need to save the EntityManager for later use in a thread local when we actually call the entity manager.
The EntityManager is removed again when the transaction is completed, regardless of whether it’s committed or rollback. See [SessionCloseSynchronizationManager](https://github.com/fharms/camel-jpa-entitymanager/blob/master/src/main/java/com/github/fharms/camel/entitymanager/CamelEntityManagerHandler.java#L180)

The diagram show the behavior of the bean proxy 
[<img src="/images/Apache-Camel/bean-state-diagram-small.png?style=centerme">](/images/Apache-Camel/bean-state-diagram.png)

The CamelEntityManagerHandler proxy part is straight forward. It wraps the existing EntityManager proxy with its own proxy and controls the flow and state.

The diagram show the behavior of the entity manager proxy 
[<img src="/images/Apache-Camel/proxy-state-diagram-small.png?style=centerme">](/images/Apache-Camel/proxy-state-diagram.png)

## <a name="solution"></a>[./](#solution) Using the Camel Entity Manager post processor with Camel

Using Camel Entity Manager post processor is easy and only requires adding the dependency below.

```xml
<dependency>
     <artifactId>camel.jpa.entitymanager</artifactId>
     <groupId>com.github.fharms</groupId>
     <version>0.0.2</version>
</dependency>
```

Then is just a matter of injecting the @PersistenceContext as you normally would do when working with JPA. 

```java
 @javax.persistence.PersistenceContext
 EntityManager em;
```

Ignore the Entity Manager created by camel and use the injected instead. Easily add the @IgnoreCamelEntityManager.

```java

 @IgnoreCamelEntityManager
 @javax.persistence.PersistenceContext
 EntityManager em;


 @IgnoreCamelEntityManager
 public void findMyEntity(Exchange exchange) {
    em.find(MyEntity.class, exchange.getIn().getBody(Integer.class))
 }
```

For a more detailed example, follow the link [Camel Entity Manager Example](https://github.com/fharms/java-examples/tree/camel_jpa_entitymanager_example/camel-entitymanager-example).

## <a name="References"></a>[./](#References) References

[Apache Camel](http://camel.apache.org/)


[Apache Camel JPA](http://camel.apache.org/jpa.html)


[Hibernate ORG](http://hibernate.org/)


[Java Examples](https://github.com/fharms/java-examples)

