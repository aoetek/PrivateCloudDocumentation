### OpenStack常用服务

##### 认证管理服务			keystone

从架构角度来看，Keystone提供了OpenStack体系中最简单的服务。它是OpenStack的核心组件之一，提供身份认证服务，包括OpenStack中租户的身份验证和授权。不同OpenStack服务之间的通信都必须经过Keystone认证，以确保授权的用户或服务能够访问所请求的OpenStack服务对象。Keystone集成了许多身份验证机制，如基于用户名/密码和令牌/身份的验证系统。此外，可以将其与现有后端集成，例如轻量级目录访问协议（Lightweight Directory Access Protocol，LDAP）和插拔式验证模块（Pluggable Authentication Module，PAM）。



##### 对象存储服务			swift

Swift是OpenStack用户可以使用的一种存储服务。它通过REST API提供对象存储功能。与传统存储解决方案（如文件共享存储或基于块的存储）相比，对象存储采用对象方式来处理所存储的数据，并从对象存储中存储和检索对象。我们来抽象地概况一下对象存储。为了存储数据，对象存储将数据拆分为较小的块并将其存储在独立容器中。这些保存数据的容器分布在存储集群节点上，并拥有冗余副本，以提供高可用性、自动恢复能力和水平可伸缩性。 

稍后我们会讨论Swift对象存储的架构。简单来说，对象存储有以下几个优点： 

去中心化，无 单点故障 （Single Point of Failure，SPOF）。 

具有自愈能力，这意味着发生故障时能自动恢复。 

通过水平扩展可以将存储空间扩展到PB级以上。 

具有高性能，通过在多个存储节点上分散负载实现。 

支持廉价硬件，这些硬件可用于冗余存储集群。



##### 块存储服务				cinder



##### 文件共享存储服务	manila



##### 镜像注册服务			glance



##### 计算服务					Nova



##### 网络服务					neutron



##### 计量服务					telemetry



##### 编排服务					heat



##### 仪表盘服务				horizon



##### 消息队列



##### 数据库



##### OpenStack各服务关系

![image-20210429104706658](openstack部署log.assets/image-20210429104706658-1625801833060.png)

##### OpenStack的服务图表

![openstack-map-v20210201](openstack部署log.assets/openstack-map-v20210201.svg)

### OpenStack安装与部署



#### 环境配置

##### 1.准备两个linux系统

我将以ubuntu 20.04 LTS为例，

分别作为controller(例子中叫做controller，如果你的主机名不叫这个，请在后续的操作将'controller'替换为你的主机名)

和compute(例子中叫做compute，如果你的主机名不叫这个，请在后续的操作中将'compute'替换为你的主机名)



##### 2.配置两个node的网络

- controller上：

  将ip address，network mask，default gateway，配置好

  编辑interface文件

  ```shell
  $ vim /etc/network/interfaces
  # 将INTERFACE_NAME替换为你当前系统的默认网卡，一般为ethxxx或者ensxxx
  auto INTERFACE_NAME 
  iface INTERFACE_NAME inet manual 
  up ip link set dev $IFACE up 
  down ip link set dev $IFACE down
  
  ```

  编辑hosts文件

  ```shell
  $ vim /etc/hosts
  # controller 
  <默认网卡的ip address> controller 
  
  # compute
  
  <默认网卡的ip address> compute
  
  #········
  ···
  ···
  ···
  ···
  
  ```

  

- 其它node上：

  将ip address，network mask，default gateway，配置好,需要注意的是，其它node的ip和controller要在同一网段

  编辑interface文件

  ```shell
  $ vim /etc/network/interfaces
  # 将INTERFACE_NAME替换为你当前系统的默认网卡，一般为ethxxx或者ensxxx
  auto INTERFACE_NAME 
  iface INTERFACE_NAME inet manual 
  up ip link set dev $IFACE up 
  down ip link set dev $IFACE down
  
  ```

  编辑hosts文件

  ```shell
  $ vim /etc/hosts
  # controller 
  <默认网卡的ip address> controller 
  
  # compute
  
  <默认网卡的ip address> compute
  
  #········
  ···
  ···
  ···
  ···
  
  ```

