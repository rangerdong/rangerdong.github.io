---
title: lnmp开发环境搭建 + mysql主从配置
date: 2017-09-11 00:29:17
tags: php, linux, git
---

现在的开发倾向于往虚拟机上进行开发，为了省去每次搭建环境在网上查找的多余时间，很久就想进行一次梳理并且把一些东西记录下来，方便下次搭建环境少走一些弯路。正好这次想搭一下主从同步，因此在virtualbox 上搭建了两个环境，均是 mysql 5.7.19 + php7.1.8 + node6.10.10 + nginx1.10.2 + centos6.9

# lnmp 环境部署

## 安装 mysql

我是下载mysql的yum源的rpm包进行安装
去 [官网yum列表](https://dev.mysql.com/downloads/repo/yum/) 中，找到你对应的版本源包地址    如图：
![yum源](/img/lnmp/mysql_yum.png)

点击 Download
![yum下载](/img/lnmp/mysql_download.png)

![yumlink](/img/lnmp/mysql_yum_link.png)

复制其中的下载链接, 进入centos 终端

```

  $ cd /tag

  $ wget https//dev.mysql.com/get/mysql57-community-release-el6-11.noarch.rpm

  $ rpm -Uvh mysql57-community-release-el6-11.noarch.rpm

```

通过命令 ` ls /etc/yum.repos.d ` 查看源是否加载成功

![mysql_linux_yum](/img/lnmp/mysql_linux_yum.png)

#### 通过yum源安装mysql

```
$ yum install mysql-community-server

$ service mysqld start  # 启动服务

$ grep 'temporary password' /var/log/mysqld.log #查看初始密码

# 去修改初始密码

$ mysql -u root -p
# 输入初始密码

mysql> alter user 'root'@'localhost' identified by 'YourNewPwd';

```
mysql 安装完毕

## 安装php7.1

同样是使用yum源安装

首先 将以下两个源 引入

```
$ rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm

$ rpm -Uvh https://mirror.webtatic.com/yum/el6/latest.rpm
```

加载完毕后可以通过yum search php71 查看相应的安装包


以下根据情况选择安装

```
# 基础
$ yum install php71w -y

$ yum install php71w-devel -y  # 此处要安装devel 包 不然找不到php命令
# nginx连接使用
$ yum install php71w-fpm -y
# 宽字节
$ yum install php71w-mbstring -y
# mysql相关
$ yum install php71w-mysqlnd -y
# redis扩展
$ yum install php71w-pecl-redis -y
# 加密使用
$ yum install php71w-mcrypt -y
# 性能加速 php5.5 以上使用
$ yum install php71w-opcache -y

```

## 安装nginx
用yum 默认安装就可以

```
$ yum install nginx -y
$ service nginx start

```
## 安装nodejs npm

由于项目中会用到nodejs，同时现在的现代前端开发都通过npm进行包管理，因此也记下nodejs的搭建

在 [node版本列表中](https://nodejs.org/dist/) 找到你需要的版本
此处我安装的是`v6.11.1`的版本

![node列表](/img/lnmp/node_list.png)

点击进入后， 会有各个平台的下载包
![node下载列表](/img/lnmp/node_down.png)
比较懒的朋友可以直接下载已编译好的包，省去自己编译的麻烦了，我就是比较懒的 😁

复制下载链接， 进入终端

```
$ cd /tmp

$ wget https://nodejs.org/dist/v6.11.1/node-v6.11.1-linux-x64.tar.gz

$ tar -zxvf node-v6.11.1-linux-x64.tar.gz
```

因为已经编译好了，只需要把解压后文件中的bin内的命令文件软连接到系统的命令就可以了

```
$ ln -s /tmp/node-v6.11.1-linux-x64/bin/node /usr/local/bin/node

$ ln -s /tmp/node-v6.11.1-linux-x64/bin/npm /usr/local/bin/npm
```
通过node -v和npm -v查看是否正常

## 安装git
朋友们应该知道，centos yum内置的git 版本只有1.7几 是不能正常使用的，需要升级到新版的git

其间安装git也遇到个小坑，也记一下，万一有朋友遇到跟我一样的问题呢，或者下次我再安装的时候出现问题呢

我们下载git的源码包编译安装

戳 [git下载列表](https://www.kernel.org/pub/software/scm/git/) 去下载你想要的版本， 这里我下载的2.14.0的
同样复制下载链接，进入终端下载

```
$ cd /tmp

$ wget https://www.kernel.org/pub/software/scm/git/git-2.14.0.tar.gz

$ tar -zxvf git-2.14.0.tar.gz

$ cd git-2.14.0
```
***此处有坑***：在编译安装之前，需要安装 `libcurl4-openssl-dev` 依赖，不然到时候git clone会出现`fatal: Unable to find remote helper for 'https'`错误，我的服务器是没安装这个包的，因此安装完后又跑回来重新编译了一下

```
$ yum install libcurl4-openssl-dev -y

# 安装完成后进入解压目录

$ pwd
/tmp/git-2.14.0

$ ./configure --prefix=/usr/local/git

$ make && make install

```
执行完毕之后 `ln -s /usr/local/git/bin/git /usr/local/bin/git`
通过命令 `git -version` 查看是否安装成功


# mysql 主从配置

在配置之前，先了解下mysql是如何进行主从配置的。 一般来说复制有三个步骤：

1. 在主库上把数据更改记录到二进制日志(Binary Log)中，即而进行日志事件
2. 备库将主库上的日志复制到自己的中继日志(Relay Log)中
3. 备库读取中继日志的事件，将其重放到备库数据上

可参考下图:
![主从配置](/img/lnmp/mysql_repl.png)

主从配置的前提是要两个数据库版本相同

首先在主库中，分配从库用户

```
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO backup@'192.168.0.*' IDENTIFIED BY 'backuppwd';
```

此处只是进行了简单的配置，该账号并不能进行select或修改数据

### 配置主库和从库
#### 主库配置
接下来是要在主库上开启一些设置，配置其打开二进制文件并指定一个独一无二的服务器id(server ID)，在主库的`my.cnf`文件中，增加或修改：

```
log_bin = mysql-bin
server_id = 1  # 通常设为1 但是不定
```

如果之前没有设置`log_bin`选项，则需要重启mysql 服务
为确保二进制日志文件是否创建，输入`SHOW MASTER STATUS`命令查看
正常情况下会出现 列名`File`值为`mysql-bin.000001`的条目
#### 从库配置
在从库中也需要增加类似配置,并重启mysql服务

```
# my.cnf

log_bin            = mysql-bin
server_id          = 2
relay_log          = /var/lib/mysql/mysql-relay-bin
log_slave_updates  = 1
read_only          = 1

```
```
不要在my.cnf中设置master_port或者master_host这类选项，已被废弃，只会导致坏处不会有好处
```
#### 启动复制
通过`CHANGE MASTER TO`语句连接主库启动复制

```
mysql> CHANGE MASTER TO MASTER_HOST='server1_host',
   - > MASTER_USER='backup',
   - > MASTER_PASSWORD='backuppwd',
   - > MASTER_LOG_FILE='mysql_bin.000001',
   - > MASTER_LOG_POS=0;
```

`MASTER_LOG_POS`设置为0，因为要从日志的开头读起，执行完上述语句后，通过`SHOW SLAVE STATUS\G;`命令查看是否正确执行
![mysql_slave](/img/lnmp/mysql_slave.png)

其中标红的正常应该是4和4，因为我这里的截图是配置完执行过复制后的状态，而且其中的`mysql-bin`文件的后缀应该是000001

运行 `START SLAVE;` 进行复制
该命令如果没有错误，再执行`SHOW SLAVE STATUS\G;` 应该就能看到两个数值增加并一致了

现在在主库中随意建一个表 会发现 在slave这个数据库中 也出现了，至此 最基本的主备复制模式 搭建完成 ，后续再了解下另外几种模式的搭建与各自的区别。有：主-主(双主复制)、主-主(主动-被动)、环形复制等
