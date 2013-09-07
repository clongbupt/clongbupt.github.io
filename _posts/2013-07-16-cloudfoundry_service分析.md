---
layout: post
category: 云计算
tags: cloudfoundry service
description: 本文章主要源自于项目组在向`Cloud Foundry V2`版本移植的大进程中, 在`msyql service`移植上遇到的一些问题总结。
---


## 一. 综述

本文章主要源自于项目组在向`Cloud Foundry V2`版本移植的大进程中, 在`msyql service`移植上遇到的一些问题总结。

由于V2版本更新迭代过快, 先列出我阅读的代码版本信息, 格式为[项目名](下载地址) - 版本号(github上提交版本前6位)：

			* [cloud_controller_ng](https://www.github.com/cloudfoundry) - d7e51a
			* [service-release](https://www.github.com) - 15d8f2
			* [	service-base](https://www.github.com) - ff9932

## 二. service综述

服务一直以来在PaaS平台上占据了很重要的位置，通过简单配置，用户便可以快速地部署应用，这也是PaaS平台最吸引人的地方之一。从分类上说，`Cloud Foundry V2`中的service主要有：

	1. 数据库服务(包括关系型和nosql)，常见的如：MySQL/Postgres/Redis/Mongodb等
	2. 存储类服务，比如：vblob/filesystem
	3. 其他服务，比如：rabbitMQ/Memcached等
	4. 第三方服务，用户自主开发和配置

从功能上看，service的结构通常包含两个组件：`service gateway`和`service node`。它们两者的功能分别叙述如下：

	* `service gateway`主要是广播其提供的服务(advertise service offerings)，并接收来自`cloud_controller`的控制命令, 主要有四种：创建服务实例，删除服务实例，绑定服务实例和解绑服务实例。`service gateway`同样会将从`CC(cloud_controller)`发送出来的命令传递给`service node`。

	* `service node`主要是接收`service gateway`转送的CC的命令，并执行这些命令，如果执行成功会将唯一标识该服务实例的`credencial`返回给`service gateway`。一般地，提供该项服务的服务器进程会跟`service node`一起跑在同一台机器上。

本文章主要描述`mysql_service`的功能和流程情况，主要包括如下功能：

	1. `mysql_gateway`的启动，主要包括启动时通过读取配置文件设置程序参数，向`CC(cloud_controller)`上报信息，启动thin服务器监听特定端口，以响应CC的HTTP请求， 连接NATS并监听相关通道；
	2. `mysql_gateway`对来自CC的HTTP请求的接收和处理；
	3. `mysql_gateway`通过NATS接收`msql_node`的信息，并转发CC的控制命令；
	4. `mysql_node`的启动，主要是启动时通过读取配置文件设置程序参数，连接NATS并发布信息，监听相关通道，连接服务的服务器进程；

下面将一一进行叙述。

### 2.1. mysql_gateway的启动

mysql_gateway的启动线路图，后面会对其进行详细解析，其中引用代码的格式如下：
	
	[项目名] - `简写文件路径` - 关键调用处 : 行数 | 关键调用处 : 行数

简写的文件路径主要有：

	1. [service-release] - bin/mysql_gateway  ->  service-release/src/mysql_service/bin/mysql_gateway.rb
	2. [vcap-services-base] - lib/base/gateway -> vcap-services-base-ff9932/lib/base/gateway.rb

具体流程如下：

	1. [serivce-release] - `bin/mysql_gateway` - Mysql::Gateway.new.start : 31行 | Mysql::Gateway : 13行
	2. [vcap-services-base] - `lib/base/gateway` - Base::Gateway.start : 77行 | Base::Gateway.async_gateway_class : 131行
	3. [vcap-services-base] - `lib/base/asynchronous_service_gateway` - AsynchronousServiceGateway.initialize : 17行 | super : 18行 
	4. [vcap-services-base] - `lib/base/base_async_gateway` - BaseAsynchronousServiceGateway.initialize : 35行 | setup : 40行
	5. [vcap-services-base] - `lib/base/asynchronous_service_gateway` - AsynchronousServiceGateway.setup : 22行 | CatalogManagerV2.new : 47行(调用*a) | send_heartbeat : 55行 | get_current_catalog : 504行 | GatewayServiceCatalog.new(调用*b) : 98行 | @catalog_manager.update_catalog(*c) : 502行 | 
		*a [vcap-services-base] - `lib/base/catalog_manager_v2` - CatalogManagerV2.initialize : 16行
		*b [vcap-services-base] - `lib/base/gateway_service_catalog` - to_hash : 10行
		*c [vcap-services-base] - `lib/base/catalog_manager_v2` - update_catalog : 80行
	6. [vcap-services-base] - `lib/base/http_handler` 
	7. [cf-uaa-lib-master] - `lib/uaa/token_issuer`
	8. [cf-uaa-lib-master] - `lib/uaa/http`

* 第1点和第2点是`service_gateway`的启动位置, 此处有`Mysql::Gateway`的定义, 该类继承自Base::Gateway。主要进行如下工作：
	
	* 处理启动参数 - OptionParser 
	* 读取配置文件 - @config  再由config生成opts
	* 实例化provisioner对象，并放到opts中
	* 实例化AsynchronousServiceGateway对象，该类间接继承自Sinatra::Base类，通过Thin服务器监听HTTP请求, 主要是CC的请求
	* 

* 第3点是`service_gateway`的核心代码，`asynchronous_service_gateway`主要进行如下工作：

	* 处理opts参数
	* 实例化CatalogManagerV2对象(针对V2版本)
	* 通过event_machine方法定时调用`send_heartbeat`, 向CC发布类似于心跳的信息，比如第一次时写入service/service_plan信息或者更新service信息
	* 通过event_machine方法定时调用`fetch_handles`, 来定时从CC处获取关于该gateway负责的service信息, 随时更新以保证不会漏掉任何service信息。当然此处还会定时检查orphan(孤儿)。

* 第4点是`asynchronous_service_gateway`的父类，主要是对sinatra框架进行自定义设置，增加相关helper方法，并对任何到来的请求进行验证

* 第5点

从CC(cloud_controller)的角度来查看mysql_gateway启动时的动作
`mysql_gateway`连接cloud_controller并注册自己的信息。信息主要是两条一条是向cloud_controller的service表插入一条mysql服务的记录, 一条是向service_plans表插入一条plan记录

### 2.2. mysql_node的启动

* `mysql_service/lib/mysql_service/node.rb` 在初始化时

		@mysql_config = options[:mysql]
		@supported_versions = options[:supported_versions]

* `ProvisionedService` 是一个数据结构, 位于`mysql_service/lib/mysql_service/node.rb` 814行, 主要包含这几个属性

		property :name, String, :key => true
		property :user, String, :required => true
		property :password, String, :required => true
		property :plan, Integer, :required => true
		property :quota_exceeded, Boolean, :default => false
		property :version, String

* `ProvisionedService` 的创建源于gateway发过来的请求, 大致流程是接收gateway的ProvisionRequest请求, 然后将其msg参数转换, 通过create等操作同ProvisionedService交互。 ProvisionRequest位于`vcap-services-base/lib/base/service_message.rb`第8行, 包含：

		required :plan, String
		optional :credentials
		optional :version

* 对于创建service的整体流程大致为：node节点在`/vcap-services-base/lib/base/base.rb`初始化时调用`/vcap-services-baselib/base/node.rb`文件中的`on_connect_node`方法, 然后触发很多监听事件, 如on_provision等;
on_provision事件在监听到`ProvisionRequest`后，解析请求发过来的msg参数，然后调用`mysql_service/lib/mysql_service/node.rb`的`provision`方法，创建`ProvisionService`对象, 至此服务创建成功

* `mysql_service/lib/mysql_service/node.rb`的`provision`方法与version有关：

		raise ServiceError.new(ServiceError::UNSUPPORTED_VERSION, version) unless @supported_versions.include?(version)

		provisioned_service = mysqlProvisionedService.create(new_port(credential["port"]), name, user, password, version)

* `mysql_service/lib/mysql_service/node.rb`的第593行

		config = @mysql_configs[instance.version]

此处的instance为一个`ProvisionedService`对象, 因此他的version为来自gateway的数据, 而@mysql_configs是从node的配置文件中得到的数据, @mysql_config到mysql那一层，其内层均包含在内部,具体如下

		mysql:
			"mysql-5.5":
				host: localhost
				port: 3306
				socket: /var/run/mysqld/mysqld.sock
				user: root
				pass: mysql 
				mysqldump_bin: /usr/bin/mysqldump
				mysql_bin: /usr/bin/mysql

### 2.3. 创建一个service

首先利用cf客户端执行命令：

			cf create-service

cf调用create-service命令时先到CC的`model/service_instance`下, 然后先调用plan.service.service_token查看token是否合理


asynchronous_service_gateway的119行的auth_token, 参见其父类base_async_gateway的76行, 该函数返回HTTP请求头的`HTTP_X_VCAP_Service_Token`字段



## 三.参考文献

1. Cloud Foundry Service Gateway源码分析 - http://blog.csdn.net/shlazww/article/details/8112874
2. Cloud Foundry中Service Gateway功能以及通信机制 - http://blog.csdn.net/shlazww/article/details/8670233
3. Services docs - http://docs.cloudfoundry.com/docs/running/architecture/services/