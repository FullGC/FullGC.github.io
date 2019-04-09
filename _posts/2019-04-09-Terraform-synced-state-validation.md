---
title: Terraform synced-state validation
layout: post
author: Dani Shemesh
permalink: /terraform-synced-state-validation/
tags:
- terraform
- hashicorp
- devops
source-id: 1O5aZbgZjmGAP1q-HsVoIREe62jcBqlT6iM5S_k5nhMw
published: true
date: 2019-04-10 14:15:45
header-img: "img/water-balance-black.jpg"
---
Terraform synced-state validation

Working with Terraform brings up some challenges. We are to focus on two of them:

1. Keeping Terraform's state consistent with the actual provider (i.e, a cloud provider) resources can be quite a challenge. Such inconsistencies are usually a result of changes performed directly in the cloud environment.


2. Keep the Terraform code synced with the Terraform state. When we like to apply changes in 	Terraform, we can:	

    1. Start by pushing the changes to the Terraform repository and then apply them. The Terraform state will be behind until the changes are affected.
 		

    2. Start by applying the changes locally, then push them. The Terraform state will be ahead until the code is pushed.	

The first can be mostly avoided if we forbid manual actions that could have been done by Terraform, but not entirely.

The real issue with the second is that after the first action (apply/push), Terraform will be out of sync. If the second action is delayed or not happening, the undesired inconsistent state will be kept.

To ensure eventual consistency here, you can automate the workflow with an automation framework (i.e. Jenkins).

However, we, the Fyber DevOps team, wanted to avoid such an automation process because we thought it would become a delaying factor, and we still manage to keep Terraform state synced with the code and the actual resources.

We do it with a little help of a Jenkins pipeline[ library](https://jenkins.io/doc/book/pipeline/shared-libraries/) script. The script initials, updates, and executes a 'terraform plan' command. If Terraform isn’t synced, it reports the unsynced components to the dedicated Slack channel. The Jenkins job is scheduled to run in the first and before the last working hour, on each of the Terraform environments, so we’ll be able to fix any inconsistency by the end of the day.

![image alt text]({{ site.url }}/public/F57qdjJQzC92gHvkkvffA_img_0.png)

*The script can be found on my [GitHub](https://github.com/FullGC/Terraform-Empty-Plan-Validator).

*It's important to note that the script is useful when you're working with the Terraform Recommended Workflow, specifically when having an environment file for each environment that defines the Terraform modules, while the resources lie in their own repository.*
