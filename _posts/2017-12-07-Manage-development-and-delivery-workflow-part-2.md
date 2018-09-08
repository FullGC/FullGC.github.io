---
title: Manage development and delivery workflow with jGit-flow and Jenkins-Pipeline - Part 2
layout: post
author: Dani Shemesh
permalink: /manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-2/
tags:
- jira
- jenkins
- pipeline
- git-flow
- jgit-flow
- ci/cd
- release
- deployment
date: 2018-09-05 14:30:45
published: true
header-img: "img/workflow-main.jpg"
---

<i>This post is the second of a three parts series of articles about manage development and CI/CD workflow with jgit-flow and Pipeline</i>

* [Part-1: Tools and Planning](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-1)
* [Part-2: Git workflow with JGit-Flow](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-2)
* [Part-3: Development and Release process with Jenkins Pipeline](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-3)

------------------------------------------------------------------------------------------

<br><br>
## Part 2: Git workflow with JGit-Flow:

Git workflow is based on git-flow, with some modifications, and implemented here with Jgit-flow-jira.

This section covers the green and purple steps in the workflow graph (note: add a link).

The basic plugin configurations needed for our story are from the 'master' branch, 'develop' branch, and the following:

````
<configuration>
  <flowInitContext>
     <masterBranchName>master</masterBranchName>
     <developBranchName>develop</developBranchName>
     <featureBranchPrefix>ST-</featureBranchPrefix>
     <releaseBranchPrefix>release-</releaseBranchPrefix>
     <hotfixBranchPrefix>hotfix-</hotfixBranchPrefix>
     <versionTagPrefix>volcano</versionTagPrefix>
  </flowInitContext>
  <scmCommentPrefix>JgitFLow step: </scmCommentPrefix>
</configuration>
````

<br><br>
### A Feature Lifecycle:

#### Start a feature git flow process

A new feature starts with the command mvn jgitflow:feature-start.

1. This prompts the user for a for the feature-branch name, which should carry the ticket ID (in our story, 'ST-145'):

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_2.png)

A new feature branch is then checked out from 'develop'

2. The ticket status is updated to 'IN PROGRESS':

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_3.png)

#### Complete a feature git flow process

The command 'mvn jgitflow:feature-finish' ends the feature lifecycle.

1. This prompts the user for the desired feature name to finish, where the default is the current branch.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_4.png)

The feature branch is then merged into 'develop', and ‘develop’ is pushed.

Before the merge, we like branch 'develop' to be pulled and the commits in the feature branch to be squashed, and so cleaner.

To achieve that, we'll add the followings configuration to jgit-flow plugin:
````
<pullDevelop>true</pullDevelop>
<squash>true</squash>
````
2. When the feature is done, we like to resolve it and pass to the QA guy. Hence, it's resolution should be switched to 'Done’, and it’s status ‘QA’

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_5.png)

<br><br>
### A Release Lifecycle:

After all the features of the next version have been completed and merged to 'develop' branch, it’s time for the git flow release process to kick in.

#### Start a release git flow process

A release process starts with the command 'mvn jgitflow:release-start' on branch 'develop’, which performs the following:

1. Prompts the user for the version name.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_6.png)

The default is the next major version (according to the pom file), in this case, 1.2.0

2. Pulls from the remote 'develop' branch (we’ve already configured).

3. Updates and commits the poms with the new version. This requires the following tag attribute:

````
<autoVersionSubmodules>true</autoVersionSubmodules>
````

4. Checks out to the new 'release' branch, called release-VERSION (e.g. release-1.2.0).

5. Pushes (this would trigger a release process of a 'release candidate' for QA machines. The release processes will be described in the next section, ‘Pipeline’)

#### Complete a release git-flow process

Once the version is approved by QA, the git flow release can be completed. The command "mvn jgitflow:release-finish" performs the following actions:

1. Merges the 'release' branch back into ‘master’

2. Tags the 'release' with its name.

3. Back merges the 'release' into ‘develop’

4. Updates 'develop's poms with '<next version>-SNAPSHOT'. (1.3-SNAPSHOT).

5. Removes the 'release' branch.

Checkout and push 'master' branch would start the release process

<br><br>
### A Hotfix Lifecycle

When there is a need for a quick fix for a code that is already in production, the git-flow hotfix comes to the rescue.

#### Start a hotfix git flow process

The 'master' branch is always synced with the latest deployment code. The Feature starts with the command 'mvn jgitflow:hotfix-start', which performs the following:

1. Prompts the user for the version name.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_7.png)

The default is the next minor version (according to the pom file), in this case, 1.2.1

2. Pulls from the remote "master" branch (done by adding the configuration).

````
<pullMaster>true</pullMaster>
````

3. Updates and commits the poms with the new version.

4. Checks out to the new 'hotfix' branch, called hotfix-VERSION (e.g. release-1.2.1).

5. Pushes (like the stage that triggers a release of a hotfix candidate' for QA machines).

#### Complete a release git-flow process

Once the version is approved by QA, the 'hotfix' git flow can be completed. The command 'mvn jgitflow:hotfix-finish' performs actions which are quite similar to the ‘release-finish’:

1. Merges the 'release' branch back into ‘master’

2. Tags the release with its name.

3. Back merges the release into 'develop'

4. Restores poms version in 'develop' (the version is still 1.3-SNAPSHOT)

5. Removes the 'release' branch.

Checkout and push 'master' branch would start the release process

<br><br>
### Summary

We've reviewed the git workflow of a new feature, 'release’ and ‘hotfix’, and ended up with the following jgit-flow configuration:

````
<configuration>
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
````

In Part 3(note: add a link) we'll review the Jenkins Pipeline job, which compiles, runs the tests and performs the release process.
