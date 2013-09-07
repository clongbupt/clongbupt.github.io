---
layout: post
category: 云计算
tags: cloudfoundry  community  ppt
description: 本文主要对7月中旬vmware组织的cloudfoundry社区分享的ppt进行阅读和总结
---

cloud foundry ppt总结
=====

## CFv2 Intro  俞勇 vmware   

[ppt地址](http://vdisk.weibo.com/s/KF4II/1374238063)

### 1. Cloud foundry V2版的主要更新

#### 1.1 Go Router

支持WebSocket以及其他方式   是指带Nginx或不带Nginx?

支持/varz和/healthz路由的http监控

Roadmap路线图

		* HTTP Connect support  HTTP连接支持
		* Non-HTTP protocols 非HTTP协议支持
		* App-specific metric gathering    应用特定信息收集
			|-- Throught   				吞吐量
			|-- Latency    				延时
			|-- Bandwidth  				带宽
			|-- HTTP Reponse codes		HTTP响应码
		

#### 1.2 DEA

一个有意思的等式：
	
	Droplet = app code + runtime + framework + container + scripts

Roadmap路线图

	* Placement pools 一个存放实例的池子
	* Shared runtimes 一个共享的运行时

参考资料， 补充： **李梦云**是国内CF Redhat移植的大神， 多关注他的github和博客

		http://limengyun.com/cloudfoundry/dea.html 

通过NATS，DEA同CC交互的例子

DEA启动会在NATS上发布消息

	DEA: NATS.publish(‘dea.start’, @hello_message_json)
	@hello_message_json for UUID, IP, Port, Version of the DEA

CC启动一个Droplet实例时

a. DEA监听NATS 对自己UUID的消息主题进行订阅
	
	DEA: NATS.subscribe(‘dea.#{UUID}.start’) {|msg|process_dea_start(msg)}
	Send message with the topic of “dea.#{uuid}.start” to start this DEA

b. CC通过NATS向DEA发布启动命令
	
	Cloud Controller: NATS.publish(‘dea.#{dea_id}.start’,json)
	DEA get the msg, call process_dea_start(msg)
	msg = droplet_id, instance_index, service, runtime


TODO

#### 1.3 Warden

科普知识：

* 源自于[Cgroup](http://en.wikipedia.org/wiki/Cgroups), 非硬件类[Hypervisor](https://en.wikipedia.org/wiki/Hypervisor)的一种隔离技术

* Warden是一个孤立的环境，Droplet只可以获得受限的CPU,内存，磁盘访问权限，网络权限

* 目前在GCE，OpenShift等IaaS/PaaS平台中广泛使用

* 满足应用隔离要求的同时，获得高性能和伸缩性

* **不直接使用Cgroup，以确保Cloud Foundry DEA对操作系统无依赖性**


#### 1.4 Build Pack

New CF CLI 新的CF客户端

	* 支持http代理?
	* 支持CC_ng的api

Org & Space & Users 机构 -> 空间 -> 用户 三层角色结构

	* org是CF架构里面最顶级的对象
	* 一个机构(org)可以包含多个空间(space)
	* 用户可以是机构管理员, 机构审核员 ... 你能想到的很多的有三层角色组成权限集合

大致关系如下：

	Org 
		|-- Domains
			|-- Routes <--> Applications
		|-- Spaces
			|-- Applications <--> Routes
			|-- Services
		|-- Users

#### 1.5 Customized domain

	* 更新DNS的cname **具体方法?**
	
	* 通过调用cd命令可以将`route`和`domain`映射

			cf map-domain somedomain.com
			cf map sintra-hello foo somedomain.com
			cf restart sinatra-hell

#### 1.6 Customized Buildpacks

主要分为三部分：

	* Runtime 运行时 Ruby/Java/Node/Python
	* Container 容器 Tomcat/Websphere/Karaf
	* Application .WAR / .rb / .py

如何使用

		cf push my-new-app --buildpack=git://github.com/johndoe/my-buildpack.git

相关资料

	* [BuildPacks List](https://devcenter.heroku.com/articles/third-party-buildpacks)
	* [buildPack示例](https://github.com/cdan/cf-mono-buildpack)
	* [CF的buildpack文档](http://docs.cloudfoundry.com/docs/using/deploying-apps/buildpacks.html) 

开发示例

探测 detect

			#!/usr/bin/env ruby 
			gemfile_path = File.join ARGV[0], "Gemfile" 

			if File.exist?(gemfile_path) 
				puts "Ruby" 
				exit 0 
			else 
				exit 1 
			end

编译 compile

			#!/usr/bin/env ruby 
			#sync output 

			$stdout.sync = true 

			build_path = ARGV[0] 
			cache_path = ARGV[1] 

			install_ruby 

			private 

			def install_ruby 
				puts "Installing Ruby" 
				# !!! build tasks go here !!! 
				# download ruby to cache_path 
				# install ruby 
			end

发布 release

config_vars：

			{ 
				"config_vars" => {}, # environment variables that should be set 
				"default_process_types" => {} # 
			}.to_yaml

default\_process\_types: 

			{ 
				"config_vars" => { "RACK_ENV" => "production" }, 
				"default_process_types" => { "web" => "bundle exec rackup config.ru -p $PORT" } 
			}.to_yaml

Roadmap 路线图

	* vFabric Import tool    ? 
	* Buildpack manager      buildPack管理器
	* Enhanced caching       增强的缓存
	* Version policy         版本管理策略


#### 1.7 Service Connector (CLI) 

`service gateway` 提供了一个接口，既可以是本地的、自身的服务， 也可以是外部的第三方服务

服务可以是在`service node`上面跑的服务如mysql，也可以是第三方的SaaS服务

`service gateway`的功能：

	* 广播service catalog
	* 向service_node发布create/delete/bind/unbind的命令
	* 向CC请求现存的服务实例和服务绑定情况, 并在本地做缓存  这个是存在内存中的  __外链__
	* 孤儿服务管理  孤儿服务是指在node上申请成功，但在CC数据库中没有记录的服务实例 
	* SaaS marketplace gateway   SaaS市场网关

RoadMap 路线图

	* API Update   估计是CC同Gateway的交互位置   目前仍然用的是V1接口
	* Multi-node support  多节点支持
	* Standalone services 单实例服务
	* looser coupling     松散耦合

Service Connector   ? TODO

	* Service type templates : Prompts for creating instances known service types such as Oracle and SQLServer 服务类型模板的支持   对于常见的服务提供
	* Option for a service instances shared across spaces   提供跨空间的服务实例共享功能选项

#### 1.8 其他

还有两个主要组件CC和UAA， PPT中也有涉及但篇幅不多

UAA的功能
	
	Token Server   一般会在HTTP头中得到  TODO增加service的理解
	ID Server (User management)
	OAuth Scopes (Groups)
	Login UI
	Access auditing


UAA Roadmap路线图

	LDAP Login Server
	Active Directory Login Server
	Horizontally scalable Login Server
	App User Management Service

CC RoadMap

	Availability zone aware placement
	Configurable network policy
	OAuth scope and role mapping
	Richer auditing with queries
	OpenStack Swift blob configuration


### 2. 其他资讯

cloudfoundry Enterprise 企业版即将推出 基于vSphere, 非常容易安装, 自带很多app

## Cloudfoundry实践 - fishbuaa(李梦云) - 网易有道   

[ppt地址](http://vdisk.weibo.com/s/KF8sy/1374238526)

主要内容是： 用源码方式部署一个cloudfoundry集群及其中的要点   **这个跟我们组类似**

PPT所说的视频： http://limengyun.com/cloudfoundry/index.html

PPT中的内容不太多, 而且有较多内容与我们做过的工作重复，更多内容需要参考他的博客理解，根据他的博客，我整理了一些要点：

1. ccdb 必须支持transaction

2. cc同uaa之间是进行的对称加密

3. Idap 自定义的验证源   即 cloud foundry中的login-server组件

4. web console 也是对官方的cfoundry进行封装, 同样基于sinatra框架， [console地址](https://github.com/limengyun2008/console)
	
			* 任务队列beanstalkd，解决部署问题
			* 根据guid创建svn目录

5.  CF V1.0 同 CF V2.0 的主要区别

	* 1.0中router使用的是nginx+lua+ruby server的方式，2.0使用了go语言gorouter，据称支持了websocket且极大提升了性能。
	* 2.0中cloud contoller新增了quota，org，space等新的概念，更方便的进行权限和资源管理。
	* 1.0中为应用打包使用的是stager组件，2.0中移除了该组件，将打包功能加入到dea中，并将所有语言的打包程序以submodule的形式放在buildpacks/vendors 目录下。
	* 完全重写了health manager
	*1.0里 dea可以独立运行，一个dea负责的所有app都以子进程的形式挂在dea主进程下。但2.0之后dea强依赖于warden提供的安全容器来运行app了

6. 小道消息 : 2.0用了go重写了1.0用nginx+lua嵌入脚本的router 改称gorouter 号称比1.0有4X的性能提升，如果属实，go前途无量。

7. 其他可能有用的组件

	Syslog Aggregator : 归集log的组件
	Collector : 监听nats中的各类消息来监控各组件的运行状态

8. Cloud Controller的主要功能:

	* 对apps的增删改读；
	* 启动、停止应用程序；
	* Staging apps（把apps打包成一个droplet）；   TODO完成
	* 修改应用程序运行环境，包括instance、mem等等；
	* 管理service，包括service与app的绑定等；
	* Cloud环境的管理；
	* 修改Cloud的用户信息；
	* 查看Cloud Foundry，以及每一个app的log信息。

9. CC和uaa对用户进行认证   主要是uaa在认证用户后生成token，然后用户请求cloud_controller时, 将这个token以request的Authentication header的形式伴随HTTP请求发送, 一个代码示例

	REQUEST: GET http://api.vcap.me/v2/spaces/1136694c-1bc2-495e-9aac-1b89894a6dcc/summary
	REQUEST_HEADERS:
	  Accept : application/json
	  Authorization : bearer eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiIzZjMyMTNmZS1jODMzLTQ4YmMtYTYwZi00ZmM0NTAzODk0ZjAiLCJzdWIiOiJkNmE2M2Q1OC04ZmQ4LTRhNzUtOGZhOC1mNWI2ZDJjOGQwMDAiLCJzY29wZSI6WyJwYXNzd29yZC53cml0ZSIsImNsb3VkX2NvbnRyb2xsZXIud3JpdGUiLCJvcGVuaWQiLCJjbG91ZF9jb250cm9sbGVyLnJlYWQiXSwiY2xpZW50X2lkIjoiY2YiLCJjaWQiOiJjZiIsInVzZXJfaWQiOiJkNmE2M2Q1OC04ZmQ4LTRhNzUtOGZhOC1mNWI2ZDJjOGQwMDAiLCJ1c2VyX25hbWUiOiJsaW15IiwiZW1haWwiOiJsaW15QHJkLm5ldGVhc2UuY29tIiwiaWF0IjoxMzcxMzcyNjQyLCJleHAiOjEzNzE5Nzc0NDIsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC91YWEvb2F1dGgvdG9rZW4iLCJhdWQiOlsib3BlbmlkIiwiY2xvdWRfY29udHJvbGxlciIsInBhc3N3b3JkIl19.cd11MxTrbCCpG_5fU9_DV1_bE9Nz_2lQ_c1kari1WXI
	  Content-Length : 0
	RESPONSE: [200]
	RESPONSE_HEADERS:
	  connection : keep-alive
	  content-length : 88
	  content-type : application/json;charset=utf-8
	  date : Mon, 17 Jun 2013 10:38:41 GMT
	  server : nginx
	  x-frame-options : sameorigin
	  x-vcap-request-id : a9f046ec-9e1d-423d-ae8e-985495504d08
	  x-xss-protection : 1; mode=block

10. cloud\_controller的数据库 需要事物支持 比如postgresql, 而如果选用mysql的话，请务必使用innodb或其它支持事物处理的引擎， 而默认的Myisam不支持事务。
示例请看lib/cloud\_controller/rest\_controller/model\_controller.rb

		# Create operation
		def create
		  logger.debug "create: #{request_attrs}"

		  json_msg = self.class::CreateMessage.decode(body)
		  @request_attrs = json_msg.extract(:stringify_keys => true)
		  raise InvalidRequest unless request_attrs

		  before_create if respond_to? :before_create
		  obj = nil
		  model.db.transaction do
		    obj = model.create_from_hash(request_attrs)
		    validate_access(:create, obj, user, roles)
		  end

		  [
		    HTTP::CREATED,
		    { "Location" => "#{self.class.path}/#{obj.guid}" },
		    serialization.render_json(self.class, obj, @opts)
		  ]
		end

11. Health Manager (简称HM) 主要负责监控app的状态，确保已经启动的app处于running状态，以及这些app的版本和instance数量是正确的。 这些确保机制主要是通过维护应用状态实现的，每个app有一个Actual State 实际运行状态，用来比较它和app的Desired State 期望状态。 当不匹配的情况出现的时候，就要把app的状态调整到期望状态，比如通过start/stop命令来控制missing/extra的instance。

12. cloud controller对请求的验证是通过解密在Authorization header中的token来实现的。 UAA可以仅仅起一个token分发者的作用，然后将用户认证的部分委托出去。

	cloudfoundry已经替我们考虑到了这一点。 只需要在cloud controller的配置项里配置到login这个配置为自己的登录服务器（假设叫login-server） 再使用cf或者cfoundry api 进行登录都会使用login-server来进行登录验证。

	其实登录请求分两种，一种是implicit登陆，即使用命令行客户端登陆的方式，另一种是专门给浏览器使用的登陆逻辑。 为了图省事我们浏览器和客户端都是用了同样的implicit登陆接口，直接把token存在cookie里，仅仅为了测试

13. dea的概念

	在提到buildpack之前 有必要解释一下DEA，dea全称是 droplet execution agency，即执行droplet的代理。

	droplet是cloudfoundry自创的一个概念，它是一个app的可运行实例配合实例启停脚本的压缩包。

	举例来说，如果我把一个php应用放置在某个路径下，然后将apache配置好，最后写一个启动脚本，然后将apache, php应用代码和启动脚本打成一个压缩包。 在另外一台环境完全相同的机器上，你只需要下载这个压缩包，解压到对应目录下，然后启动脚本，应用就可以完美的复制到这台机器上。 这就是cloudfoundry进行动态扩容的原理和基础。 一个打好压缩包就是一个droplet，打压缩包的程序就叫buildpack，打包的过程叫staging。 不同的应用类型对应的buildpack代码也不同。如果要增加一门cloudfoundry默认不支持的语言或者应用类型，就需要自定义buildpack。

14. http://www.appfog.com 是一个基于cloudfoundry的paas，他自定义了许多语言的buildpack支持并在github上开源出来（github： https://github.com/appfog/ ）
