---
layout: post
title: "Cloud Orchestration In My View"
tagline : "Cloud Orchestration 之我见"
description: "Cloud Orchestration In My View"
category: "Cloud"
tags: [cloud, orchestration, BOSH, HEAT]
published: false
---
{% include JB/setup %}

随着 IaaS产品的日趋成熟和完善，围绕整个 IaaS平台的周边工具也日益丰富起来。作为 AWS的最有力的竞争者， Openstack日趋丰富的生态系统进一步拉近与AWS的距离。然而 IaaS平台归根结底是为了简化运维工作，方便运维和开发人员管理硬件资源，共同实现 APP的快速上线。这里包含了笔者认为 IaaS平台能够提供的最核心的价值 ——

**硬件资源池化** : 利用虚拟化的方法将异构的计算/网络/存储资源抽象成统一的上层模型，利用内在的调度方法管理内部的物理资源

**操作手段标准化** ：提供标准API能够对由 IaaS平台管理的物理资源进行操作，使通过软件定义计算/网络/存储成为可能，从而快速生成所需要的运行环境。

这两点作为 IaaS平台的核心价值，改变了软件开发人员的思维模式，同时也使得 Cloud Orchestration成为可能。本文的主要内容分成以下三个部分：

1. 介绍 Cloud Orchestration的概念和来源
2. 现有 Cloud Orchestration的工具，重点介绍 HEAT和 BOSH
3. 个人的一些看法

## Cloud Orchestration的概念和来源

wikipedia给出的[定义](http://en.wikipedia.org/wiki/Orchestration_\(computing\)) :

>>Cloud service orchestration therefore is the:

>>* Composing of architecture, tools and processes by humans to deliver a defined service

>>* Stitching of software and hardware components together to deliver a defined Service

>>* Connecting and automating of work flows when applicable to deliver a defined service

Orchestration就是将服务中涉及的工具、流程、结构描述成一个可执行的流程，结合软硬件的接口，使之可以按照描述自动化、可重复地交付。具体到Cloud Orchestration，主要就是通过定义一系列的元数据来描述交付过程中涉及的环境管理(OS. network, storage ..)、软件包管理、配置管理以及监控和升级操作，以便自动化和标准化地完成从申请资源到交付/升级服务的整个生命周期。

### 为什么需要Cloud Orchestration？

简单来说就是降低管理大规模集群和服务所需的人力和时间消耗，保证服务环境的一致性和可靠性。具体来说主要是面对以下三个现象：

**将IaaS当作单纯的VM生成器** 
- 这个现象非常普遍，因为IaaS最基本的功能就是提供VM，然而其提供资源标准操作手段的价值被忽略了，API才是云服务的入口。

**多种角色贯穿整个环境管理过程**
- OS由安全人员指定，网络/存储归运维人员，数据库等管理归中间件人员配置。多种角色之间如果沟通不畅，很可能导致环境发生不可预知的错误。

**多种环境管理不一致**
- 开发/QA/产品环境由不同的团队采用不同的方法维护，多种环境管理不一致可能会导致服务运行环境不可预期，出现错误难以复现等问题。

针对以上三个现象，所以我们需要一个统一的手段去管理从VM生成到服务交付中的各个环节，控制服务的运行环境。

### 如何实现Cloud Orchestration

正是由于IaaS提供了标准的操作计算、网络、存储资源的手段，才使得我们可以利用其资源池中的资源来 build所需要的服务。
主要有两个部分的工作

1. 定义DSL 
定义一个DSL来描述服务发布整个过程中的各个数据和操作。目前常用标准的有 _"AWS Cloud Formation"_ 和 _"TOSCA"_ ，当然也可以自己定义DSL，只有具有足够能力描述整个过程即可

2. 实现执行引擎(Execution Engine)
为DSL实现的一个标准执行引擎，用于执行由DSL描述的脚本。

## 现有工具介绍

_Cloud Orchestration_ 工具在2012年开始出现，影响较大的包括 _"AWS Cloud Formation"_ ,  _"Ubuntu Juju"_ ,  _"Openstack HEAT"_ 
以及 _"Pivotal BOSH"_ 。相比puppet/chef等配置管理工具，这些工具同IaaS平台结合起来，涵盖了软件交付的整个生命周期。

下图介绍了这些主流工具的功能图谱，如有纰漏请指出。

![orchestration-tools-overview](/images/orchestration-tools-overview.jpg)


以下我们简要介绍**BOSH**以及**HEAT**两个开源项目的结构。

### BOSH

### HEAT

## 个人理解