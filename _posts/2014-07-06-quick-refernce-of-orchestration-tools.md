---
layout: post
title: "Quick Refernce of Orchestration Tools"
tagline : "Orchestration 工具快读"
description: ""
category: "DevOps"
tags: ["orchestration"]
---
{% include JB/setup %}

1. Openstack HEAT (Apache)

  Pros: The community is really active, will release with openstack milestone. 
  TOSCA, Cloud Formattion, Openstack Template native schema.

  Cons: Only work with openstack/AWS, need working with cloud-init which should be pre-installed in the image.

2. BOSH (Apache)

  Pros: Easy to use, community is active, have commercial customer – Cloud Foundry.

  Cons: Running agent in user instance, design for CF.

3. Puppet/Chef (Apache)

  Pros: A lot scripts existed.

  Cons: Weak in provisioning, more about configuration/state management.

4. vagrant (MIT)

  Pros: A programable way (in DSL) to do provisioning, with mutil-platform support.

  Cons: Mainly working for develop environment built

5. Ansible (GPLv3)

  This one is the most important competitor for HEAT, written in python and without any agent running on customer instance (work via ssh).

  Pros: No agent, from its document, it could do Application Deployment，Configuration Management，Cloud & Amazon (AWS, EC2) Automation，Continuous Delivery (with Jenkins, etc.) 

  Cons: A heavy one, even more heavy than HEAT. It needs a lot of configuration scripts to be re-written (as it contents a full configuration manage framework instead of puppet). 

6. Foreman (GPLv3)

  Redhat's solution which gives system administrators the power to easily automate repetitive tasks, quickly deploy applications, and proactively manage servers, on-premises or in the cloud.

7. Others

  7.1 Ubuntu Juju – It has a great GUI and could do drug-and-drop to provisioning service, looks like core part is not open sourced
  
  7.2 Redhat satellite – Work in redhat/centos, long term support.
