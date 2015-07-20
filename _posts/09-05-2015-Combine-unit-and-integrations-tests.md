---
layout: post
title: Combine unit and integrations tests in the same module
categories : [Unit-Test]
tags:
- Java
- Maven
- JUnit
- Unit test
- Integration test
post_author: Flemming Harms
redirect_from: "/2015/05/09/Combine-unit-and-integrations-tests/"
---

If you do a little research on how to combine unit and integration tests you will find
out there is at least two different ways of doing this, and both involves special Maven and IDE configuration.

## <a name="thegoal"></a>[./](#thegoal) The goal is

*  Unit and Integration test should exist in the same source folder
*  Executing unit tests from the IDE should only run unit tests
*  Executing “mvn install” should only execute unit tests, unless other is specified “mvn install -P integration-test”

## <a name="whatdidifoundout"></a>[./](#whatdidifoundout) What I found out

The prefer way is to setup a new source directory for the module/project containing the integration tests and use the [build-helper-maven-plugin](http://mojo.codehaus.org/build-helper-maven-plugin/ "http://mojo.codehaus.org/build-helper-maven-plugin/") to add an extra build source directory to maven.

Another solution is to place the integration tests in it's own module or multiple modules. The point here is the integration tests are separated from the unit test because they exist in another module location.

Both solutions comes with pro and cons, and in the end it all comes down to which solution is the most suitable for your project. But I think there is one big disadvantage with both solutions, it requires extra plug-ins and a number of configurations and special settings to make it work.

## <a name="solution"></a>[./](#solution) The solution

I decided to come up with another solution which required minimum configurations and was easy to use and understand. The advantage of this solution is I use the feature in surefire and failsafe plugin to separate the unit and integration tests for Maven, and a JUnit runner for Eclipse.

To control when the unit tests or integrations tests should run I use Maven profiles. The `<skipITs>true</skipITs>` skip the integration tests and the `<skipTests>true</skipTests>` skip the unit tests. This is supported out of the box by surefire and failsafe.

```xml
<profiles>
  <profile>
    <id>unit-test</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
      <skipITs>true</skipITs>
    </properties>
  </profile>
  <profile>
    <id>integration-test</id>
    <properties>
      <skipTests>true</skipTests>
      <skipITs>false</skipITs>
    </properties>
  </profile>
</profiles>
```

The xml snippet below shows a minium configuration for the failsafe plug-in. Pay attention to the `<skipTests>${skipITs}</skipTests>` which disable the regulare unit tests. The full [POM](https://github.com/fharms/java-examples/blob/master/combine-unit-and-integration-test/pom.xml) file

```xml
<plugin>
 <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-failsafe-plugin</artifactId>
 <version>2.18.1</version>
 <executions>
    <execution>
        <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>
        </goals>
         <configuration>
          <systemPropertyVariables>
            <!-- This make sure the EclipseIntegrationRunner will execute the tests -->
            <runEclipseIntegrationTests>true</runEclipseIntegrationTests>
          </systemPropertyVariables>
          <!-- this little trick prevent the unit test for running -->
          <skipTests>${skipITs}</skipTests>
        </configuration>
    </execution>
 </executions>
</plugin>
```

The tricky part was to prevent Eclipse from running the integration test when they co-exist with the other unit tests. Of course if you run as "Maven test" there is no problem, but I want to use the Eclipse "Run as JUnit Test" and exclude the integration tests

The trick was to create a JUnit runner to prevent integration tests for running. The [EclipseIntegrationRunner](https://github.com/fharms/java-examples/blob/master/combine-unit-and-integration-test/src/test/java/com/fharms/services/runner/EclipseIntegrationTestRunner.java) make sure the tests are only executed if the system property ** runEclipseIntegrationTests=true **

```java
@RunWith(EclipseIntegrationRunner.class)
public class MyIT { ... }

```

The down side is you have to remember adding the @RunWith(EclipseIntegrationRunner.class),
otherwise they will execute with the normal unit tests. Another down side is you can't use multiple @RunWith with the same test case.

## <a name="wheredoifindthecode"></a>[./](#wheredoifindthecode) Where do I find the code.

>Jump to the [source code](https://github.com/fharms/java-examples/tree/master/combine-unit-and-integration-test) on github
