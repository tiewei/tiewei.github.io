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

本文介绍在Cloud Foundry v1版本中，Cloud Controller、Stager、DEA 是如何协同工作，实现app正常运行的。在Cloud Foundry V2版本中引入了dea\_next和cc\_next，并且Stager操作将在dea中进行，在处理流程中存在一些不同，将专门写blog介绍。

Cloud Foundry作为开源PaaS平台，对外提供的核心功能主要有两种 —— 为用户代码提供运行环境(runtime/framework)和为用户代码提供常用服务(数据库/消息队列等)。前者加上用户代码在Cloud Foundry的术语中称作一个Droplet，后者称为Service。本文介绍前者的主要工作流程，后者将写专文介绍。

## Cloud Foundry APP 相关组件

当用户push一个app给Cloud Foundry，到最终app能够正常运行，主要由以下7个组件协同工作：

1. **vmc** :
v1版客户端，这里主要的操作就是利用`vmc push`将用户程序代码上传给Cloud Foundry，以及利用`vmc apps`获取app运行状态。代码见<https://github.com/cloudfoundry/vmc>， 本文所采用版本为0.5.1

2. **nats** :
核心组件，Cloud Foundry内部的消息队列服务，cloud\_controller, stager, dea, health\_manager\_next之间的消息请求都是在消息队列中发送的。在Cloud Foundry中是一个存在单点失败(single points of failure - SPOF)的节点，如何避免SPOF的问题已经有一些讨论，见<https://github.com/cloudfoundry/cf-release/issues/32>

3. **cloud\_controller** (下简称cc):
核心组件，Cloud Foundry响应用户请求的节点，可以很好的横向扩展，是一个ROR项目。在部署应用的过程中主要是响应用户请求，管理用户代码，请求stager和dea进行app的部署，并订阅health\_manager\_next的消息，响应用户查看app状态的请求。该节点在Cloud Foundry中可以很方便地横向扩展，代码见 <https://github.com/cloudfoundry/cloud_controller>.

4. **stager** :
核心组件，Cloud Foundry中将用户代码和runtime/framework结合起来生成droplet的关键节点。目前提供java, ruby, python等rumtime，以及node.js,play, rake, rails, sinatra, java\_web等很多framework(java\_web就是提供了tomcat容器)。可扩展，代码见<https://github.com/cloudfoundry/stager>

5. **dea** :
核心组件，Cloud Foundry中运行droplet的节点，droplet包含用户app以及所需runtime和framework，以及所需的依赖包和start/stop脚本，dea只是提供运行用户代码的容器，提供最基本的隔离功能(v2版本的dea_next利用warden实现的LXC子集能够实现更好的隔离)。可扩展，代码见<https://github.com/cloudfoundry/dea>

6. **debian\_nfs\_server** :
可选组件，cloud\_controller保管用户代码的后台NFS存储服务，如果没有部署，则在cloud\_controller本地文件系统存储

7. **health\_manager\_next** :
可选组件，监控Cloud Foundry中app和各个组件的运行状态。这里主要是利用health\_manager\_next监控用户app的状态，如果没有部署，在用`vmc apps`查看app状态时，status信息会为n/a，不可扩展，代码见<https://github.com/cloudfoundry/health_manager>

8. **router**:
必须组件，当APP部署完成后，dea向router注册app的url，未来向该app发送的访问请求(url=xxx.cf.com)都会被转发至对应的dea处理，而操作请求(url=api.cf.com/apps/xxx)仍然转发至cc处理

## PUSH APP时组件操作概述

1. **vmc -> cloud_controller** : vmc根据用户代码新建app -> 上传用户app代码，此处通过REST API交互

2. **cloud_controller** : cc响应vmc请求，创建app，存储app代码

3. **cloud_controller -> stager** : cc将app相关消息发送至NAT，由stager接收。此操作是利用stager-client进行的，代码见<https://github.com/cloudfoundry/stager-client>

4. **stager  -> cloud_controller** : 接收到消息的stager根据消息中的download\_uri从cc下载app的代码包，并利用vcap-staging制作droplet，完成后将droplet上传至cc。vcap-staging完成了整个部署APP过程中其中最为关键的操作，包括下载/安装/上传依赖包，打包运行环境，编辑start/stop脚本等，代码见<https://github.com/cloudfoundry/vcap-staging>

5. **vmc -> cloud_controller** : vmc向cc发送启动命令，在等待启动过程中(step 6-7)，通过REST API `GET /apps/test/instances`获取启动状态

6. **cloud_controller -> dea** : cc在NAT上发布消息寻找合适的dea，dea接收消息后下载droplet，并根据设定的resources限额启动droplet，并在NAT上发布运行状态

7. **dea -> router** : dea根据app的url向router注册，注册信息通过NATS传递

## 代码分析

上一节简要介绍了工作流程，本节从代码内部，详细分析工作流程中的关键步骤。

### Step1: upload app code in cc 

由于create/update app的过程十分相似，因此，我们首先介绍上传app的过程。

