---
title: Manage development and delivery workflow with jGit-flow and Jenkins-Pipeline
layout: post
author: Dani Shemesh
permalink: /manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline/
tags:
- jira 
- jenkins
- pipeline
- git-flow
- jgit-flow
- ci/cd
- release
- deployment
date: 2018-09-10 14:40:45
source-id: 1Td5LRw5SmlrqLcm__0bKL8me9ylZu6Xgq1J0cF0FjHA
published: true
header-img: "img/pipes.jpg"
---

<i>This post is the first of a three parts series of articles on manage CI/CD workflow with jgit-flow and Pipeline</i>

* [Part-1: Tools and Planning](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-1)
* [Part-2: Git workflow with JGit-Flow](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2)
* [Part-3: Development and Release process with Jenkins Pipeline](https://fullgc.github.io/how-to-tune-akka-to-get-the-most-from-your-actor-based-system-part-2)

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

J[Git-Flow-Jira](https://github.com/FullGC/jgit-flow-jira) is a fork that I made for jgit-flow, which uses a Jira client to change the state of a Jira ticket during the lifecycle of a feature. Unfortunately, the project is not bug-free, and currently maintained mostly by the users and not by Atlassian. It is, however, published as open source and written very clearly. Jgitflow-jira contains a fix for this [open bug](https://ecosystem.atlassian.net/browse/MJF-109) as well.

**Jenkins(Pipeline)**

[Jenkins Pipeline](https://jenkins.io/doc/book/pipeline/) (or simply "Pipeline") is a suite of plugins which supports implementing and integrating *continuous delivery pipelines* into Jenkins.

As opposed to the historic Gui-driven CI/CD tools for Jenkins jobs, the definition of a Pipeline is written into a text file (called a [Jenkinsfile](https://jenkins.io/doc/book/pipeline/jenkinsfile)) as a code. This in turn can be committed to a project's source control repository.

We will use Pipeline for build, tests and release.

The Pipeline script would be written in Groovy and would use Jenkins syntax and shell commands. 

### **Complete development, release and deployment plan**

The flow-chart below represents the Jira, Git, and deployment flow that we'll learn how to implement in the following sections.

We'll review a development flow of the server team feature, 'ST-145’, and the process of releasing and deploying the next version: 1.2.0, of an application called 'volcano’.

There are many shapes and arrows in the graph, but there's no need to make sense of them all right now, since we’re going to do exactly that in the following sections.![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_1.png)

## Part 2: Git workflow with JGit-Flow:

Git workflow is based on git-flow, with some modifications, and implemented here with Jgit-flow-jira.

This section covers the green and purple steps in the workflow graph (note: add a link).

The basic plugin configurations needed for our story are from the 'master' branch, 'develop' branch, and the following:

<table>
  <tr>
    <td><configuration>
  <flowInitContext>
     <masterBranchName>master</masterBranchName>
     <developBranchName>develop</developBranchName>
     <featureBranchPrefix>ST-</featureBranchPrefix>
     <releaseBranchPrefix>release-</releaseBranchPrefix>
     <hotfixBranchPrefix>hotfix-</hotfixBranchPrefix>
     <versionTagPrefix>volcano</versionTagPrefix>
  </flowInitContext>
  <scmCommentPrefix>JgitFLow step: </scmCommentPrefix>
</configuration></td>
  </tr>
</table>


## **A Feature Lifecycle:**

### Start a feature git flow process

A new feature starts with the command **mvn jgitflow:feature-start. **

1. This prompts the user for a for the feature-branch name, which should carry the ticket ID (in our story, 'ST-145'):

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_2.png)

**A new feature branch is then checked out from 'develop'**

2. The ticket status is updated to 'IN PROGRESS':

	![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_3.png)

### Complete a feature git flow process

### The command** mvn jgitflow:feature-finish ends the feature lifecycle. **

1. This prompts the user for the desired feature name to finish, where the default is the current branch.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_4.png)

The feature branch is then merged into 'develop', and ‘develop’ is pushed.

Before the merge, we like branch 'develop' to be pulled and the commits in the feature branch to be squashed, and so cleaner.

To achieve that, we'll add the followings configuration to jgit-flow plugin:

<**pullDevelop**>true</**pullDevelop**>

<**squash**>true</**squash**>

2. When the feature is done, we like to resolve it and pass to the QA guy. Hence, it's resolution should be switched to 'Done’, and it’s status ‘QA’

	![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_5.png)

## **A Release Lifecycle:**

After all the features of the next version have been completed and merged to 'develop' branch, it’s time for the git flow release process to kick in.

### Start a release git flow process

A release process starts with the command **mvn jgitflow:release-start **on branch' develop’, which performs the following:

1. Prompts the user for the version name.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_6.png)

The default is the next major version (according to the pom file), in this case, 1.2.0

2. Pulls from the remote 'develop' branch (we’ve already configured).

3. Updates and commits the poms with the new version. This requires the following tag attribute: <**autoVersionSubmodules**>true</**autoVersionSubmodules**>

4. Checks out to the new 'release' branch, called release-VERSION (e.g. release-1.2.0).

5. Pushes (this would trigger a release process of a 'release candidate' for QA machines. The release processes will be described in the next section, ‘Pipeline’)

### Complete a release git-flow process

Once the version is approved by QA, the git flow release can be completed. The command "mvn jgitflow:release-finish" performs the following actions:

    1. Merges the 'release' branch back into ‘master’

    2. Tags the 'release' with its name.

    3. Back merges the 'release' into ‘develop’

    4. Updates 'develop's poms with "<next version>-SNAPSHOT". (1.3-SNAPSHOT).

    5. Removes the 'release' branch.

Checkout and push 'master' branch would start the release process

## **A Hotfix Lifecycle:**

When there is a need for a quick fix for a code that is already in production, the git-flow hotfix comes to the rescue.

### Start a hotfix git flow process

The 'master' branch is always synced with the latest deployment code. The Feature starts with the command **mvn jgitflow:hotfix-start**, which performs the following:

1. Prompts the user for the version name.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_7.png)

The default is the next minor version (according to the pom file), in this case, 1.2.1

2. Pulls from the remote "master" branch (done by adding the configuration **<****pullMaster****>**true**</****pullMaster****>**).

3. Updates and commits the poms with the new version.

4. Checks out to the new 'hotfix' branch, called hotfix-VERSION (e.g. release-1.2.1).

5. Pushes (like the stage that triggers a release of a hotfix candidate' for QA machines).

### Complete a release git-flow process

Once the version is approved by QA, the 'hotfix' git flow can be completed. The command "mvn jgitflow:hotfix-finish" performs actions which are quite similar to the ‘release-finish’:

1. Merges the 'release' branch back into ‘master’

2. Tags the release with its name.

3. Back merges the release into 'develop'

4. Restores poms version in 'develop' (the version is still 1.3-SNAPSHOT)

5. Removes the 'release' branch.

Checkout and push 'master' branch would start the release process

## **Summary **

We've reviewed the git workflow of a new feature, 'release’ and ‘hotfix’, and ended up with the following jgit-flow configuration:

<table>
  <tr>
    <td><configuration>
    <flowInitContext>
         <masterBranchName>master</masterBranchName>
         <developBranchName>develop</developBranchName>
         <featureBranchPrefix>ST-</featureBranchPrefix>
         <releaseBranchPrefix>release-</releaseBranchPrefix>
         <hotfixBranchPrefix>hotfix-</hotfixBranchPrefix>
         <versionTagPrefix>volcano</versionTagPrefix>
    </flowInitContext>
    <scmCommentPrefix>JgitFLow step: </scmCommentPrefix>
    <pullDevelop>true</pullDevelop>
    <pullMaster>true</pullMaster>
    <squash>true</squash>
    <autoVersionSubmodules>true</autoVersionSubmodules>
</configuration>
</td>
  </tr>
</table>


In Part 3(note: add a link) we'll review the Jenkins Pipeline job, which compiles, runs the tests and performs the release process.

# Part 3: Development and Release process with Jenkins Pipeline

The [Pipeline plugin](https://wiki.jenkins-ci.org/display/JENKINS/Pipeline+Plugin), allows users to implement a project's entire build/test/deploy pipeline in a Jenkinsfile and stores that alongside their code.

*A **Disclaimer before we begin writing the Jenkinsfile. There are endless ways to implement a CI/CD process. The release and deploy steps we'll discuss and implement are just one approach. Moreover, there are usually multiple ways of writing most of the commands in the Jenkinsfile: Native Groovy (Pipeline plugin DSL is groovy based), use a shell script, a Pipeline script code, external libraries (could be written in Java), etc..  *

### Multibranch Pipeline

In a [Multibranch Pipeline](https://jenkins.io/doc/book/pipeline/multibranch/) project, Jenkins automatically discovers, manages and executes Pipelines for branches which contain a Jenkinsfile in source control (It is possible to write a Pipeline script directly in the job configuration though). It enables an implementation of different Jenkinsfiles for different branches. However, here we're going to implement a single Jenkinsfile for multiple branches.

#### Configuration

The entire definition of the Pipeline would be written in the Jenkinsfile, except the following: In the 'Branch Source' we’ll declare the source control and repository that we’ll work with (this can also be done with code). In addition, we’ll set the branch discovery strategy to "All Branches", meaning the job would start for every modification of the repository (i.e. for every push).

Then we'll exclude the release and 'hotfix’ branches (this will be explained later).

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_8.png)

### Writing the Jenkinsfile, step-by-step

1. **Context**. The Pipeline job should be run on a dedicated Jenkins slave, 'server CICD', hence the script would be written inside a node context:

<table>
  <tr>
    <td>node('Server CICD) {

   }</td>
  </tr>
</table>


2. **Checkout**. This step checkouts code from source control. Scm is a special variable which instructs the checkout step to clone the specific revision which triggers this Pipeline run.

<table>
  <tr>
    <td>stage('Checkout') {
   checkout([
           $class           : 'GitSCM',
           branches         : scm.branches,
           extensions       : scm.extensions + [[$class: 'LocalBranch', localBranch: '']],
           userRemoteConfigs: scm.userRemoteConfigs
   ])
}</td>
  </tr>
</table>


3. **Build**.

a. **Maven build**: We are using the maven build tool. Maven was built by a shell command. We like to get a detailed report from Pipeline on a failure, including failed tests, links to them, and statistics. Moreover, we like the job status to become automatically 'unstable' if there were failed tests. These are provided by the [Pipeline Maven plugin](https://wiki.jenkins.io/display/JENKINS/Pipeline+Maven+Plugin), which wraps the maven build command.

<table>
  <tr>
    <td>withMaven(jdk: 'JDK 8 update 66', maven: 'Maven 3.0.5') {
           sh "mvn -Dmaven.test.failure.ignore=true clean install -Dsetup-profile=automation"
       }</td>
  </tr>
</table>


![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_9.png)

b. **Handle build exceptions** and test failures: On maven build failure:

If Pipeline checked out a feature branch (triggered by a push to a branch which starts with 'ST-'  ),  a notification email should be sent to the feature owner only. We’ll use the [Mailer plugin](https://wiki.jenkins.io/display/JENKINS/Mailer) for that.

Otherwise, we like an email notification to be sent to all server members, and a notification to the slack channel as well ([Slack plugin](https://jenkins.io/doc/pipeline/steps/slack/)). This should include a list of the last Git commits with the committer name, so that we can get an idea of what code modification broke the build.

If an exception has been thrown during the build, we like to:

1. Catch it

2. Change the build status to 'failure'

3. Send the appropriate notifications

4. Throw the exception


The final script for the build looks like this:

<table>
  <tr>
    <td>String branch = env.BRANCH_NAME.toString()
stage('Maven build') {

    //returns a set of git revisions with the name of the committer
   @NonCPS
   def commitList = {
       def changes = ""
       currentBuild.changeSets.each { set ->
           set.each { entry ->
               changes += "${entry.commitId} - ${entry.msg} \n by ${entry.author.fullName}\n"
           }
       }
       return changes
   }

   def handleFailures = {
       if (branch.startsWith("ST-")) {
           step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: emailextrecipients([[$class: 'RequesterRecipientProvider']]), sendToIndividuals: true])
       } else {
           step('Send notification') {
               mail(to: 'server@fullgc.com',
                       subject: "Maven build failed for branch: " + env.BRANCH_NAME.toString(),
                       body: "last commits are: " + commitList().toString() + " (<$BUILD_URL/console|Job>)");
           }
           slackSend channel: 'server', color: 'warning', message: "Maven build failed for branch ${branch} \". \nlast Commits are: \n" + commitList().toString() + "\n (<$BUILD_URL/console|Job>)"
       }
   }

   try {
       withMaven(jdk: 'JDK 8 update 66', maven: 'Maven 3.0.5') {
           sh "mvn -Dmaven.test.failure.ignore=true clean install -Dsetup-profile=automation"
       }
       if (currentBuild.result("UNSTABLE")) {
           handleFailures()
       }
   } catch (Exception e) {
       manager.buildFailure()
       handleFailures()
       throw e
   }
}</td>
  </tr>
</table>


4. **Release process**. In this process, we'll upload a tar (the maven build output) to s3, where the environment depends on the git branch we’re working on. The code would be placed in the 'process’ step:

              a.** Release tar name. **The release file is a tar file (the maven

             build output). Its name should

             represent the release version. The release version is found in

             the root pom.xml file, and we'll

             extract it from there



b. **Release candidate tar name. **This is somewhat tricky.

On **release/hotfix **only, we like to create a release candidate for QA.

The release candidate holds the name:

<volcano version>RC-<RC number>

             In our story, the first release candidate would be: 'volcano-1.2.0-RC-1'(‘volcano-1.2.1-RC-1’ in the case of a hotfix)

             The RC number starts with 1. If a QA person found a bug, we'd need to fix it, and then increment the RC number, i.e. 'volcano-1.2.0-RC-2’ and so on.

             We'll use a text file with the current RC number to know what the next version should be for release. We then update the file, commit changes and create a new tart with the correct name.

<table>
  <tr>
    <td>def pom = readFile 'pom.xml'
def project = new XmlSlurper().parseText(pom)
String version = project.version.toString()
String tarName = "volcano-${version}-release-pack.tar.gz"

stage('Release'){

@NonCPS
def newVersion = {
   def doesFileExist = fileExists 'releases.txt'
   if (doesFileExist) {
       echo "file releases.txt exists"
       def file = readFile 'releases.txt'
       String fileContents = file.toString()
       Integer newTarVersion = fileContents.toInteger() + 1
       writeFile file: 'releases.txt', text: newTarVersion.toString()
       echo "new tar version is: " + newTarVersion
   } else {
       echo "file releases.txt does not exist"
       writeFile file: 'releases.txt', text: "1"
   }
   readFile 'releases.txt'
}

if (branch.equals("release") || branch.equals("hotfix")) {
   String tarRCVersion = newVersion()
   newTarName = "volcano${version}-RC-${tarRCVersion}-release-pack.tar.gz"
   sh "mv ./volcano/target/${tarName} ./viper/target/${newTarName}"
   tarName = newTarName
}
}
step('Commit and push releases file') {
   sh "git remote set-url origin git@bitbucket.org:fullgc/volcano.git"
   sh "git add -A"
   sh "git commit -m 'update volcano version to '${tarRCVersion}"
   sh "git push"
}</td>
  </tr>
</table>


             Note that we did exclude the release/hotfix branches. This allows a couple of team members to work on the branch when QA has made a rejection or there is a bug to fix, without the new version being released with every push.



**c. Upload tar to s3. **Chef will deploy a new volcano version, with the appropriate version in s3.

This would require amazon s3 credentials.

For the upload itself, there is a pipeline script. Nevertheless, we'll implement it here using

Amazon CLI commands, using the shell.

<table>
  <tr>
    <td>step('Upload tar to s3 cli') {
   withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
       sh "pip install --user awscli"
       sh "sudo apt-get -y install awscli"
       sh "aws s3 cp ./volcano/target/${tarName} s3://fullgc/tars/"
   }
}</td>
  </tr>
</table>


5. **Deployment process. **Here we won't deploy using Pipeline, but by the deployment tool, Chef in our case. We won’t go too deeply into how Chef performs a deployment, but suffice to say this: In order for Chef to know that there is a new 'volcano’ version it needs to deploy, the version in the [environment](https://docs.chef.io/environments.html) (qa or development or production) file needs to be updated to the new version.

    1. First, we'll check out the Chef repository and 'cd’  in the environments directory which the environment files rely on.

    2. Replace the current version of the appropriate environment for the new version. This can be done by shell tools like jq. Here we'll use Groovy.

    3. Commit and push changes

    4. Send a slack notification

    5. If the branch is develop or master, it removes the releases.txt file

<table>
  <tr>
    <td>if (!branch.startsWith('ST-')){
  stage('Deploy') {

  def incrementVersion = {
     def environment = {
     switch(branch) {
         case 'develop': 'development.json'
             break
         case 'master': 'production.json'
             break
         default: 'qa.json'
     }
  }
   def environmentFileContent = readFile environment
   def environmentJson = new groovy.json.JsonSlurper().parseText(environmentFileContent)
   environmentJson.default_attributes.volcano.version = "volcano-${version}"
   String environmentPrettyJsonString = new groovy.json.JsonBuilder(environmentJson).toPrettyString()
   environmentJson = null
   writeFile file: $environment, text: environmentPrettyJsonString

  sh "git commit -am 'Changed volcano version in $environment env to '${version}"
   sh "git push"
}

checkout changelog: false, poll: false, scm: [$class: 'GitSCM', browser: [$class: 'BitbucketWeb', repoUrl:     'https://bitbucket.org/fullgc/chef'], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'LocalBranch', localBranch:
  '**']], submoduleCfg: [], userRemoteConfigs: [[url: 'git@bitbucket.org:fullgc/chef']]]
   dir('environments/') {
       incrementVersion()
   }
slackSend channel: 'server', color: 'good', message: " New volcano version ${version} is being deployed to $environment"
  }
if (branch == 'develop' || branch == 'master') sh "rm releases.txt"
}</td>
  </tr>
  <tr>
    <td></td>
  </tr>
</table>


In the image below an example of a Pipeline run:

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_10.png)

### Tips

* To save a lot of time, use the [Pipeline Syntax](https://jenkins-prod.inner-active.mobi/job/DevOps/job/Terraform/job/Clean_Validation/pipeline-syntax/) for every pipeline command

* While working on the Jenkinsfile, you don't have to commit and push every modification just to test an execution. You can use run an execution using ['replay’](https://jenkins.io/doc/book/pipeline/development/#replay) until everything works.

* [Jenkins Blue Ocean](https://jenkins.io/doc/book/blueocean/) plugin provides a new awesome user experience, check it out.

* Pipeline is still new, but you shouldn't get too frustrated by weird errors you may get while writing the script, most of them are common and the solution can be easily found on the web.

* The [Pipeline Unit Testing](https://github.com/jenkinsci/JenkinsPipelineUnit/) Framework allows you to unit test Pipelines before running them in full.


## **Wrapping up**

Pipeline as a code is pretty much a game changer, in the sense that it is now in the hands of every programmer, allowing them to write a full release (and deployment) process, that can fit the development workflow easily.

------------------------------------------------------------------------------------------

*The complete code can be found in my **[GitHu*b](https://github.com/FullGC/volcano_manage_cicd)*.*

