---
layout: post
title: "Workflow of Push App in Cloud Foundry v1"
tagline : "Cloud Foundry V1版本中部署应用的工作流程"
description: "Workflow of Push App in Cloud Foundry v1"
category: cloudfoundry
tags: [Cloudfoundry, cloud_controller, stager, dea]
published: false
---
{% include JB/setup %}

本文介绍在Cloud Foundry v1版本中，Cloud Controller、Stager、DEA 是如何协同工作，实现app正常运行的。在Cloud Foundry V2版本中引入了dea\_next和cc\_next，在处理流程中存在一些不同，将专门写blog介绍。

Cloud Foundry作为开源PaaS平台，对外提供的核心功能主要有两种 —— 为用户代码提供运行环境(runtime/framework)和为用户代码提供常用服务(数据库/消息队列等)。前者加上用户代码在Cloud Foundry的术语中称作一个Droplet，后者称为Service。本文介绍前者的主要工作流程，后者将写专文介绍。

## Cloud Foundry App 相关组件

当用户push一个app给Cloud Foundry，到最终app能够正常运行，主要由以下5个组件协同工作：

1. **vmc**
v1版客户端，这里主要的操作就是利用`vmc push`将用户程序代码上传给Cloud Foundry，以及利用`vmc apps`获取app运行状态。

2. **nats**
核心组件，Cloud Foundry内部的消息队列服务，cloud\_controller, stager, dea, health\_manager\_next之间的消息请求都是在消息队列中发送的。在Cloud Foundry中是一个存在单点失败(single points of failure - SPOF)的节点，如何避免SPOF的问题已经有一些讨论，见<https://github.com/cloudfoundry/cf-release/issues/32>

3. **cloud\_controller**
核心组件，Cloud Foundry响应用户请求的节点，可以很好的横向扩展，是一个ROR项目。在部署应用的过程中主要是响应用户请求，管理用户代码，请求stager和dea进行app的部署，并订阅health\_manager\_next的消息，响应用户查看app状态的请求。

4. **stager**
核心组件，Cloud Foundry中将用户代码和runtime/framework结合起来生成droplet的关键节点。目前提供java, ruby, python等rumtime，以及node.js,play, rake, rails, sinatra, java\_web等很多framework(java\_web就是提供了tomcat容器)。

5. **dea**
核心组件，Cloud Foundry中运行droplet的节点，droplet包含用户app以及所需runtime和framework，以及所需的依赖包和start/stop脚本，dea只是提供运行用户代码的容器，提供最基本的隔离功能(v2版本的dea_next利用warden实现的LXC子集能够实现更好的隔离)。

6. **debian\_nfs\_server**
可选组件，cloud\_controller保管用户代码的后台NFS存储服务，如果没有部署，则在cloud\_controller本地文件系统存储

7. **health\_manager\_next**
可选组件，监控Cloud Foundry中app和各个组件的运行状态。这里主要是利用health\_manager\_next监控用户app的状态，如果没有部署，在用`vmc apps`查看app状态时，status信息会为n/a

## Cloud Foundry 部署App流程



