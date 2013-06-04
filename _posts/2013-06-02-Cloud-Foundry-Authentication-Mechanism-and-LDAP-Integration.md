---
layout: post
title: "Cloud Foundry Authentication Mechanism and LDAP Integration"
tagline: "Cloud Foundry 认证机制与LDAP集成"
description: "Cloud Foundry Authentication Mechanism"
category: cloudfoundry
tags: [Cloudfoundry, Authorization, LDAP]
published: false
---
{% include JB/setup %}

本文简要介绍Cloud Foundry的用户认证过程，以及相关项目组件，并介绍同企业LDAP认证集成的方法。

## Cloud Foundry 认证

Cloud Foundry 作为如今炙手可热的开源项目，借助其可以方便搭建企业内部PaaS平台，然而在集成时首先遇到的问题就是同企业内部系统中的授权系统（如LDAP）集成。

起初，Cloud Foundry的认证系统也从最初的在Cloud Controller组件中，用户将用户名和密码存在`cloud_controller`的数据库中，当登录时提供用户名和密码，并获取一个token，后续操作需要提供token进行验证后方能进行，如图所示：

![Cloud Foundry Authentication - 1][1]

进而增加了UAA (User Account and Authentication) 组件和ACM (Access Control Management)专门用于用户认证和访问控制(后者本文暂不涉及)，UAA采用多种开放的标准协议，支持[OAuth][4]验证，TOKEN采用标准的[JWT(JSON Web TOKEN)][5]格式封装，并对外开放[SCIM(System for Cross-domain Identity Management)][6]接口进行用户操作。在此基础上，需要授权与认证的组件就相当于OAuth验证的活动参与者，VMC作为客户，Cloud Controller作为第三方客户端，UAA则扮演服务提供方的角色，同时基于Cloud Foundry开发的系统也可以通过OAuth协议请求UAA授权和认证。此时结构如图所示：

![Cloud Foundry Authentication - 2][2]

然而，在私用云中往往涉及外部认证的场景，一方面来自企业认证(LDAP等方法)的用户在登录时需要能够进行外部验证以保证用户身份，另一方面用户在访问cloud foundry时需要能够继续通过UAA进行认证，为此又加入了login-server组件来支持外部授权，同时作为UAA的一个特殊客户端，可以申请UAA的TOKEN。最终形成了一个支持扩展的认证组件集合，如图所示。

![Cloud Foundry Authentication - 3][3]

与认证活动的相关的组件，主要有以下4个：

1. **vmc** : 
V1版客户端，由于V2近期将release，将采用cf客户端，但原理和功能与vmc一样，但由于最新的0.5.x在请求认证api上有些差异，不能正常请求login-server，本文所述方法vmc需要0.4.x或0.3.x版本。vmc是一个Ruby项目，代码见 <https://github.com/cloudfoundry/vmc>
2. **cloud_controller** :
VCAP的控制组件V1版本，主要功能是告知客户端(vmc)进行用户认证请求的地址，并且根据用户TOKEN请求UAA认证。cloud\_controller是一个ROR项目，代码见 <https://github.com/cloudfoundry/cloud_controller>
3. **uaa** :
用户认证模块，Cloud Foundry进行用户管理/认证的核心模块，后台DB保存用户信息，对外提供多种认证接口，如[OAuth 2][2]和[SCIM][4]，以及[JWT][3]格式的支持。uaa是一个Java Spring项目，代码见 <https://github.com/cloudfoundry/uaa>
4. **login-server** : 
如果需要额外的外部授权方式以及定制登录页面，可采用login-server进行，只进行授权，不进行用户管理，无db保存用户数据。login-server是一个Java Spring项目，代码见 <https://github.com/cloudfoundry/login-server>

## 认证过程

## 配置选项

## 代码调整

## Reference:

1. [OAuth 2][4] - token based authentication for web applications and APIs. Defines the client software as a role. Separates issuing tokens from how you use a token. Token issuance is defined both for browsers and for REST clients using a username/password. Token format is not defined by OAuth2, but one proposed standard format is JWT.

2. [JWT][5] - JSON Web Tokens, an upcoming standard format for structured tokens (containing data) which are integrity protected and optionally encrypted. 

3. [SCIM][6] - cross-domain user account creation and management. REST API for CRUD operations around user accounts 

4. <https://github.com/TieWei/uaa/blob/master/docs/UAA-CC-ACM-VMC-Interactions.rst>
5. <http://blog.cloudfoundry.com/2012/07/23/introducing-the-uaa-and-security-for-cloud-foundry>
6. <http://blog.cloudfoundry.com/2012/11/05/how-to-integrate-an-application-with-cloud-foundry-using-oauth2>
7. <http://blog.cloudfoundry.com/2013/02/19/open-standards-in-cloud-foundry-identity-services>
8. <http://blog.cloudfoundry.com/2012/10/09/securing-restful-web-services-with-oauth2>
9. <http://blog.cloudfoundry.com/2012/07/24/high-level-features-of-the-uaa>

# END

[1]: /images/CF-authorization-1.png "cloud-foundry-authorization-1"
[2]: /images/CF-authorization-2.png "cloud-foundry-authorization-2"
[3]: /images/CF-authorization-3.png "cloud-foundry-authorization-3"
[4]: http://oauth.net/2/ "oauth2"
[5]: http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html "JWT"
[6]: http://www.simplecloud.info/ "SCIM"