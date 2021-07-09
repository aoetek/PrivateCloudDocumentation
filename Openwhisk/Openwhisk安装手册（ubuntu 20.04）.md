# Openwhisk安装手册（ubuntu 20.04）

[官方文档](https://github.com/apache/openwhisk-devtools/blob/master/docker-compose/README.md)

### 1. 环境

#### 1.1 更新apt

```
$ sudo apt update
```



```
$ sudo apt upgrade
```

#### 1.2 安装java

```
$ sudo apt install openjdk-11-jdk
```

#### 1.3 安装nodejs LTS 和 npm

[官方地址](https://nodejs.org/en/download/) [官方文档](https://github.com/nodesource/distributions/blob/master/README.md)

##### 1.3.1 安装

```
# Using Ubuntu
$ curl -sL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
$ sudo apt-get install -y nodejs
```

##### 1.3.2 验证

```
$ node -v

$ npm -v
```

#### 1.4 安装zip

```
$ sudo apt install zip
```



#### 1.5安装docker

[官方手册](https://docs.docker.com/engine/install/ubuntu/)

##### 1.5.1 删除老版本docker 

(package not found是正常的) 

```
$ sudo apt remove docker docker-engine docker.io containerd runc
```

##### 1.5.2 安装所需要的工具

```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

##### 1.5.3 安装官方的GPG key

```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

##### 1.5.4 验证key安装

```
$ sudo apt-key fingerprint 0EBFCD88

pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

或者用下面命令显示所有key并自行找到上述的key：

```
$ sudo apt-key list
```

##### 1.5.5 添加stable库

```
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

##### 1.5.6 更新apt 并安装docker

```
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd.io
```

##### 1.5.7 Docker 镜像配置(可选)

上aliyun找容器镜像服务，在镜像工具->镜像加速器中找到自己的加速器地址

```
$ sudo mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxxxx.mirror.aliyuncs.com"]
}
EOF

$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

##### 1.5.8 验证安装

```
$ sudo docker pull hello-wolrd
$ sudo docker run hello-wolrd
```

#### 1.6 安装docker-compose

[官方文档](https://docs.docker.com/compose/install/)

##### 1.6.1 下载可执行文件

```
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-Linux-x86_64" -o /usr/local/bin/docker-compose
```

地址中的1.28.2为版本号，最后的docker-compose-Linux-x86_64是使用的系统版本，如有需要请自行更改

##### 1.6.2 给可执行文件添加权限

```
$ sudo chmod +x /usr/local/bin/docker-compose
```

##### 1.6.3 验证安装

```
$ docker-compose -version
```

#### 1.7 安装git(阿里云默认系统有)

##### 1.7.1 安装git

```
$ sudo apt install git
```

配置git代理（可选）

```
# 设置代理
$ git config --global http.proxy 'socks5://127.0.0.1:1080' 
$ git config --global https.proxy 'socks5://127.0.0.1:1080'
# 查看代理
$ git config --global --get http.proxy
$ git config --global --get https.proxy
# 取消代理
$ git config --global --unset http.proxy
$ git config --global --unset https.proxy

```

##### 1.7.2 验证安装

```
$ git --version
```

#### 1.8 安装make（阿里云默认系统有）

```
$ sudo apt install make
```

### 2. 下载Openwhisk docker compose部署工具相关文件

#### 2.1 创建项目文件夹

```
$ mkdir ~/Openwhisk
$ cd ~/Openwhisk
```

#### 2.2 下载Openwhisk核心源码

```
$ git clone https://github.com/apache/openwhisk.git
```

#### 2.3 下载Openwhisk-devtools

```
$ git clone https://github.com/apache/openwhisk-devtools.git
```

#### 2.4 下载Openwhisk-cli

##### 2.4.1 下载

请访问 [Openwhisk-cli Release Page](https://github.com/apache/openwhisk-cli/releases) 来找到想要的版本地址，这里我使用1.1.0

```
$ wget https://github.com/apache/openwhisk-cli/releases/download/1.2.0/OpenWhisk_CLI-1.2.0-linux-amd64.tgz
```

##### 2.4.2 解压拷贝

```
# 创建一个CLI的文件夹
$ mkdir openwhisk-cli

# 解压
$ tar zxvf OpenWhisk_CLI-1.2.0-linux-amd64.tgz -C ./openwhisk-cli

# 拷贝
$ cp openwhisk-cli/wsk openwhisk/bin/wsk
```

##### 2.4.3 清理文件

```
$ rm OpenWhisk_CLI-1.2.0-linux-amd64.tgz
```

##### 2.4.4 验证安装

```
$ ./openwhisk/bin/wsk
```

#### 2.5 下载Openwhisk-catalog

```
$ git clone https://github.com/apache/openwhisk-catalog.git
```
#### 2.6 项目结构

全部下载完成后项目结构大概是这样

```
~/Openwhisk
	--openwhisk							//openwhisk 核心
	--openwhisk-cli						//cli 工具
	--openwhisk-catalog			
	--openwhisk-devtools
		--docker-compose
			--apigateway				//api 工具
			--Makefile					//make编译文件
			--docker-compose.yml		//docker compose 运行配置文件
			...
        ...
```

### 3 用Docker Compose 部署Openwhisk 

#### 3.1 配置创建Makefile 配置文件

```
# 进入Makefile文件所在的目录
$ cd ~/Openwhisk/openwhisk-devtools/docker-compose/
# 创建make.props文件
$ vim make.props

#将如下内容输入make.props 保存
#建议将WSK_CLI 的~手动替换成用户目录，即WSK_CLI设置地址时使用全路径
#实际上 ~ 就等于 用户目录 
#但是如果打 ~ Makefile文件中download-cli部分就认为你没设置cli地址，要重新下载

OPENWHISK_PROJECT_HOME=~/Openwhisk/openwhisk
OPENWHISK_CATALOG_HOME=~/Openwhisk/openwhisk-catalog
WSK_CLI=~/Openwhisk/openwhisk-cli/wsk
```

#### 3.2 创建bash dash 连接

Ubuntu中/bin/sh链接默认指向的是dash shell，make 命令里要使用bash

```
$ sudo ln -sf /bin/bash /bin/sh
```



#### 3.2 部署Openwhisk

```
$ cd ~/Openwhisk/openwhisk-devtools/docker-compose/
$ sudo make $(cat ./make.props) quick-start
```



当你看到hello world 运行了，就完成了：

```
Installing apimgmt actions
ok: updated action apimgmt/getApi
ok: updated action apimgmt/createApi
ok: updated action apimgmt/deleteApi
waiting for the Whisk invoker to come up ...
 ... OK
creating the hello.js function ...
invoking the hello-world function ...
adding the function to whisk ...
ok: created action hello
invoking the function ...
invocation result: { "payload": "Hello, World!" }
{ "payload": "Hello, World!" }
creating an API from the hello function ...
ok: updated action hello
invoking:  http://xxx.xxx.xxx.xxx:xxx/api/xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/hello/world
  "payload": "Hello, World!"
ok: APIs
Action          Verb  API Name  URL
/guest/hello     get    /hello  http://xxx.xxx.xxx.xxx:xxx/api/xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/hello/world
deleting the API ...
ok: deleted API /hello
deleting the function ...
ok: deleted action hello
To invoke the function again use: make hello-world
To stop OpenWhisk use: make destroy
To use the wsk CLI: export WSK_CONFIG_FILE=~/Openwhisk/openwhisk-devtools/docker-compose/.wskprops
                    or copy the file to /root/.wskprops
```



### 4 wsk配置

[WSK官方配置文档](https://github.com/apache/openwhisk/blob/master/docs/cli.md#openwhisk-cli)

wsk 配置文件会提示在最后

```
To stop OpenWhisk use: sudo make destroy
To use the wsk CLI: export WSK_CONFIG_FILE=~/Openwhisk/openwhisk-devtools/docker-compose/.wskprops
                    or copy the file to /root/.wskprops
```

然后使用如下命令将.wskprops中的properties 设置好(若有问题请尝试加上sudo)

```
$ cd ~/Openwhisk/openwhisk-cli/
$ wsk property set --auth [文件中的AUTH]
$ wsk property set --apihost [文件中的APIHOST]
```

测试

```
$ wsk -i action list
```

