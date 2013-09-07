micro-paas在CentOS 6上安装
=====

参考资料

* [有道CF组的CentOS 6安装文档](https://github.com/youdao-cf/docs/tree/master/install)

#### 1.1 nats安装

build.rb步骤

	* 安装ruby   # 可以先尝试yum安装  而后尝试编译安装, 具体可以参考[网址](https://github.com/limengyun2008/vcap/blob/yae/dev_setup/cookbooks/ruby/libraries/ruby_install.rb)
	
	* git clone http://git.ebcloud.com/paas-cf-v2/nats.git  # 自行寻找详细地址
	* git checkout  #固定版本
	* git submodule update --init   # 进行子模块更新

control.rb步骤

	* 生成nats的配置文件

	* ./bin/nats-server -d -c router.yml   # 执行启动命令 -d命令是启动debug 


#### 1.2 gorouter安装

build.rb步骤

	* go语言环境    # 推荐寻找yum安装的方式
	
	* git clone http://git.ebcloud.com/paas-cf-v2/gorouter.git
	* git checkout  #固定版本
	* git submodule update --init   # 进行子模块更新
	
	* ./bin/go install router/router   # router构建

control.rb步骤

	* 修改router的配置文件

	* ./bin/router -c router.yml   # 执行启动命令 

#### 1.3 uaa安装

build.rb步骤

	* 安装java和maven环境

	* 安装postgres数据库    # 此处不推荐用yum安装, 安装的版本可能不对, 版本务必统一为9.2.4, 最好编译安装, 如果遇到问题再一起研究
	* 编译citext拓展  # 这个CC数据库需要用到
	
	* git clone http://git.ebcloud.com/paas-cf-v2/uaa.git
	* git checkout  #固定版本
	* git submodule update --init   # 进行子模块更新

	* 删除bin/uaa里面的"-P vcap"这一块，是对uaa启动代码进行hack
	* 调整maven repo的位置

control.rb步骤

	* 判断是否第一次执行 
		如果是 创建uaa数据库 并创建访问uaa数据库的root角色
	
	* 修改uaa的配置文件

	* bin/uaa   # 执行启动命令 


#### 1.4 cloud_controller安装

build.rb步骤
	
	* git clone http://git.ebcloud.com/paas-cf-v2/cloud_controller_ng.git
	* git checkout  #固定版本
	* git submodule update --init   # 进行子模块更新

	* 安装基础库，主要包括libxml2-dev libmysql libsqlite libpq 它们分别是xml/mysql/sqlte/postgres的ruby支持
	不过在redhat下面的支持库不是以lib为前缀， 尝试执行
	sudo yum -y install mysql-devel postgresql-devel sqlite-devel

	* bundle install 

control.rb步骤

	* 判断是否第一次执行 
		如果是 创建CC数据库 并创建访问CC数据库的root角色
	
	* 修改cloud_controller的配置文件

	* bin/cloud_controller   # 执行启动命令 

#### 1.5 health_manager安装

build.rb步骤
	
	* git clone http://git.ebcloud.com/paas-cf-v2/health_manager.git
	* git checkout  # 固定版本

	* bundle install 

control.rb步骤
	
	* 修改health_manager的配置文件

	* 设置BUNDLE_GEMFILE的env
	* ./health_manager   # 执行启动命令 health_manager必须要到bin目录下执行, 不能通过bin/health_manager执行
 
#### 1.6 dea安装

build.rb步骤

	* 先安装warden 参考之前的ubuntu下的build和control   TODO  考虑平台问题debootstrap quota apparmor-profiles在CentOS下是什么？具体请参考[网址](https://github.com/youdao-cf/docs/blob/master/install/warden.html.md)
	
	* git clone http://git.ebcloud.com/paas-cf-v2/dea_ng.git
	* git checkout  #固定版本
	* git submodule update --init   # 进行子模块更新

	* bundle install  # 如果带false参数 表示当前项目下的Gemfile中的test/production等group不会被调用

	* go/bin/go get launchpad.net/goyaml   # 这一步是干什么

	* bundle exec rake dir_server:install  # 安装directory_server

	* 修改buildpack获取语言包的位置

control.rb步骤
	
	* 修改dea的配置文件

	* bin/dea config_file   # 之前需要先启动warden directory_server

#### 1.7 service安装

build.rb步骤

	* 预先安装mysql    # 可以先用yum安装, 再编译安装  不过得注意版本统一
	* 预先安装redis    # 编译安装即可
	
	* git clone http://git.ebcloud.com/paas-cf-v2/cf-services-release.git
	* git checkout  # 固定版本
	* git submodule update --init  # 进行子模块更新

	* bundle install 

control.rb步骤
	
	* 修改mysql_gateway 和 mysql_node 的配置文件

	* bin/mysql_gateway config_file
	* bin/mysql_node config_file