- 验证连接

  在两个node上相互ping对方node，以及外网。

  controller  node上

  ```shell
  $ ping -c 4 www.baidu.com
  $ ping -c 4 compute
  ```

  compute node 上

  ```shell
  $ ping -c 4 www.baidu.com
  $ ping -c 4 controller
  ```

  

  

##### 3.同步两个node的时间

- 两个系统分别安装chrony

  ```shell
  $ apt install chrony
  ```

- controller编辑chrony.conf文件

  ```shell
  $ vim /etc/chrony.conf
  
  # 加入下面这行,其中NTP_SERVER替换为你自己选择的NTP授时ip
  server NTP_SERVER iburst
  
  # 加入下面这行，其中ip地址替换为你网卡的ip地址
  allow IP_ADDRESS/24
  ```

  退出编辑并重启服务

  ```shell
  $ service chrony restart
  ```

- compute编辑chrony.conf文件

  ```shell
  $ vim /etc/chrony.conf
  
  # 加入下面这行,其中NTP_SERVER替换为你自己选择的NTP授时ip
  server controller iburst
  
  # 加入下面这行，其中ip地址替换为你网卡的ip地址
  allow IP_ADDRESS/24
  ```

  退出编辑并重启服务

  ```shell
  $ service chrony restart
  ```

- 验证操作，两个node都需要验证。

  ```shell
  $ chronyc sources
  ```

  

##### 4.安装OpenStack

我选择的是ubuntu 20.04 LTS，所以安装的是openstack victoria版本

所有node都需要安装

```shell
$ add-apt-repository cloud-archive:victoria
$ apt update && apt dist-upgrade
$ apt install python3-openstackclient
```





##### 5.安装数据库

我选择的是MariaDB ，完全兼容mysql

只需要在controller node上安装

```shell
$ apt install mariadb-server python3-pymysql

# 创建并编辑配置文件,IP_ADDRESS替换为你的网卡ip

$ vim /etc/mysql/mariadb.conf.d/99-openstack.cnf

[mysqld]
bind-address = IP_ADDRESS

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

# 退出编辑并且重启服务
$ service mysql restart
# 设定一个root密码
$ mysql_secure_installation
```





##### 6.安装消息队列MQ

我这里选择的是RabbitMQ，只需要在controller node 上安装

```shell
$ apt install rabbitmq-server
# 添加openstack用户,其中RABBIT_PASS是你自己设定一个密码
$ rabbitmqctl add_user openstack RABBIT_PASS
# 授予读写权限
$ rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```





##### 7.安装 memcahed

安装在controller node 上

```shell
$ apt install memcached python-memcache

# 编辑配置文件,添加下行，IP_ADDRESS替换为你controller的默认网卡ip，如果已经有了-l 这行，就注释掉
$ vim /etc/memcached.conf
-l IP_ADDRESS

# 退出编辑并重启服务
$ service memcached restart
```





##### 8.安装etcd

安装至controller node 上

```shell
$ apt install etcd

# 编辑配置文件，找到并修改下列参数,IP_ADDRESS替换为网卡的默认ip
$ vim /etc/default/etcd

ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://IP_ADDRESS:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://IP_ADDRESS:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://IP_ADDRESS:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://IP_ADDRESS:2379"

# 激活并重启etcd服务
$ systemctl enable etcd
$ systemctl restart etcd
```





#### 安装服务

##### 1.认证管理服务keystone

安装至controller node

