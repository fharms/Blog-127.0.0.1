---
layout: post
title: JBoss Marshalling the hidden serialization framework
categories : [JBoss-Marshalling]
tags:
 - Java
 - Hibernate
 - JBoss Marshalling
 - Serialization
post_author: Flemming Harms
redirect_from: "/2015/06/29/JBoss-Marshalling-the-hidden-serialization-framework/"
---
A few years back, I had the challenge to find a better solution for sending data between our client and server, and I think it’s about time to blog about it :-) We have been using XStream to serialize our java model to XML and it is sent via web service. This has been working fine for a long time, but our data structure was increasing in size and we started to suffer for performance, increasing memory footprint and unnecessary complexity.

It was time to come up with a new solution.
<!--more-->
## <a name=”thegoal”>[./](#thegoal) The goal is to
*  Increase serialization speed and lower memory footprint.
*  Support for custom object translators.
*  Serializes arbitrary object graphs.
*  Cycle detection.

## <a name="whatdidifoundout"></a>[./](#whatdidifoundout) What did I found out
First of all, there are a lot of frameworks that can help you to serialize your java objects, and they all come with pro and cons. So, I started to do a little research on different frameworks based on the listed requirements.

I narrowed it down to three frameworks that seem interesting:

* Kryo seems promising and really fast. Unfortunately, we had to do a lot refactoring on inner classes to get it to work with Kryo.
* Protobuf is another really fast serialization framework. The biggest issue with Protobuf, in our case, was that you have to describe the data structured up front.
* JBoss Marshalling is a sub project under the JBoss Remoting project. This looked very promising, especially because it supported all of our requirements right out off the box.

After a few prototypes, we decided to go with the JBoss Marshalling. The rest of the post shows how easy it is to use JBoss Marshalling for serializing java objects.

## <a name="solution"></a>[./](#solution) The solution
I created a small example showing how you can serialize/deserialize an object graph to the disk. This could easily be changed to use with a REST service between a client and server.

To spice up the example, I’m adding Hibernate to show how you can use object resolvers to unproxy Hibernate objects in the serialization process.

### The first step is to set up the MarshallerFactory.
The MashallerFactory is the entry point and will create a MarshallerFactory with the specified protocol. The parameter “river” sets the protocol it’s using for serialization.

See the [MarshallingService](https://github.com/fharms/java-examples/blob/master/jboss-marshalling-example/src/main/java/com/fharms/marshalling/service/MarshallingService.java#L56)

```java
MarshallerFactory marshallerFactory =
      Marshalling.getProvidedMarshallerFactory("river");
```

At the moment, you can choose between the “river” and “serial” protocol, and there is not much information on the river protocol. However, this is what [@dmlloyd0](https://twitter.com/dmlloyd0) from the JBoss Marshalling project told me.

>The river protocol has a higher degree of compression. There are short forms for dozens of JDK classes and objects that require an extended form for standard serialization.

### Next step is the configuration
There are different configuration options you can set but not all are necessary by default, so you can play around with them to optimize performance.

``` java
 MarshallingConfiguration configuration =
      new MarshallingConfiguration();
 configuration.setVersion(3);
 configuration.setClassCount(10);
 configuration.setBufferSize(8096);
 configuration.setInstanceCount(100);
 configuration.setExceptionListener(new MarshallingException());
 configuration.setClassResolver(new SimpleClassResolver(getClass().getClassLoader()));
```

* The **"setClassCount"** will pre-size the internal map for class information. To avoid unnecessary resize later, find the number that is closest to the number of classes your model holds.
* The **"setBufferSize"** sets the size of the internal byte array for the input buffer. The bytes’ read is equal to the length of the buffer size. You can experiment with this parameter to increase performance. The default size is 512.
* The **"setInstanceCount"** will pre-size the internal map for instances. To avoid unnecessary resize later, find the number that is closest to the number of instances your object model holds.
* The **"setExceptionListener"** gives possibility to set your own exception lister. This is useful if you want to intercept special exceptions.
* The **"setClassResolver"** sets the classloader, which is use to resolve the classes. There are three options: “SimpleClassResolver”, “ContextClassResolver” and “ModularClassResolver”. The last option is for JBoss Module, and my guess is you can stick with the first two.

There are plenty more configuration options you can play with.

### Set up the object resolver and create marshaller.
The object resolver gives you the possibility to intercept objects when they are processed. There are two types of object resolvers: “pre resolver” and “resolver”.

```java
private Marshaller createMarshaller() throws IOException {
   configuration.setObjectPreResolver(new ChainingObjectResolver(Collections.singletonList(new HibernateDetachResolver())));
      return marshallerFactory.createMarshaller(configuration);
}

```
* **"setObjectPreResolver"** is invoked before user replacement and global object resolver. This is useful when handling Hibernate SerializableProxy objects to avoid the user replacement being executed before we can get the implementation class.
* **"setObjectResolver"** is invoked after user replacement and global object resolver. This is typically the object resolve you will use for most cases.

### The final step is to serialize and deserialize your objects.
The important part here is that you call the start before you serialize and deserialize and stop when you are finished.

```java
 public void serialize(Object object, OutputStream outputStream) throws IOException {
   Marshaller marshaller = createMarshaller();
   marshaller.start(Marshalling.createByteOutput(outputStream));
   try {
      marshaller.writeObject(object);
   } finally {
      marshaller.finish();
   }
}

public <T extends Object> T deserialize(InputStream inputStream, Class<T> type) throws Exception {
   Unmarshaller unmarshaller = createUnmarshaller();
   unmarshaller.start(Marshalling.createByteInput(inputStream));
   try {
      return type.cast(unmarshaller.readObject());
   } finally {
      unmarshaller.finish();
   }
}
```

## <a name="wheredoifindthecode"></a>[./](#wheredoifindthecode) Where do I find the code.

>Jump to the [source code](https://github.com/fharms/java-examples/tree/master/jboss-marshalling-example) on github

## <a name="whatdidilearn"></a>[./](#whatdidilearn) What did I learn?
The [JBoss Marshalling](http://jbossmarshalling.jboss.org/) might not be the fasted serialization framework, but it does follow the java serialization specification, which means it pretty much works right out of the box with arbitrary object graphs.

The object resolvers are really powerful because it gives you control over the serialization and deserialization process. We have build object resolvers that detach objects and only send the id of the object because they already known on the client.

Some of our concerns were that we haven’t seen many others using Marshalling beside internal JBoss projects and our self. It’s hard to find examples and documentation. However, there is help to get both from the IRC channel #jboss-marshalling and the API documentation [Marshalling API](http://jbossmarshalling.jboss.org/docs)

You can also learn a great deal from the Marshalling [source code](https://github.com/jboss-remoting/jboss-marshalling)

If you are looking for a fast serialization framework that requires little investment and low risk, JBoss Marshalling is a good choice and we still use it as serialization framework between our client and server.
