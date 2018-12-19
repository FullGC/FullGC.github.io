---
title: Developing behavior-driven tests for JEE web applications with Jbehave - Part 2
layout: post
author: Dani Shemesh
permalink: /developing-behavior-driven-tests-for-JEE-web-applications-with-Jbehave-part-2/
tags:
- jee
- java
- jbehave
- bdd
- automation
- tests
date: 2017-09-11 14:10:45
published: true
header-img: "img/behave-color.jpg"
---

<i>This post is the first of a three parts series of articles about Developing behavior-driven tests for JEE web applications with Jbehave</i>

* [Part-1: Terminology, Tools and the 'Volcano' stories](https://fullgc.github.io/developing-behavior-with-jbehave-part-1)
* [Part-2: Writing Stories and Java Implementation](https://fullgc.github.io/developing-behavior-driven-tests-for-JEE-web-applications-with-Jbehave-part-2)
* [Part-3: Automate the tests and generate reports](https://fullgc.github.io/developing-behavior-driven-tests-for-JEE-web-applications-with-Jbehave-part-3)

------------------------------------------------------------------------------------------

<br><br>
## Part 2 - Writing Stories and Java Implementation
Following is a discussion of some Volcano test cases and their Java implementation.

-----------------------------------------------------------------------------------
### **Registration story**

Let's zoom in on the first story, which is the Volcano registration story file:

````
Narrative:
As a Volcano enthusiast Dani would like to register to Volcano social network

Scenario: A new user is signing up for the Volcano system
Given dani shemesh is a Volcano enthusiast
When dani is signing up for Volcano with the user-name: dani and the password: 123456
Then dani is able to log in with the user-name: dani and the password: 123456
````

Note that the story is written in the third person. This will let us capture the subject (i.e. the user) later.

The 'Given' is linked with a Java method. The method should:

1.  Carry the Jbehave @Given annotation

2. The String parameter of the annotation should match the text described in the step, while the user name (i.e.)  'dani shemesh' is replaced with a variable $user

3. In order to capture the user parameter, the function receives a String parameter, following the @Named annotation

The @Given implementation of this scenario is a dummy for obvious reasons.

````java
public class UserAccount {
    Cache cache;
...
    @Given("$user is a Volcano enthusiast")
        public String user(@Named("user")String user) {
        return "let " + user + "be a volcano enthusiast";
}
````

Next is the implementation of the registration step where we cache the user details:

````javascript
@When("$user is signing up for Volcano with the user-name: $user and the password: $password")
public void  newUser(@Named("user")String user, @Named("password") String password) {
   RequestDispatcher.createUser(user, password);
   cache.getUsers().put(user, password);
}
````

And the verification step, where the user should be able to log in and have a live token.

````javascript
@Then("$user is $ableorNot to log in with the user-name: $user and the password: $password")
public void logIn(@Named("user")String user, @Named("ableorNotAble")String ableOrNot, @Named("password") String password) throws IOException {
   String token = RequestDispatcher.logIn(user, password).getResponseBody();
   if (Objects.equals(ableOrNot, "able")){
       assert !token.isEmpty();
       cache.getToken().put(user, token);
   }
   else assert token.isEmpty();
}
````

<br><br>
#### **Implementation challenges**

Note that, we have used a cache to store the new user's details in the registration step.

Considering other future user actions, like "change password" and “add a friend”, the user will be required to be logged-in, means that this would be a given step:

**Given** dani is logged in

Now, we like to:

1. Re-use the java implementation of the log-in method (reuse @Given methods)

2. Avoid creating a new user and log-in before each scenario  (reuse @When actions)

This brings up unexpected challenges from an OOP point of view. Before discussing them, let's explain why

<br><br>
#### The Jbehave environment

Though we are writing in Java, we are not in an OOP scope, but in Jbehave's.

When JBehave starts up, it registers all the implemented methods and their parameters, then reads the steps in each scenario for each story and maps them to the appropriate method in the code behind. Here all their values are in memory and are executed one by one. The memory is cleaned after each story.

<br><br>
#### Re-use the @Given methods

This one is easy. Instead of inheritance/composition, we can place the @Given methods wherever we like, and Jbehave will identify it. So, we'll create a dedicated class for given methods, dealing with user scenarios:

````javascript
public class GivenUserSteps {
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
}
````

##### Implement tests for similar entities in the same class

We can once again take advantage of the Jbehave environment to place test implementations with similar characters together, even if they implement steps of different stories, for example:

````javascript
public class UserAccount {

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
….
}
````

##### Re-use actions with Dependency Injection using Spring

In order to reuse actions, in our case the user that we have created and logged in with, in the "registration" step, we need to cache. We cannot put the cache in a global variable or static at some place, since we are working with different classes and the memory cleans up after each story.

Luckily, [Jbehave supports some of the most popular, Java based, dependency injection plugins](https://jbehave.org/reference/latest/dependency-injection.html). The most popular is probably Spring, and we'll use it to inject the resources (i.e. the cache) we need for reusing actions.

![image alt text]({{ site.url }}/public/dB6XOsGGWuUM1t1RHDV3g_img_5.png)

###### Configure Spring

We'll use 'org.springframework’ artifacts for the Spring integration:

````xml
<artifactId>spring-context</artifactId>
````

To make our test classes to be singleton spring beans, using the @Service annotation

````xml
<artifactId>spring-test</artifactId>
````

to identify the test classes beans under  com.fullgc.jbehave namespace, using @ContextConfiguration annotation and a spring-context xml file.

````xml
* <artifactId>spring-beans</artifactId>
````

To inject the resources to the test classes, using the @Autowired annotation

And the following Thucydides artifact

````xml
<groupId>net.thucydides</groupId>
````

````xml
<artifactId>thucydides-junit</artifactId>
````

               For the integration of the bean classes and with Jbehave, using SpringIntegration class.

The UserAccount test class now looks like this:

````javascript
@ContextConfiguration(locations = "/spring-context.xml")
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

````

Note that the new user is cached and reused for a "change password" action.

(We use the cache in the same fashion as the 'add friend' story, which we won’t cover here, but can be found in the ‘Volcano’ repository)

The other test classes should be modified in a similar fashion.

The spring context:

````xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:context="http://www.springframework.org/schema/context"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:task="http://www.springframework.org/schema/task"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-3.0.xsd">

   <context:component-scan base-package="com.fullgc.jbehave"/>
   <task:annotation-driven />
</beans>
````

<br><br>
### **Next**
In [Part-3](https://fullgc.github.io/developing-behavior-driven-tests-for-JEE-web-applications-with-Jbehave-part-3) we'll learn how to launch the web app in compile time, run the tests and generate summary reports.