---
title: Developing behavior-driven tests for JEE web applications with Jbehave
layout: post
author: Dani Shemesh
permalink: /developing-behavior-driven-tests-for-JEE-web-applications-with-Jbehave.md/
tags:
- jee
- java
- jbehave
- bdd
- automation
- tests
source-id: 1Td5LRw5SmlrqLcm__0bKL8me9ylZu6Xgq1J0cF0FjHA
published: true
---
Developing behavior-driven tests for JEE web applications with Jbehave

Behavior-driven development, or BDD, is an agile software development process that provides the developers, QA, project managers and business team with a shared tool-set and process for software development collaboration. 

In this guide, we'll learn to design, develop and automate [Black-box](https://en.wikipedia.org/wiki/Black-box_testing) tests for a JEE web application in a [BDD](https://en.wikipedia.org/wiki/Behavior-driven_development) fashion. We’ll develop on top of [Jbehave](https://jbehave.org/) framework. 

# Part 1 - Terminology, Tools and the 'Volcano' stories

## **Black-box testing**

According to Wikipedia, "Black-box testing is a method of software testing that examines the functionality of an application without peering into its internal structures or workings". As such, Black-box testing focuses entirely on the inputs and outputs of the software system – the “black box”.

## **Behavior-driven development(BDD)**

