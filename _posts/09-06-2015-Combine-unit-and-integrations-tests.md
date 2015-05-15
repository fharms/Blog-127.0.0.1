---
layout: post
title: Combine unit and integrations tests in the same module
---
 
If you do a little research on how to combine unit and integration tests you will find
out there is at least two different ways of doing this, and both involves special Maven and IDE configuration.

The goal is:

*  Unit and Integration test should exist in the same source folder
*  Executing unit tests from the IDE should only run unit tests
*  Executing “mvn install” should only execute unit tests, unless other is specified “mvn install -P integration-test”

From what I found out, the prefer way is to setup a new source directory for the module/project containing the integration tests and use the [build-helper-maven-plugin](http://mojo.codehaus.org/build-helper-maven-plugin/ "http://mojo.codehaus.org/build-helper-maven-plugin/") to add an extra build source directory to maven.

Another solution is to place the integration tests in it's own module or multiple modules. The point here is the integration tests are separated from the unit test because they exist in another module location.

Both solutions comes with pro and cons, and in the end it all comes down to which solution is the most suitable for your project. But I think there is one big disadvantage with both solutions, it requires extra plug-ins and a number of configurations and special settings to make it work.

So I decided to come up with another solution which required minimum configurations and was easy to use and understand. The advantage of this solution is I use a feature in surefire and failsafe plugin to separate the unit and integration tests.

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

The tricky part was to prevent Eclipse from running the integration test when they co-exist with other unit test. Of course if you run as "Maven test" there is no problem, but I want to use the Eclipse "Run as JUnit Test" and exclude the integration tests

The trick was to create a JUnit runner to prevent integration for running when "Run as JUnit Test" is executed.

```java
@RunWith(EclipseIntegrationRunner.class)
public class MyIT { ... }

```

The down side is you have to remember adding the @RunWith(EclipseIntegrationRunner.class),
otherwise they will execute with the normal unit tests. Another down side is you can't use multiple @RunWith with your test cases. 

# Requirements for running the example

* Maven 3.x or higher
* JDK 7 or higher

# Run the example

*  Build and run only the unit tests
>mvn install
*  Build and run only the integration tests
>mvn install -P integration-test
