---
layout: post
categories: 云计算
tags: mysql_service token cloudfoundry
---

description: 本文主要描述了mysql\_gateway的token流程, 主要是尝试将cloud\_controller与service\_gateway之间通信用到的token按流程进行分析

### 综述

本文主要描述了mysql_gateway的token流程, 主要是尝试将cloud_controller与service_gateway之间通信用到的token按流程进行分析。

### 前面的话

本文章主要源自于项目组在向`Cloud Foundry V2`版本移植的大进程中, 在`msyql service`移植上遇到的一些问题总结。

由于V2版本更新迭代过快, 先列出我阅读的代码版本信息, 格式为 [项目名] (下载地址) - 版本号(github上提交版本前6位)：

  * [cloud_controller_ng] (https://github.com/cloudfoundry/cloud_controller_ng) - d7e51a
  * [service-release] (https://github.com/cloudfoundry/cf-services-release) - 15d8f2
  * [service-base] (https://github.com/cloudfoundry/vcap-services-base) - ff9932
  * [vcap-common] (https://github.com/cloudfoundry/vcap-common) - f7653c

代码描述难免含糊不清, 而大段的源代码截取又会让文章显得过于冗长。因此，我定义了代码描述的关键点，其格式为：

  [项目名] - `简写文件路径` - 关键调用处 : 行数

### mysql_gateway配置文件中的token

通过config和opts层层转包, 在AsynchronousServiceGateway类中转化为@token, 关键点如下：

      * [vcap-services-base] - `lib/base/gateway` - Services::Base::Gateway.parse_gateway_config : 136 - 156行
      * [vcap-services-base] - `lib/base/gateway` - Services::AsynchronousServiceGateway.setup : 26行

通过读取配置文件中的`service_auth_tokens`域, 转为config[:token] -> opts[:token] -> AsynchronousServiceGateway::@token

### CC请求时进行token验证

每次CC的HTTP请求到达mysql_gateway时都必须对请求进行验证，关键点如下：

      * [vcap-services-base] - `lib/base/base_async_gateway` - VCAP::Services::BaseAsynchronousServiceGateway < Sinatra::Base.before : 44行

      * [vcap-services-base] - `lib/base/base_async_gateway` - VCAP::Services::AsynchronousServiceGateway < BaseAsynchronousServiceGateway.validate_incoming_request : 113行

      * 在validate_incoming_request方法内的119行

          unless auth_token && (auth_token == @token)

`auth_token`参见其父类`base_async_gateway`的76行, 解析后该函数返回HTTP请求头的`HTTP_X_VCAP_Service_Token`字段, 具体如下：

      def auth_token
        @auth_token ||= request_header(VCAP::Services::Api::GATEWAY_TOKEN_HEADER)
        @auth_token
      end

      # /vcap-common-f7653c1140e6/lib/services/api/const.rb第4行
      GATEWAY_TOKEN_HEADER = 'X-VCAP-Service-Token'

### CC请求中的token生成

根据前面知道, CC在向gateway发送请求时需要在请求头中带上token值, 而该token值是CC从数据库中获取得到, 然后在创建HTTP请求时，将token塞到HTTP头中，并发送给gateway。这个流程的关键点如下：

      * [cloud_controller_ng] - `lib/cloud_controller/models/managed_service_instance` - CloudController::Models::NGServiceGatewayClient.service_gateway_client - 169行
      * [vcap-common] `lib/services/api/clients/service_gateway_client.rb` Services::Api::ServiceGatewayClient.initialize - 62行
      * [vcap-common] `lib/services/api/async_requests.rb.rb` Services::Api::AsyncHttpRequest.new - 19行

### 数据库中的token生成

service的token数据位于Postgres数据库服务的cloud_controller数据库的service_auth_tokens数据表。一般每个service都会在该表中创建唯一的一条记录。正常的创建方式为通过cf客户端调用create-auth-token方法创建。

数据库表结构如下

      id        integer
      guid      text
      created_at    timestamp
      updated_at    timestamp
      label     citext      # mysql服务为mysql
      provider    citext      # mysql服务为core
      token       text        # 我的数据记录为6kFfnLwOa0nVS0edoRCoGw==+|5beb4679
      salt      text

#### 1 通过cf客户端创建

      cf target
      cf login
      cf create-service-auth-token --label mysql --provider core --token '0xdeadbeef'

#### 2 手动创建 (已废)

手动创建的方式是在发现cf客户端创建之前, 通过阅读源代码进行的一种尝试, 需要修改CC的源代码才能成功创建服务。因为此处的token值为原文，没有生成密文, CC无法解密。

  向cloud_controller库的servie_auth_token插入一条记录

      insert into service_auth_tokens(guid,created_time,name,label,token,salt) values(1,'20130706','mysql','mysql','0xdeadbeef','deadbeef');

  * 修改CC源码 `model/service_auth_token.rb`的token方法第29行

      def token
        return super
        #return unless super
        VCAP::CloudController::Encryptor.decrypt(super, salt)
      end