vmc 将用户app的代码打包成zip，调用REST API上传zip包.这里上传的zip包中只包括更新部分的文件，如果文件的fingerprints在cc中已经存在，则在zip包中不会包含这些文件，并在HTTP HEAD中resources中标注这些已经在cc中的资源

	POST http://api.cf.com/apps/:app/application
	{:_method=>"put", :resources=>"[]", :application=>#<UploadIO:0x0000000180d788 @content_type="application/zip", @original_filename="test.zip", @local_path="/tmp/test.zip", @io=#<File:/tmp/test.zip>, @opts={}>}
	其中test为app的name

根据cc的`routes.rb`([github](https://github.com/cloudfoundry/cloud_controller/blob/master/cloud_controller/config/routes.rb#L20))，处理代码如下：

`AppsController#upload`-[github](https://github.com/cloudfoundry/cloud_controller/blob/master/cloud_controller/app/controllers/apps_controller.rb#L79)

	def upload
	   ...
	      file = get_uploaded_file
	      resources = json_param(:resources)
	      package = AppPackage.new(@app, file, resources)
	      @app.latest_bits_from(package)
	   ...
  	end

 这里将上传对应的app与上传的文件关联新建一个AppPackage对象。

`latest_bits_from(app_package)`-[github](https://github.com/cloudfoundry/cloud_controller/blob/master/cloud_controller/models/app.rb#L326)
	
	def latest_bits_from(app_package)
      sha1 = app_package.to_zip 
      unless self.package_hash == sha1
        ...
        unless self.package_hash.nil?
          FileUtils.rm_f(self.legacy_unstaged_package_path)
        end
        self.package_state = 'PENDING'
        self.package_hash = sha1
        save!
      end
    end

将对上传的文件利用`to_zip`方法进行处理，得出sha的值，并根据该值与数据库中app关联的package进行比较，如果不同，则更新sha值并将package_state设为PENDING状态

`AppPackage#to_zip`-[github](https://github.com/cloudfoundry/cloud_controller/blob/master/cloud_controller/models/app_package.rb#L7)

    def to_zip
      tmpdir = Dir.mktmpdir
      dir = path = nil
      check_package_size
      timed_section(CloudController.logger, 'app_to_zip') do
        dir = unpack_upload
        synchronize_pool_with(dir)
        path = AppPackage.repack_app_in(dir, tmpdir, :zip)
        sha1 = save_package(path) if path
      end
    ensure
      FileUtils.rm_rf(tmpdir)
      FileUtils.rm_rf(dir) if dir
      FileUtils.rm_rf(File.dirname(path)) if path
    end

此处在`check_package_size`检查package的大小是否超过限制(config中的`max_droplet_size`，默认512M)，`unpack_upload`将zip包解压，将其复制到resource pool





### Step2: create/update app in cc

### Step3: publish message to stage in cc

### Step4: stage app in stager

### Step5: publish message to start/stop instance

## Troube Shooting

1. create manifest failed - NoMethodError: undefined method "buildpack" for #<CFoundry::V1::App 'test'>

造成此问题的原因在于manifests-vmc-plugin-0.6.3.rc2的一个bug。此包会根据用户设定生成部署的manifest以便在日后部署能够自动进行大多数步骤。然而，由于V1版App对象没有buildpack属性(见cfoundry-0.5.3.rc7/lib/cfoundry/v1/app.rb)，而在manifests-vmc-plugin中会根据此属性生成manifest文件(代码见<https://github.com/cloudfoundry/manifests-vmc-plugin/blob/master/lib/manifests-vmc-plugin.rb#L189>).解决此问题的方法是将此行改为

`if app.respond_to?("buildpack") and buildpack = app.buildpack`

本人已经将此bug fix提交给repo的owner，pull request见此<https://github.com/cloudfoundry/manifests-vmc-plugin/pull/4>。







		GET http://api.cf.com/apps/test check not named
		GET http://api.cf.com/info get frameworks
		GET http://api.cf.com/info/runtimes get runtimes
		GET http://api.cf.com/info  get limits
		POST http://api.cf.com/apps 
		request {"name":"test","instances":1,"staging":{"model":"sinatra","stack":"ruby19"},"resources":{"memory":64}
		response {
		  "result": "success",
		  "redirect": "http://api.cf.com/apps/test"
		}
		GET http://api.cf.com/apps/test
		request {"name":"test","instances":1,"staging":{"model":"sinatra","stack":"ruby19"},"resources":{"memory":64}}
		response
		{
		  "name": "test",
		  "staging": {
		    "model": "sinatra",
		    "stack": "ruby19"
		  },
		  "uris": [

		  ],
		  "instances": 1,
		  "runningInstances": 0,
		  "resources": {
		    "memory": 64,
		    "disk": 2048,
		    "fds": 256
		  },
		  "state": "STOPPED",
		  "services": [

		  ],
		  "version": "47bfb5b4433f70629c4e02559b55ec38-0",
		  "env": [

		  ],
		  "meta": {
		    "debug": null,
		    "console": null,
		    "version": 1,
		    "created": 1372217637
		  }
		}
		PUT http://api.cf.com/apps/test
		 {"name":"test","instances":1,"state":"STOPPED","staging":{"model":"sinatra","stack":"ruby19"},"resources":{"memory":64,"disk":2048,"fds":256},"env":[],"uris":["test.cf.com"],"services":[],"console":null,"debug":null}

		POST http://api.cf.com/apps/test/application upload
		{:_method=>"put", :resources=>"[]", :application=>#<UploadIO:0x0000000180d788 @content_type="application/zip", @original_filename="test.zip", @local_path="/tmp/test.zip", @io=#<File:/tmp/test.zip>, @opts={}>}

		 PUT http://api.cf.com/apps/test
		 {"name":"test","instances":1,"state":"STARTED","staging":{"model":"sinatra","stack":"ruby18"},"resources":{"memory":64,"disk":2048,"fds":256},"env":[],"uris":["test.cf.com"],"services":[],"console":null,"debug":null}
		 GET http://api.cf.com/apps/test/instances
		 RESPONSE_BODY:
		{
		  "instances": [
		    {
		      "index": 0,
		      "state": "RUNNING",
		      "since": 1372217907,
		      "debug_ip": null,
		      "debug_port": null,
		      "console_ip": null,
		      "console_port": null
		    }
		  ]
		}