Behavior-driven development is an extension of [test-driven development](https://en.wikipedia.org/wiki/Test-driven_development) that makes use of a simple, domain-specific scripting language.

In BDD, you describe what you want the system to do by talking through example behavior. Work from the outside-in to implement those behaviors using examples to validate you're what you're building.

The customer (who may be a Scrum Product Owner) describes what they want, and the developers ask questions to flesh out enough detail about the behavior to be able to implement it". ([BDD in a nutshell](http://agilecoach.typepad.com/agile-coaching/2012/03/bdd-in-a-nutshell.html))

Structure:

* A BDD **story** is a description of a requirement and its business benefit, and a set of criteria by which we all agree that it is "done". There should be a story for each feature. 

* A Story consists of a **narrative** and one or more **scenarios**.

* A narrative is a short, simple description of a feature told from *the perspective of a person or role that requires the new functionality*. The narrative shifts the focus from writing features to discussing them ([technologyconversations.com](https://technologyconversations.com/2013/11/17/behavior-driven-development-bdd-value-through-collaboration-part-2-narrative/)).

* A scenario consists of **steps**, in the format of **'Given-When-Then'**. ‘Given’ and ‘When’ trigger actions, and ‘Then’ is the verification:

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_0.jpg)

## **Jbehave in a nutshell**

[JBehave](https://jbehave.org/) is an open-source framework for Behavior-Driven Development.

It supports Java-based development, and plain English is used to form the story. 

The steps in the story are visually linked to a corresponding Java method:

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_1.png)

JBehave supports multiple mechanisms for parameter injection. In the above example, the 'user' and ‘password’ as arguments extracted from the @When step, with the @named annotation, following a natural order to the parameters in the annotated Java method.

In addition, Jbehave provides an easy way to create more intelligent data types than these strings. There are multiple plugins for generating comprehensive and interactive reports.

There are many (many!) advanced features that are worth checking out (see advanced topics in the [Jbehave site](https://jbehave.org/reference/stable/reporting-stories.html)); we'll only be using a few of them. We’ll explain how Jbehave works at a lower level later.

## **Thucydides**

Thucydides is a tool designed to make writing automated acceptance tests easier.

Thucydides and JBehave work well together. Thucydides uses simple conventions to make it easier to get started in writing and implementing JBehave stories. It reports on both JBehave and Thucydides steps, which can be seamlessly combined in the same class.

## **Requirements and tools**

* There are JBehave plugins for IntelliJ-Idea and Eclipse. Both come with a custom JBehave Story Editor which provides a syntax highlighting, step hyperlink detection and link to corresponding Java method, step autocompletion, detecting both unimplemented steps and more. Hence, one of these IDE is required. 

* We use maven for import libraries, build, run and automated tests

* You can download and follow the source code through the guide. The dispatcher of the 'tests' module is written in Scala. To run it you’ll need a Scala SDK.

## **Jbehave Plugin and the 'Volcano' Stories**

We use Idea IntelliJ-IDE with 'Jbehave support' plugin for writing the stories and the code behind.  

### Volcano

'Volcano' is intended to be a social network for Volcano enthusiasts.

The project manager of 'Volcano' would like to add some basic features and behaviors: 

1. User account

    1. Registration to 'Volcano'.

    2. Change password.

2. User network

    3. Add a new friend.

Each feature will be described in it's own Jbehave story file. Here is how the "Registration" story looks like before applying the '‘Jbehave support’’ plugin:   

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_2.png)

And after:then: 

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_3.png)

At this stage the steps are marked in red, and we get a message when the mouse hovers that there is no Java method that linked with the steps.

### Initial Dependencies

Here Thucydides kicks in. 

We'll use '*net.thucydides’* libraries for the implementation. 

The following Jbehave plugin artifact:    

<table>
  <tr>
    <td><groupId>net.thucydides</groupId>
 <artifactId>thucydides-jbehave-plugin</artifactId></td>
  </tr>
</table>


This includes the Jbehave libraries that are required for the Java implementation code behind:

  ![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_4.png)

'Jbehave-core' provides the basic Jbehave BDD building blocks: The @Given @When @Then Annotations, and the annotations responsible for the parameter injection (i.e. @Named) ‘Jbehave-junit-runner’ provides functionality for the story and scenarios lifecycle and reporting.

Later on, we'll use it to identify the stories and run the test. 

## **Next**

In part two (//add link) we'll implement the 'Registration’ story, and review solutions to the implementation challenges.

# Part 2 - Writing Stories and Java Implementation

Following is a discussion of some Volcano test cases and their Java implementation.

## **Registration story**

Let's zoom in on the first story, which is the Volcano registration story file:  

<table>
  <tr>
    <td>Narrative:
As a Volcano enthusiast Dani would like to register to Volcano social network

Scenario: A new user is signing up for the Volcano system
Given dani shemesh is a Volcano enthusiast
When dani is signing up for Volcano with the user-name: dani and the password: 123456
Then dani is able to log in with the user-name: dani and the password: 123456</td>
  </tr>
</table>


Note that the story is written in the third person. This will let us capture the subject (i.e. the user) later.

The 'Given' is linked with a Java method. The method should:

1.  Carry the Jbehave @Given annotation

2. The String parameter of the annotation should match the text described in the step, while the user name (i.e.)  'dani shemesh' is replaced with a variable $user

3. In order to capture the user parameter, the function receives a String parameter, following the @Named annotation

The @Given implementation of this scenario is a dummy for obvious reasons. 

<table>
  <tr>
    <td>public class UserAccount
...
Cache cache;
...
@Given("$user is a Volcano enthusiast")
   public String user(@Named("user")String user) {
     return "let " + user + "be a volcano enthusiast";
}</td>
  </tr>
</table>


Next is the implementation of the registration step where we cache the user details:

<table>
  <tr>
    <td>@When("$user is signing up for Volcano with the user-name: $user and the password: $password")
public void  newUser(@Named("user")String user, @Named("password") String password) {
   RequestDispatcher.createUser(user, password);
   cache.getUsers().put(user, password);
}</td>
  </tr>
</table>


And the verification step, where the user should be able to log in and have a live token.

<table>
  <tr>
    <td>@Then("$user is $ableorNot to log in with the user-name: $user and the password: $password")
public void logIn(@Named("user")String user, @Named("ableorNotAble")String ableOrNot, @Named("password") String password) throws IOException {
   String token = RequestDispatcher.logIn(user, password).getResponseBody();
   if (Objects.equals(ableOrNot, "able")){
       assert !token.isEmpty();
       cache.getToken().put(user, token);
   }
   else assert token.isEmpty();
}</td>
  </tr>
</table>


## **Implementation challenges**

Note that, we have used a cache to store the new user's details in the registration step. 

Considering other future user actions, like "change password" and “add a friend”, the user will be required to be logged-in, means that this would be a given step:

**Given** dani is logged in

Now, we like to: 

1. Re-use the java implementation of the log-in method (reuse @Given methods)

2. Avoid creating a new user and log-in before each scenario  (reuse @When actions) 

This brings up unexpected challenges from an OOP point of view. Before discussing them, let's explain why

### The Jbehave environment

Though we are writing in Java, we are not in an OOP scope, but in Jbehave's. 

When JBehave starts up, it registers all the implemented methods and their parameters, then reads the steps in each scenario for each story and maps them to the appropriate method in the code behind. Here all their values are in memory and are executed one by one. The memory is cleaned after each story.

### Re-use the @Given methods

This one is easy. Instead of inheritance/composition, we can place the @Given methods wherever we like, and Jbehave will identify it. So, we'll create a dedicated class for given methods, dealing with user scenarios:

<table>
  <tr>
    <td>public class GivenUserSteps {
Cache cache;
...
@Given("$user is a Volcano enthusiast")
public String user(@Named("user") String user) {
   return "let " + user + "be a volcano enthusiast";
}

@Given("$user is logged in")
public void logIn(@Named("user") String user) throws IOException {
   String session = RequestDispatcher.logIn(user, cache.getUsers().get(user)).getResponseBody();
   cache.getToken().put(user, session);
}</td>
  </tr>
</table>


#### Implement tests for similar entities in the same class

We can once again take advantage of the Jbehave environment to place test implementations with similar characters together, even if they implement steps of different stories, for example:

<table>
  <tr>
    <td>public class UserAccount {
...
   Cache cache;

   @When("$user is signing up for Volcano with the user-name: $user and the password: $password")
   public void  newUser(@Named("user")String user, @Named("password") String password) {
       RequestDispatcher.createUser(user, password);
       cache.getUsers().put(user, password);
   }

   @When("$user is changing his password to $newPassword")
   public void  changePassword(@Named("user")String user, @Named("newPassword") String newPassword) throws IOException {
       String response = RequestDispatcher.changePassword(cache.getToken().get(user), user, cache.getUsers().get(user), newPassword).getResponseBody();
       if (response.equals("OK"))
           cache.getUsers().put(user, newPassword);
   }
….</td>
  </tr>
</table>


#### Re-use actions with Dependency Injection using Spring

In order to reuse actions, in our case the user that we have created and logged in with, in the "registration" step, we need to cache. We cannot put the cache in a global variable or static at some place, since we are working with different classes and the memory cleans up after each story.

Luckily, [Jbehave supports some of the most popular, Java based, dependency injection plugins](https://jbehave.org/reference/latest/dependency-injection.html). The most popular is probably Spring, and we'll use it to inject the resources (i.e. the cache) we need for reusing actions.

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_5.png)

##### Configure Spring

We'll use 'org.springframework’ artifacts for the Spring integration:

* <**artifactId**>spring-context</**artifactId**> 

To make our test classes to be singleton spring beans, using the @Service annotation 

* <**artifactId**>spring-test</**artifactId**> 

to identify the test classes beans under  com.fullgc.jbehave namespace, using @ContextConfiguration annotation and a spring-context xml file.

* <**artifactId**>spring-beans</**artifactId**>

To inject the resources to the test classes, using the @Autowired annotation

And the following Thucydides artifact 

* <**groupId**>net.thucydides</**groupId**>

<**artifactId**>thucydides-junit</**artifactId**>

               For the integration of the bean classes and with Jbehave, using SpringIntegration class.

The UserAccount test class now looks like this:

<table>
  <tr>
    <td>@ContextConfiguration(locations = "/spring-context.xml")
@Service
public class UserAccount {

   @Rule
   public SpringIntegration springIntegration = new SpringIntegration();

   @Autowired
   CacheBean cache;

   @When("$user is signing up for Volcano with the user-name: $user and the password: $password")
   public void  newUser(@Named("user")String user, @Named("password") String password) {
       RequestDispatcher.createUser(user, password);
       cache.getUsers().put(user, password);
   }

   @When("$user is changing his password to $newPassword")
   public void  changePassword(@Named("user")String user, @Named("newPassword") String newPassword) throws IOException {
       String response = RequestDispatcher.changePassword(cache.getToken().get(user), user, cache.getUsers().get(user), newPassword).getResponseBody();
       if (response.equals("OK"))
           cache.getUsers().put(user, newPassword);
   }
...
}
</td>
  </tr>
</table>


Note that the new user is cached and reused for a "change password" action.

(We use the cache in the same fashion as the 'add friend' story, which we won’t cover here, but can be found in the ‘Volcano’ repository)

The other test classes should be modified in a similar fashion.

The spring context:

<table>
  <tr>
    <td><?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:context="http://www.springframework.org/schema/context"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:task="http://www.springframework.org/schema/task"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-3.0.xsd">

   <context:component-scan base-package="com.fullgc.jbehave"/>
   <task:annotation-driven />
</beans></td>
  </tr>
</table>


# Part 3 - Automate the tests and generate reports

Previously, we implemented the step cases in a Java code.

In this part we'll learn how to automate them and generate informative reports at the end of the run. 

## **Identify implementation methods**

You may recall we mentioned that Jbehave registered the Java implementation methods on start up. But how does it identify an implementation step? 

Basically, these methods would have to override the configuration() method, which is part of the jbehave library. We have however, let 'thucydides' work for us. The ‘thucydides-core’ library (that comes with the ‘jbehave-plugin’), provides the ‘ThucydidesJUnitStories’ class. This identifies the ‘.story’ files and the links for the Java implementations. All we need is to have a Java class that inherits from it in the Java implementation root package:

<table>
  <tr>
    <td>import net.thucydides.jbehave.ThucydidesJUnitStories;
public class ApiBDDTestSuite extends ThucydidesJUnitStories {}</td>
  </tr>
</table>


#### Final project structure:

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_6.png)

## **Running the tests**

At this point, we can run the tests as Junit.

## Running the tests manually 

In order for the black-box tests to pass, we need to have a server up and running. To run the tests with the IDE, right click on the 'jbehave' package, 

  ![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_7.png)

In this fashion, we can run the tests quickly, debugging them and the server. 

## Automate the tests with Jetty(Maven plugin)

What we really want is to test the Volcano server as part of the maven build, the 'maven compile' task.  

In this case, we need maven to start a Volcano HTTP server, run the tests, and get Volcano down.

[Eclipse Jetty](https://www.eclipse.org/jetty/) is an open-source project providing an HTTP server and javax.servlet container

We'll use a Jetty maven-plugin, to start a Volcano server on which the tests will be running. 

The maven plugin should be configured as follows:

<table>
  <tr>
    <td><plugin>
   <groupId>org.eclipse.jetty</groupId>
   <artifactId>jetty-maven-plugin</artifactId>
   <version>${jetty.version}</version>
   <configuration>
       <httpConnector>
           <port>8083</port>
       </httpConnector>
       <scanIntervalSeconds>0</scanIntervalSeconds>
       <stopKey>stopApiTestEnv</stopKey>
       <stopPort>9999</stopPort>
       <jvmArgs>-Xms1024m -Xmx2048m -XX:PermSize=256M -XX:MaxPermSize=512M</jvmArgs>
       <contextHandlers>
           <contextHandler implementation="org.eclipse.jetty.maven.plugin.JettyWebAppContext">
               <war>${project.basedir}/../volcano_core/target/volcano_core-1.2-SNAPSHOT.war</war>
               <contextPath>/volcano</contextPath>
           </contextHandler>
       </contextHandlers>
   </configuration>
   <executions>
       <execution>
           <id>start-jetty</id>
           <phase>pre-integration-test</phase>
           <goals>
               <goal>start</goal>
           </goals>
       </execution>
       <execution>
           <id>stop-jetty</id>
           <phase>post-integration-test</phase>
           <goals>
               <goal>stop</goal>
           </goals>
       </execution>
   </executions>
</plugin>
</td>
  </tr>
</table>


Note that:

1. The server's port needs to be available, you may want to declare a different port from the default that your server is using.

2. The memory you declare allocated with the jvmargs needs to be enough to get the server up

3. In **contextHandlers **we provide the web application archive to be used. If you have multiple web modules in your project that need to be tested, provide them all. 

4. In the executions section we tell jetty to start before the tests are executed and stop afterwards. If we don't explicitly ask that, then the jetty would stay up.

## **Test Summary Reports**

Test Summary Reports are an important deliverable. They are needed to reflect test results in a clear way, allowing them to be analyzed quickly.

Jbehave [storyReporter](https://jbehave.org/reference/stable/reporting-stories.html) supports many report formats and will generate the reports for each step and story automatically after the tests have been completed in the /target/jbehave folder.

We'll use another Thucydides maven plugin to generate some better-looking reports:

<table>
  <tr>
    <td><plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-site-plugin</artifactId>
   <version>3.2</version>
   <configuration>
       <reportPlugins>
           <plugin>
               <groupId>net.thucydides.maven.plugins</groupId>
               <artifactId>maven-thucydides-plugin</artifactId>
               <version>${thucydides.version}</version>
           </plugin>
       </reportPlugins>
   </configuration>
</plugin>
</td>
  </tr>
</table>


This will generate various reports automatically at the end of the Maven run, in /target/site.

The 'index.html' file that has been created is the most comprehensive, interactive report:

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_8.png)

A click on the 'change password' test (story) for example, would provide some additional data on each step result.

	![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_9.png)

You can zoom in with a click on the various test components to get results and statistics. 

## **Wrapping up**

Behavior Driven Development is a methodology for developing software through example-based communication among developers, QAs, and project managers.

With Jbehave, this means writing Given-When-Then scenarios to illustrate examples in a plain text, which links to Java implementation methods.

As part of our mission to create Blackbox tests in a BDD fashion for the Volcano application, we've covered some test cases, discussed design and challenges, and reviewed some post-compilation reports.

------------------------------------------------------------------------------------------

*The complete code can be found in my **[GitHu*b](https://github.com/FullGC/volcano)*.*
