---
layout: post
title: Hibernate OGM with JPA and Infinispan on Wildfly 8.2
tags: Java, Hibernate OGM, JPA, Infinispan, Wildfly
---

Recently, we had a task where we had to choose whether we should use the current architecture.

> “Entity->JPA->Hibernate ORM->Cache->SQL”

Alternatively, whether to use Hibernate OGM as a persistence layer to a NoSQL provider.

>“Entity->JPA->Hibernate OGM->NoSQL”
<!--more-->
There were many strong arguments for using Hibernate OGM, but as always when you encounter new technology, you are trying to determine the work and impact based on documentation, forums, blogs or short tutorials typically based on the latest version. However, another story is getting it to work with a running application, which is a few versions behind the latest and greatest.
 
So I decided to create a POC with our current technology stack the almighty [Wildfly 8.2](http://wildfly.org/news/2014/11/20/WildFly82-Final-Released/), Hibernate ORM, Infinispan and now Hibernate OGM 4.1.3.Final.
 
## <a name=”thegoal”>[./](#thegoal) The goal is
*  Install Hibernate OGM 4.1.3 into Wildfly 8.2
*  Deploy an application using Hibernate OGM
 
## <a name="whatdidifoundout"></a>[./](#whatdidifoundout) What did I found out
 
I started with the [Hibernate OGM documentation](http://docs.jboss.org/hibernate/ogm/4.1/reference/en-US/html/ogm-configuration.html#ogm-configuration-jbossmodule) and the first question I encountered, was it even possible to get it to work with Infinispan 6.0.2?
 
<center><blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/SanneGrinovero">@SanneGrinovero</a> Is it possible to get the hibernate-ogm 4.1.3 to work with Infinispan 6.0.2 in Wildfly 8.2? It seems bound to Infinispan 7</p>&mdash; Flemming Harms (@fnharms) <a href="https://twitter.com/fnharms/status/589742460915609600">April 19, 2015</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script></center>
 
Fortunately, this is really easy to do. When you install the modules, it will co-exist with the current version of Infinispan 6, but the downside is you will have to maintain two sets of configuration files if you want to customize Infinispan 7. The good news is that the module will work out of the box with the default configuration.
 
## <a name="solution"></a>[./](#solution) The solution
 
I have created a small solution where I create a simple remote queue service, where you are able to subscribe, unsubscribe and post/consume messages.
 
In the package :
>[org.hibernate.ogm.infinispan7.jpa.example.model](https://github.com/fharms/java-examples/tree/master/hibernate-ogm-infinispan7-jpa-example/src/main/java/com/fharms/ogm/infinispan7/jpa/example/model) you will find the JPA entities.
 
>[org.hibernate.ogm.infinispan7.jpa.example.dao](https://github.com/fharms/java-examples/tree/master/hibernate-ogm-infinispan7-jpa-example/src/main/java/com/fharms/ogm/infinispan7/jpa/example/dao) you will find the service that is responsible for persisting the entities.
 
I have created an [arquillian test](https://github.com/fharms/java-examples/blob/master/hibernate-ogm-infinispan7-jpa-example/src/test/java/com/fharms/ogm/infinispan7/jpa/example/dao/RemoteEventDaoIT.java) to show it actually work.
 
The main pom file takes care of all the setup and installation of Hibernate OGM. I use the maven-dependency-plugin for this, but if you want to use a DIY method, follow the [manual steps](#manually_install_hibernate)
 
```xml
<plugin>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
       <execution>
             <id>unpack</id>
             <phase>pre-integration-test</phase>
             <goals>
                   <goal>unpack</goal>
             </goals>
             <configuration>
                   <artifactItems>
                         <artifactItem>
                               <groupId>org.wildfly</groupId>
                               <artifactId>wildfly-dist</artifactId>
                               <version>${version.wildfly}</version>
                               <type>zip</type>
                               <overWrite>false</overWrite>
                               <outputDirectory>${project.build.directory}</outputDirectory>
                         </artifactItem>
                         <artifactItem>
                               <groupId>org.hibernate.ogm</groupId>
                               <artifactId>hibernate-ogm-modules-wildfly8</artifactId>
                               <type>zip</type>
                               <version>${version.hibernate-ogm}</version>
                               <overWrite>false</overWrite>
                               <outputDirectory>${jboss.home}/modules</outputDirectory>
                         </artifactItem>
                         <artifactItem>
                               <groupId>org.hibernate</groupId>
                               <artifactId>hibernate-search-modules</artifactId>
                               <classifier>wildfly-8-dist</classifier>
                               <version>${version.hibernate-search}</version>
                               <type>zip</type>
                               <overWrite>false</overWrite>
                               <outputDirectory>${jboss.home}/modules</outputDirectory>
                         </artifactItem>
                         <artifactItem>
                               <groupId>org.infinispan</groupId>
                               <artifactId>infinispan-as-embedded-modules</artifactId>
                               <version>${version.infinispan}</version>
                               <type>zip</type>
                               <overWrite>false</overWrite>
                               <outputDirectory>${jboss.home}/modules</outputDirectory>
                         </artifactItem>
                   </artifactItems>
             </configuration>
       </execution>
    </executions>
</plugin>
```
 
There are a few issues with the persistence.xml that you have to be aware of.
 
* The provider “HibernateOgmPersistence” bypasses the autodetection of entities, and this is the reason why they are specified in the persistence unit. This is a known issue. [OGM-818](https://hibernate.atlassian.net/browse/OGM-818)
* Adding the property "hibernate.search.default.directory_provider=infinispan" & "hibernate.search.default.exclusive_index_use=false" gave a considerable performance improvement.
* Want to try how easy and transparent it is to switch between Hibernate ORM and OGM. Change the persistence unit name in the class com.fharms.ogm.infinispan7.jpa.example.dao.RemoteEventDao to “RemoteEventQueueSQL”
 
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<persistence xmlns="http://java.sun.com/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" version="2.0" xsi:schemaLocation="http://java.sun.com/xml/ns/persistence http://java.sun.com/xml/ns/persistence/persistence_2_0.xsd">
  <persistence-unit name="RemoteEventQueue" transaction-type="JTA">
       <provider>org.hibernate.ogm.jpa.HibernateOgmPersistence</provider>
       <!-- When using HibernateOgmPersistence provider it does not scan for entity classes -->
      <class>com.fharms.ogm.infinispan7.jpa.example.model.RemoteEvent</class>
      <class>com.fharms.ogm.infinispan7.jpa.example.model.Subscriber</class>
      <class>com.fharms.ogm.infinispan7.jpa.example.model.EventVO</class>
      <properties>
             <!-- Changing from filebase to Infinispan gave huge performance from 3 sec to 177 ms -->
             <property name="hibernate.search.default.directory_provider" value="infinispan"/>
             <property name="hibernate.search.default.exclusive_index_use" value="false"/>
            <property name="hibernate.transaction.jta.platform"
                        value="org.hibernate.service.jta.platform.internal.JBossAppServerJtaPlatform" />
            <property name="hibernate.ogm.datastore.provider"
                        value="infinispan" />
            <property name="hibernate.ogm.infinispan.configuration_resource_name" value="com/fharms/ogm/infinispan7/jpa/example/dao/infinispan-local.xml"/>
            </properties>
      </persistence-unit>
    
      <!-- If you want to run with Hibernate ORM and SQL database, change the persistence unitname
        in the class com.fharms.ogm.infinispan7.jpa.example.dao.RemoteEventDao 
      -->
      <persistence-unit name="RemoteEventQueueSQL" transaction-type="JTA">
       <jta-data-source>java:jboss/datasources/ExampleDS</jta-data-source>
      <properties>
            <property name="hibernate.hbm2ddl.auto" value="create" />
             <!-- Changing from filebase to Infinispan gave huge performance from 3 sec to 177 ms -->
             <property name="hibernate.search.default.directory_provider" value="infinispan"/>
             <property name="hibernate.search.default.exclusive_index_use" value="false"/>
      </properties>
      </persistence-unit>
</persistence>
```
 
## <a name="wheredoifindthecode"></a>[./](#wheredoifindthecode) Where do I find the code.
 
>Jump to the [source code](https://github.com/fharms/java-examples/tree/master/hibernate-ogm-infinispan7-jpa-example) on github
 
## <a name="whatdidilearn"></a>[./](#whatdidilearn) What did I learn?
 
It was straightforward to use for this use case, but with all software, there are always issues. So don’t expect anything less with Hibernate OGM.
 
There are some issues you have to be aware of when installing for the first time, as described in this post, but the Hibernate OGM documentation is a good place to start. The documentation provided walks you through the installation and setup.
 
Then [source code](https://github.com/hibernate/hibernate-ogm) is also a good place to look and I learned a great deal by looking into the unit tests.
 
So did we choose Hibernate OGM? Well, not this time because we couldn't justify taking the risk with introducing a new framework at this point. However, this is not the last time I’ll be working with Hibernate OGM; that’s for sure.
 
:)
 
## <a name="manually_install_hibernate"></a>[./](#manually_install_hibernate) Manual install the Hibernate OGM & Infinispan
This small guide shows how you can manually update Wildfly with Hibernate OGM & Infinispan module.
 
* Follow the guide [Packaging Hibernate OGM applications for WildFly 8.2](https://docs.jboss.org/hibernate/ogm/4.1/reference/en-US/html/ogm-configuration.html#_packaging_hibernate_ogm_applications_for_wildfly_8_2 "Packaging Hibernate OGM applications for WildFly 8.2")
* Next you need to download the [Infinispan 7 Wildfly module](http://downloads.jboss.org/infinispan/7.1.1.Final/infinispan-as-embedded-modules-7.1.1.Final.zip "Infinispan 7 Wildfly module") and unzip in the "module/" directory
* A finally download the [Hibernate Search 5.1 Wildfly module](https://repository.jboss.org/nexus/service/local/repositories/releases/content/org/hibernate/hibernate-search-modules/5.1.0.Final/hibernate-search-modules-5.1.0.Final-wildfly-8-dist.zip "Hibernate Search Wildfly module") and unzip in the "module/" directory
 
Your Wildfly is now updated and ready for Hibernate OGM.