```shell
# 创建数据库
$ mysql
$ CREATE DATABASE keystone;

# 授权该库，并且将KEYSTONE_DBPASS替换为你自己喜欢的密码
$ GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
$ GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';

# 退出数据库，开始安装keystone
$ apt install keystone

# 编辑配置文件
$ vim /etc/keystone/keystone.conf

# 找到[database]这一项，并修改如下，其中KEYSTONE_DBPASS替换为刚刚你设置的密码
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

# 找到[token]这一项，并修改如下：
[token]
# ...
provider = fernet

# 退出编辑并填充认证服务数据库
$ su -s /bin/sh -c "keystone-manage db_sync" keystone

# 初始化秘钥库
$ keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
$ keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

# Bootstrap the Identity service,将ADMIN_PASS设定为你喜欢的密码
$ keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

# 配置Apache http server,编辑配置文件
$ vim /etc/apache2/apache2.conf

# 加入下行
ServerName controller

# 退出编辑，并重启服务
$ service apache2 restart

# 创建并编辑配置文件,填入下列参数，ADMIN_PASS为你刚刚在bootstrap操作中设定的密码
$ cd
$ vim admin-openrc

export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

创建实例练习

[参考官方手册](https://docs.openstack.org/keystone/victoria/install/keystone-users-ubuntu.html)

............

............



##### 2.镜像注册服务glance

安装至controller node，需要确保有mysql服务



- 创建glance服务凭证

```shell
# 使用root账号进入mysql
$ mysql

# 创建glance数据库
CREATE DATABASE glance;

# 授权，并退出mysql
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
  
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';

# 获取admin的凭据，来获取cli命令
$ . admin-openrc

# 接下来是创建服务凭证
# 先创建glance用户，密码填入你喜欢的

$ openstack user create --domain default --password-prompt glance


User Password:
Repeat User Password:

+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3f4e777c4062483ab8d9edd7dff829df |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

# 然后添加admin角色到glance用户,admin这个project即为你在admin-openrc中配置的。
$ openstack role add --project admin --user glance admin

# 最后创建glance服务
$ openstack service create --name glance \
  --description "OpenStack Image" image


+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

# 创建镜像服务api端口
$ openstack endpoint create --region RegionOne \
>   image public http://controller:9292


+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | adcdbea6b16249bfb324ed4cafa1a76f |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 04f86d10743e432e8378dec21a6b14d6 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a6e4b153c2ae4c919eccfdbb7dceb5d2 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+


$ openstack endpoint create --region RegionOne \
  image admin http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0c37ed58103f4300a84ff125a539032d |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

```



- 安装，配置glance

```shell
# 安装glance
$ apt install glance

# 编辑配置文件，分别找到以下选项并修改参数如下，controller就是你的主机名，GLANCE_PASS修改为你之前设置的密码,修改完毕后保存并退出编辑。

$ vim /etc/glance/glance-api.conf

