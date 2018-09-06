---
title: Manage development and delivery workflow with jGit-flow and Jenkins-Pipeline
layout: post
author: Dani Shemesh
permalink: /manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-1/
tags:
- jira 
- jenkins
- pipeline
- git-flow
- jgit-flow
- ci/cd
- release
- deployment
date: 2018-09-05 14:40:45
published: true
header-img: "img/pipes.jpg"
---

<i>This post is the first of a three parts series of articles on manage CI/CD workflow with jgit-flow and Pipeline</i>

* [Part-1: Tools and Planning](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-1)
* [Part-2: Git workflow with JGit-Flow](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-2)
* [Part-3: Development and Release process with Jenkins Pipeline](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-3)

------------------------------------------------------------------------------------------

Manage development and delivery workflow with jGit-flow and Jenkins-Pipeline

This article describes a development and delivery workflow, from a Jira ticket to a version release (and deployment) using a popular stack, including Jira, Git, Maven, and Jenkins. 

## Part 1 - Tools and Planning

Let's start with a quick review of the tools we’ll use for the workflow implementation

### **Jira**

[Atlassian Jira](https://en.wikipedia.org/wiki/Jira_(software)) is a popular proprietary issue tracking system.

We'll manipulate Atlassian Jira feature tickets along the flow. This can be skipped if you don’t use Jira.

The project we'll manage would be part of the Server team (ST) and the feature that we like to implement and deploy would be ST-145.

Its initial ticket status is 'open', the resolution is ‘unresolved’:

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_0.png)

### **GitFlow**

[GitFlow](http://nvie.com/posts/a-successful-git-branching-model/) is a branching model for Git, created by Vincent Driessen.

The GitFlow workflow defines a strict branching model designed around the project release. It uses the following branches:

* Master: Stores the official release history. The origin/master is the main branch where the source code of HEAD always reflects a *production-ready* state.

* Develop: Serves as an integration branch for features

* Feature: Each new feature resides in its own branch. Feature branches use 'develop' as their parent branch. When a feature is complete, it gets merged back into ‘develop’

* Release: Supports preparation of a new production release.

* Hotfix: When a critical bug in a production version must be resolved immediately, a 'hotfix' branch may be branched off from the corresponding tag on the 'master' branch that marks the production version.

If you're new to git-flow, please take some time to read about it [in Driessen's post](http://nvie.com/posts/a-successful-git-branching-model/) or in [Atlassian's Guid](https://www.atlassian.com/git/tutorials/comparing-workflows#!workflow-gitflow)e.

### **Jgit-flow (Maven plugin)**

[JGit-Flow](https://bitbucket.org/atlassian/jgit-flow/wiki/Home) [maven plugin](https://mvnrepository.com/artifact/external.atlassian.jgitflow/jgitflow-maven-plugin) is a Java implementation of GitFlow, and like Jira, it was published by Atlassian. It was designed for releasing a maven-based project and includes many other useful features.

jGit-flow provides the following git-flow basic functionality:

* [jgitflow:feature-start](https://bitbucket.org/atlassian/jgit-flow/wiki/goals/feature-start) Starts a feature branch

* [jgitflow:feature-finish](https://bitbucket.org/atlassian/jgit-flow/wiki/goals/feature-finish) Merges a feature branch

* [jgitflow:release-start](https://bitbucket.org/atlassian/jgit-flow/wiki/goals/release-start) Starts a release

* [jgitflow:release-finish](https://bitbucket.org/atlassian/jgit-flow/wiki/goals/release-finish) Merges a release

* [jgitflow:hotfix-start](https://bitbucket.org/atlassian/jgit-flow/wiki/goals/hotfix-start) Starts a hotfix

* [jgitflow:hotfix-finish](https://bitbucket.org/atlassian/jgit-flow/wiki/goals/hotfix-finish) Merges a hotfix

Each feature contains many attributes, providing very useful functionality (described in the links), that we'll use later on.

### **Jgit-flow-jira**

[JGit-Flow-Jira](https://github.com/FullGC/jgit-flow-jira) is a fork that I made for jgit-flow, which uses a Jira client to change the state of a Jira ticket during the lifecycle of a feature. Unfortunately, the project is not bug-free, and currently maintained mostly by the users and not by Atlassian. It is, however, published as open source and written very clearly. Jgitflow-jira contains a fix for this [open bug](https://ecosystem.atlassian.net/browse/MJF-109) as well.


**Jenkins(Pipeline)**

[Jenkins Pipeline](https://jenkins.io/doc/book/pipeline/) (or simply "Pipeline") is a suite of plugins which supports implementing and integrating *continuous delivery pipelines* into Jenkins.

As opposed to the historic Gui-driven CI/CD tools for Jenkins jobs, the definition of a Pipeline is written into a text file (called a [Jenkinsfile](https://jenkins.io/doc/book/pipeline/jenkinsfile)) as a code. This in turn can be committed to a project's source control repository.

We will use Pipeline for build, tests and release.

The Pipeline script would be written in Groovy and would use Jenkins syntax and shell commands. 

### **Complete development, release and deployment plan**

The flow-chart below represents the Jira, Git, and deployment flow that we'll learn how to implement in the following sections.

We'll review a development flow of the server team feature, 'ST-145’, and the process of releasing and deploying the next version: 1.2.0, of an application called 'volcano’.

There are many shapes and arrows in the graph, but there's no need to make sense of them all right now, since we’re going to do exactly that in the following sections.![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_1.png)

![image alt text]({{ site.url }}/img/workflow.png)
