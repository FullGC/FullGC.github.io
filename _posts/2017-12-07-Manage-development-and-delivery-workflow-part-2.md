---
title: Manage development and delivery workflow with jGit-flow and Jenkins-Pipeline - Part 2
layout: post
author: Dani Shemesh
permalink: /manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-2/
tags:
- jira
- git-flow
- jgit-flow
- ci/cd
- Atlassian
date: 2018-09-11 14:10:45
published: true
header-img: "img/workflow-main.jpg"
---

<i>This post is the second of a three parts series of articles about manage development and CI/CD workflow with jgit-flow and Pipeline</i>

* [Part-1: Tools and Planning](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-1)
* [Part-2: Git workflow with JGit-Flow](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-2)
* [Part-3: Development and delivery process with Jenkins Pipeline](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-3)

------------------------------------------------------------------------------------------


## Part 2: Git workflow with JGit-Flow:

The Git workflow is based on 'git-flow', with some modifications, and implemented here with 'Jgit-flow-jira'.

This section covers the green and purple steps in the workflow graph ([from part-1](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-1)).

The following is the basic plugin configuration needed for our story. It gives a name to the Git branches and tags. 
The 'scmCommentPrefix' would be the prefix for the commits performed by 'jGit-flow'('i.e. the Git squash operation')

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

A new feature starts with the command <i>mvn jgitflow:feature-start</i>.

- This prompts the user for the feature-branch name, which should carry the ticket ID (in our story, 'ST-145'):

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_2.png)

A new feature branch is then checked out from 'develop' and the ticket status is updated to 'IN PROGRESS':

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_3.png)

#### Complete a feature git flow process

The command <i>mvn jgitflow:feature-finish</i> ends the feature lifecycle.

It prompts the user for the desired feature name to finish, where the default is the current branch.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_4.png)

The feature branch is then merged into 'develop', and ‘develop’ is pushed.

Before the merge, we like branch 'develop' to be pulled and the commits in the feature branch to be squashed, and so cleaner.

To achieve that, we'll add the followings configuration to jgit-flow plugin:
````
<pullDevelop>true</pullDevelop>
<squash>true</squash>
````
- When the feature is done, we like to resolve it and pass to the QA guy. Hence, it's resolution should be switched to 'Done’, and it’s status ‘QA’

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_5.png)

<br><br>
### A Release Lifecycle:

After all the features of the next version have been completed and merged to 'develop' branch, it’s time for the git flow release process to kick in.

#### Start a release git flow process

A release process starts with the command <i>mvn jgitflow:release-start</i> on branch 'develop’, which prompts the user for the version name.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_6.png)

The default is the next major version (according to the pom file), in this case, 1.2.0.
Then it performs the following actions:

- Pulls from the remote 'develop' branch (we’ve already configured).

- Updates and commits the poms with the new version. This requires the following tag attribute:

````
<autoVersionSubmodules>true</autoVersionSubmodules>
````

- Checks out to the new 'release' branch, called release-VERSION (e.g. release-1.2.0).

- Pushes (this would trigger a release process of a 'release candidate' for QA machines. (The release processes will be described in the next section, ‘Pipeline’)

#### Complete a release git-flow process

Once the version is approved by QA, the 'git-flow' release can be completed.
The command <i>mvn jgitflow:release-finish</i> performs the following:

- Merges the 'release' branch back into ‘master’

- Tags the 'release' with its name.

- Back merges the 'release' into ‘develop’

- Updates 'develop's poms with '<next version>-SNAPSHOT'. (1.3-SNAPSHOT).

- Removes the 'release' branch.

Checkout and push 'master' branch would start the release process

<br><br>
### A Hotfix Lifecycle

When there is a need for a quick fix for a code that is already in production, the 'git-flow' hotfix comes to the rescue.

#### Start a hotfix git flow process

The 'master' branch is always synced with the latest deployment code.
The Feature starts with the command <i>mvn jgitflow:hotfix-start</i> prompts the user for the version name.

![image alt text]({{ site.url }}/public/l8Up2rOYZomboTh06PZE0A_img_7.png)

The default is the next minor version (according to the pom file), in this case, 1.2.1

- Pulls from the remote 'master' branch (done by adding the configuration).

````
<pullMaster>true</pullMaster>
````

- Updates and commits the poms with the new version.

- Checks out to the new 'hotfix' branch, called hotfix-VERSION (e.g. release-1.2.1).

- Pushes (like the stage that triggers a release of a hotfix candidate' for QA machines).

#### Complete a release git-flow process

Once the version is approved by QA, the 'hotfix' git flow can be completed.
The command <i>mvn jgitflow:hotfix-finish</i> performs actions which are quite similar to the ‘release-finish’:

- Merges the 'release' branch back into ‘master’

- Tags the release with its name.

- Back merges the release into 'develop'

- Restores poms version in 'develop' (the version is still 1.3-SNAPSHOT)

- Removes the 'release' branch.

Checkout and push 'master' branch would start the release process

<br><br>
### Summary

We've reviewed the git workflow of a new feature, 'release’ and ‘hotfix’, and ended up with the following 'jgit-flow' configuration:

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

In [part-3](https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-3) we'll review the Jenkins Pipeline script, which compiles, runs the tests and performs the release process.

<div id="disqus_thread"></div>
<script>
/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
var disqus_config = function () {
this.page.url = "https://fullgc.github.io/manage-development-and-delivery-workflow-with-jgit-flow-and-jenkins-pipeline-part-2/"
this.page.identifier = workflow-2
};
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://FullGC.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>