[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/


# 添加镜像服务数据库,如果出现大量报错，先检查授权是否执行过。
$ su -s /bin/sh -c "glance-manage db_sync" glance

# 重启镜像服务
$ service glance-api restart
```



- 验证操作

  ```shell
  # 获取admin凭据来访问管理员cli命令
  $ . admin-openrc
  
  # 下载镜像
  $ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
  
  # 上传镜像(QCOW2磁盘格式，bare容器格式，public权限，这样所有的project都可以访问)、如果提升没有权限或者internal server error应该是授权没成功，可以再授权一次
  $ glance image-create --name "cirros" \
    --file cirros-0.4.0-x86_64-disk.img \
    --disk-format qcow2 --container-format bare \
    --visibility=public
    
  +------------------+----------------------------------------------------------------------------------+
  | Property         | Value                                                                            |
  +------------------+----------------------------------------------------------------------------------+
  | checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                 |
  | container_format | bare                                                                             |
  | created_at       | 2021-05-10T06:31:45Z                                                             |
  | disk_format      | qcow2                                                                            |
  | id               | 8f66cd19-18a2-49ed-a879-ef4b76a9c14e                                             |
  | min_disk         | 0                                                                                |
  | min_ram          | 0                                                                                |
  | name             | cirros                                                                           |
  | os_hash_algo     | sha512                                                                           |
  | os_hash_value    | 6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e |
  |                  | 2161b5b5186106570c17a9e58b64dd39390617cd5a350f78                                 |
  | os_hidden        | False                                                                            |
  | owner            | 5bd1f3d7d0e04a8dbf07fd5365074f66                                                 |
  | protected        | False                                                                            |
  | size             | 12716032                                                                         |
  | status           | active                                                                           |
  | tags             | []                                                                               |
  | updated_at       | 2021-05-10T06:31:45Z                                                             |
  | virtual_size     | 46137344                                                                         |
  | visibility       | public                                                                           |
  +------------------+----------------------------------------------------------------------------------+
  
  # 查看镜像仓库
  $ glance image-list
  +--------------------------------------+--------+
  | ID                                   | Name   |
  +--------------------------------------+--------+
  | 8f66cd19-18a2-49ed-a879-ef4b76a9c14e | cirros |
  +--------------------------------------+--------+
  
  
  ```

  

##### 3.资源布局服务Placement

​	安装至controller node上

- 建立数据库

  ```shell
  # 进入数据库
  $ mysql	
  
  # 创建placement数据库
  MariaDB [(none)]> CREATE DATABASE placement;
  
  # 授权，将PLACEMENT_DBPASS改成你自己喜欢的密码，完成之后退出数据库界面
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
    IDENTIFIED BY 'PLACEMENT_DBPASS';
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
    IDENTIFIED BY 'PLACEMENT_DBPASS';
    
  
  ```

- 配置用户以及端口

  ```shell
  # 取得admin的权限cli命令
  $ . admin-openrc
  
  # 创建placement服务用户，填入自己喜欢的密码
  $ openstack user create --domain default --password-prompt placement
  
  User Password:7729787
  Repeat User Password:7729787
  +---------------------+----------------------------------+
  | Field               | Value                            |
  +---------------------+----------------------------------+
  | domain_id           | default                          |
  | enabled             | True                             |
  | id                  | fa742015a6494a949f67629884fc7ec8 |
  | name                | placement                        |
  | options             | {}                               |
  | password_expires_at | None                             |
  +---------------------+----------------------------------+
  
  
  # 将placement用户添加到service（如果你的项目名称不是这个，可以修改为你自己的）项目中
  $ openstack role add --project service --user placement admin
  
  # 在服务目录中添加 api接口
  $ openstack service create --name placement \
    --description "Placement API" placement
  
  +-------------+----------------------------------+
  | Field       | Value                            |
  +-------------+----------------------------------+
  | description | Placement API                    |
  | enabled     | True                             |
  | id          | 2d1a27022e6e4185b86adac4444c495f |
  | name        | placement                        |
  | type        | placement                        |
  +-------------+----------------------------------+
  
  # 创建placement服务的api端口
  $ openstack endpoint create --region RegionOne \
    placement public http://controller:8778
  
  +--------------+----------------------------------+
  | Field        | Value                            |
  +--------------+----------------------------------+
  | enabled      | True                             |
  | id           | 2b1b2637908b4137a9c2e0470487cbc0 |
  | interface    | public                           |
  | region       | RegionOne                        |
  | region_id    | RegionOne                        |
  | service_id   | 2d1a27022e6e4185b86adac4444c495f |
  | service_name | placement                        |
  | service_type | placement                        |
  | url          | http://controller:8778           |
  +--------------+----------------------------------+
  
  $ openstack endpoint create --region RegionOne \
    placement internal http://controller:8778
  
  +--------------+----------------------------------+
  | Field        | Value                            |
  +--------------+----------------------------------+
  | enabled      | True                             |
  | id           | 02bcda9a150a4bd7993ff4879df971ab |
  | interface    | internal                         |
  | region       | RegionOne                        |
  | region_id    | RegionOne                        |
  | service_id   | 2d1a27022e6e4185b86adac4444c495f |
  | service_name | placement                        |
  | service_type | placement                        |
  | url          | http://controller:8778           |
  +--------------+----------------------------------+
  
  $ openstack endpoint create --region RegionOne \
    placement admin http://controller:8778
  
  +--------------+----------------------------------+
  | Field        | Value                            |
  +--------------+----------------------------------+
  | enabled      | True                             |
  | id           | 3d71177b9e0f406f98cbff198d74b182 |
  | interface    | admin                            |
  | region       | RegionOne                        |
  | region_id    | RegionOne                        |
  | service_id   | 2d1a27022e6e4185b86adac4444c495f |
  | service_name | placement                        |
  | service_type | placement                        |
  | url          | http://controller:8778           |
  +--------------+----------------------------------+
  ```

- 安装和配置

  ```shell
  # 安装placement
  $ apt install placement-api
  
  # 编辑配置文件 
  $ vim /etc/placement/placement.conf
  
  # 找到并修改下列参数，PLACEMENT_DBPASS修改为你的placement数据库密码
  [placement_database]
  # ...
  connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
  
  # PLACEMENT_PASS 修改为你之前的placement认证密码，keystone_authtoken中请把其它无关的全部注释
  [api]
  # ...
  auth_strategy = keystone
  
  [keystone_authtoken]
  # ...
  auth_url = http://controller:5000/v3
  memcached_servers = controller:11211
  auth_type = password
  project_domain_name = Default
  user_domain_name = Default
  project_name = service
  username = placement
  password = PLACEMENT_PASS
  
  # 退出编辑并且填充placement数据库
  $ su -s /bin/sh -c "placement-manage db sync" placement
  
  
  ```



- 完成安装

  ```shell
  # 重启服务
  $ service apache2 restart
  ```

- 验证安装

  ```shell
  # 获取admin权限cli命令
  $ . admin-openrc
  
  # 查看状态
  $ placement-status upgrade check
  
  # 运行一些例子
  # 安装osc-placement
  $ pip3 install osc-placement
  
  # 列出可用资源
  $ openstack --os-placement-api-version 1.2 resource class list --sort-column name
  # 如果有报错，则是因为还没安装osc-placement
  +----------------------------+
  | name                       |
  +----------------------------+
  | DISK_GB                    |
  | IPV4_ADDRESS               |
  | ...                        |
  
  $ openstack --os-placement-api-version 1.6 trait list --sort-column name
  +---------------------------------------+
  | name                                  |
  +---------------------------------------+
  | COMPUTE_DEVICE_TAGGING                |
  | COMPUTE_NET_ATTACH_INTERFACE          |
  | ...                                   |
  ```

##### 4.计算服务nova

本项服务需要在各个节点安装，各有区别，需要先安装placement。

###### 1.controller node

- 准备工作

  ```shell
  # 进入mysql
  $ mysql
  
  # 创建数据库
  MariaDB [(none)]> CREATE DATABASE nova_api;
  MariaDB [(none)]> CREATE DATABASE nova;
  MariaDB [(none)]> CREATE DATABASE nova_cell0;
  
  # 授权，然后退出数据库
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
    IDENTIFIED BY '123';
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
    IDENTIFIED BY '123';
  
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
    IDENTIFIED BY '123';
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
    IDENTIFIED BY '123';
  
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
    IDENTIFIED BY '123';
  MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
    IDENTIFIED BY '123';
  
  # 取得admin权限cli命令
  $ . admin-openrc
  
  # 创建计算服务凭证，密码自己填
  # nova
  $ openstack user create --domain default --password-prompt nova
  
  User Password:123	
  Repeat User Password:123
  +---------------------+----------------------------------+
  | Field               | Value                            |
  +---------------------+----------------------------------+
  | domain_id           | default                          |
  | enabled             | True                             |
  | id                  | 8a7dbf5279404537b1c7b86c033620fe |
  | name                | nova                             |
  | options             | {}                               |
  | password_expires_at | None                             |
  +---------------------+----------------------------------+
  
  # 将admin添加到nova用户，命令没有输出
  $ openstack role add --project service --user nova admin
  
  # 创建nova服务
  $ openstack service create --name nova \
    --description "OpenStack Compute" compute
  
  +-------------+----------------------------------+
  | Field       | Value                            |
  +-------------+----------------------------------+
  | description | OpenStack Compute                |
  | enabled     | True                             |
  | id          | 060d59eac51b4594815603d75a00aba2 |
  | name        | nova                             |
  | type        | compute                          |
  +-------------+----------------------------------+
  
  # 创建compute服务api端口
  $ openstack endpoint create --region RegionOne \
    compute public http://controller:8774/v2.1
  
  +--------------+-------------------------------------------+
  | Field        | Value                                     |
  +--------------+-------------------------------------------+
  | enabled      | True                                      |
  | id           | 3c1caa473bfe4390a11e7177894bcc7b          |
  | interface    | public                                    |
  | region       | RegionOne                                 |
  | region_id    | RegionOne                                 |
  | service_id   | 060d59eac51b4594815603d75a00aba2          |
  | service_name | nova                                      |
  | service_type | compute                                   |
  | url          | http://controller:8774/v2.1               |
  +--------------+-------------------------------------------+
  
  $ openstack endpoint create --region RegionOne \
    compute internal http://controller:8774/v2.1
  
  +--------------+-------------------------------------------+
  | Field        | Value                                     |
  +--------------+-------------------------------------------+
  | enabled      | True                                      |
  | id           | e3c918de680746a586eac1f2d9bc10ab          |
  | interface    | internal                                  |
  | region       | RegionOne                                 |
  | region_id    | RegionOne                                 |
  | service_id   | 060d59eac51b4594815603d75a00aba2          |
  | service_name | nova                                      |
  | service_type | compute                                   |
  | url          | http://controller:8774/v2.1               |
  +--------------+-------------------------------------------+
  
  $ openstack endpoint create --region RegionOne \
    compute admin http://controller:8774/v2.1
  
  +--------------+-------------------------------------------+
  | Field        | Value                                     |
  +--------------+-------------------------------------------+
  | enabled      | True                                      |
  | id           | 38f7af91666a47cfb97b4dc790b94424          |
  | interface    | admin                                     |
  | region       | RegionOne                                 |
  | region_id    | RegionOne                                 |
  | service_id   | 060d59eac51b4594815603d75a00aba2          |
  | service_name | nova                                      |
  | service_type | compute                                   |
  | url          | http://controller:8774/v2.1               |
  +--------------+-------------------------------------------+
  ```



- 安装，配置

  ```shell
  # 安装计算服务
  $ apt install nova-api nova-conductor nova-novncproxy nova-scheduler
  
  # 编辑配置文件,修改一下参数，请把相应的密码，参数，ip，node名称等修改为你自己的。
      $ vim /etc/nova/nova.conf
  
  [api_database]
  # ...
  connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api
  
  [database]
  # ...
  connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
  
  [DEFAULT]
  # ...
  transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
  
  [api]
  # ...
  auth_strategy = keystone
  
  [keystone_authtoken]
  # ...
  www_authenticate_uri = http://controller:5000/
  auth_url = http://controller:5000/
  memcached_servers = controller:11211
  auth_type = password
  project_domain_name = Default
  user_domain_name = Default
  project_name = service
  username = nova
  password = NOVA_PASS
  
  [DEFAULT]
  # ...
  my_ip = 10.0.0.11
  
  [vnc]
  enabled = true
  # ...
  server_listen = $my_ip
  server_proxyclient_address = $my_ip
  
  [glance]
  # ...
  api_servers = http://controller:9292
  
  [oslo_concurrency]
  # ...如果出现packaging bug ，请注释掉DEFAULT中的log_dir
  lock_path = /var/lib/nova/tmp
  
  [placement]
  # ...
  region_name = RegionOne
  project_domain_name = Default
  project_name = service
  auth_type = password
  user_domain_name = Default
  auth_url = http://controller:5000/v3
  username = placement
  password = PLACEMENT_PASS
  
  # 填充nova-api数据库
  $ su -s /bin/sh -c "nova-manage api_db sync" nova
  
  # 注册cell0数据库
  $ su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
  
  # 创建cell1单元
  $ su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
  
  # 填充nova数据库
  $ su -s /bin/sh -c "nova-manage db sync" nova
  
  # 验证nova cell0 和cell1 是否正确注册
  $ su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
  ```



- 完成安装

  ```shell
  $ service nova-api restart
  $ service nova-scheduler restart
  $ service nova-conductor restart
  $ service nova-novncproxy restart
  ```

  

###### 2.compute node

- 安装，配置

  ```shell
  # 安装nova-compute
  $ apt install nova-compute
  
  # 编辑配置文件,注意将node名称，相关密码改为自己的
  $ 
  ```



```
openstack --os-auth-url http://openstack_v1:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue
```

```
keystone-manage bootstrap --bootstrap-password 123 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

![image-20210518151604863](openstack部署log.assets/image-20210518151604863-1625801913969.png)

解决方案：扫描可用的计算节点，然后再次验证就解决了

```shell
root@controller:~# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 39584dcc-8769-47d4-91ad-8797af372f8f
Checking host mapping for compute host 'compute': e1ec6a5f-d9f8-41eb-8e23-d3f8cccbd9ec
Creating host mapping for compute host 'compute': e1ec6a5f-d9f8-41eb-8e23-d3f8cccbd9ec
Checking host mapping for compute host 'controller': 95aa7a49-c748-4490-8e08-e0ef7c56424b
Creating host mapping for compute host 'controller': 95aa7a49-c748-4490-8e08-e0ef7c56424b
Found 2 unmapped computes in cell: 39584dcc-8769-47d4-91ad-8797af372f8f



root@controller:~# nova-status upgrade check
+-------------------------------------------+
| Upgrade Check Results                     |
+-------------------------------------------+
| Check: Cells v2                           |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Placement API                      |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Ironic Flavor Migration            |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Cinder API                         |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy Scope-based Defaults        |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Policy File JSON to YAML Migration |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
| Check: Older than N-1 computes            |
| Result: Success                           |
| Details: None                             |
+-------------------------------------------+
```



##### 5.网络服务

###### Configure the Linux bridge agent[¶](https://docs.openstack.org/neutron/ussuri/install/controller-install-option1-ubuntu.html#configure-the-linux-bridge-agent)

The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups.

- Edit the `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` file and complete the following actions:

  - In the `[linux_bridge]` section, map the provider virtual network to the provider physical network interface:

    ```
    [linux_bridge]
    physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
    ```







ensure your Linux operating system kernel supports network bridge filters
by verifying all the following `sysctl` values are set to `1`

```powershell
net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-ip6tables

执行：
[root@controller ~]# vim /etc/sysctl.conf 
[root@controller ~]# sysctl -p
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

[root@controller ~]# lsmod|grep bridge
bridge                151336  1 br_netfilter
stp                    12976  1 bridge
llc                    14552  2 stp,bridge
```

keystone配置

```
keystone-manage bootstrap --bootstrap-password 123 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

镜像下载失败，

##### 6.dashboard错误记录

- 

##### ![image-20210617150702204](openstack部署log.assets/image-20210617150702204.png)

直接注释掉该项参数即可

```shell
#OPENSTACK_NEUTRON_NETWORK = {
#            ...
#                'enable_router': False,
#                    'enable_quotas': False,
#                        'enable_ipv6': False,
#                            'enable_distributed_router': False,
#                                'enable_ha_router': False,
#                                    'enable_lb': False,
#                                        'enable_firewall': False,
#                                            'enable_vpn': False,
#                                                'enable_fip_topology_check': False,
#                                                }

```

- dashboard无法登录，显示invalid credit

  修改配置文件中的此行，把identity改成5000

  ```shell 
  OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
  ```

  

### 搭建完毕测试

#### 可视化界面

![image-20210709092857448](openstack部署log.assets/image-20210709092857448.png)

- 创建网络

![image-20210709113952719](openstack部署log.assets/image-20210709113952719.png)



![image-20210709114008400](openstack部署log.assets/image-20210709114008400.png)



之后在安全组添加规则，此次为测试阶段，均使用ICMP全部通过。



- 创建实例

  ![image-20210709114225088](openstack部署log.assets/image-20210709114225088.png)



实例创建完毕之后

- 启动实例![image-20210709114344007](openstack部署log.assets/image-20210709114344007.png)



### 结语

到目前为止，搭建工作初步成型，之后需要结合其它技术在openstack上继续进行测试和研发。
