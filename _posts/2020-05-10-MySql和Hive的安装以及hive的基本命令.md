# hive及mysql的安装（最细）

[TOC]

安装版本：apache-hive-2.1.1-bin
http://archive.apache.org/dist/hive/

## 1.安装MySQL数据库	

### 	1.下载安装MySQL

?		wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
		rpm -ivh mysql-community-release-el7-5.noarch.rpm
		yum update
		yum install -y mysql-server

### 	2.启动MySQL服务

?		systemctl start mysqld.service
		systemctl restart mysqld.service

### 	3.通过MySQL自带安装脚本进入设置

?		/usr/bin/mysql_secure_installation（ynyy）

### 	4.进入MySQL客户端进行授权(远程连接)

 		grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;
		flush privileges;

## 2.hive安装部署

### 	1.解压安装包

?		tar -zxf /root/software/apache-hive-2.1.1-bin.tar.gz -C /usr/local/src/

### 	2.修改hive的配置文件

#### 		1、进入hive下的conf目录，复制一份hive-env.sh

```
HADOOP_HOME=/usr/local/src/hadoop-2.6.0
# Hive Configuration Directory can be controlled by:
export HIVE_CONF_DIR=/usr/local/src/apache-hive-2.1.1-bin/conf	
```

#### 		2、修改hive-site.xml(自己创建)，添加以下内容

        <configuration>
        <property>
                <name>javax.jdo.option.ConnectionURL</name>
                <value>jdbc:mysql://bigdata3:3306/hive?createDatabaseIfNotExist=true</value>
        </property>
    	<property>
                <name>javax.jdo.option.ConnectionDriverName</name>
                <value>com.mysql.jdbc.Driver</value>
        </property>
        <property>
                <name>javax.jdo.option.ConnectionUserName</name>
                <value>root</value>
        </property>
        <property>
                <name>javax.jdo.option.ConnectionPassword</name>
                <value>123456</value>
        </property>
    <property>
                <name>datanucleus.schema.autoCreateAll</name>
                <value>true</value>
        </property>
        <property>
                <name>hive.metastore.schema.verification</name>
                <value>false</value>
        </property>
        <property>
                <name>hive.server2.thrift.bind.host</name>
                <value>bigdata3</value>
        </property>
        <!--
        <property>
                <name>hive.metastore.uris</name>
                <value>thrift://bigdata3:9083</value>
        </property>
    -->
    </configuration>
### 3.上传mysql的lib包（非常重要）

?	将mysql的lib包上传到hive的lib目录下
	mysql-connector-java-5.1.38.jar

### 4.配置hive的环境变量

?	配置slave2的环境变量

```
vim /etc/profile
	export HIVE_HOME=/usr/local/src/apache-hive-2.1.1-bin
	export PATH=:HIVE_HOME/bin:PATH
```

### 5.启动

#### 		1、进入hive目录下

?		bin/hive

#### 		2、Hive Jdbc

首先需要启动hiveserver2
前台启动：bin/hive --service hiveserver2
后台启动：nohup bin/hive --service hiveserver2 &
后台启动metastore
bin/hive --service metastore 2>&1 >> /var/log.log &
bin/beeline 连接hiveserver2 （**如果beeline报错参考下面这个博客即可**）
https://blog.csdn.net/JENREY/article/details/79870864
需要将以下配置文件配置到Hadoop的etc/hadoop/core-site.xml当中

##### 		注：配置文件中的root就是启动hadoop集群的用户名（我是这样认为的）

```
<property>
    <name>hadoop.proxyuser.root.hosts</name>
    <!--value>master</value-->
    <value>*</value>
</property>

<property>
    <name>hadoop.proxyuser.root.groups</name>
    <!--value>hadoop</value-->
    <value>*</value>
</property>
```

?			第一种启动方式：
				bin/beeline
				!connect jdbc:hive2://bigdata3:10000
				最后输入用户名和密码(启动Hadoop的用户名和密码)
			第二种启动方式：
				beeline  -u  jdbc:hive2://bigdata3:10000  -n  root

#### 		3、Hive命令

?		使用 –e  参数来直接执行hql的语句
		bin/hive -e "use myhive;select * from test;"

		使用 –f  参数通过指定文本文件来执行hql的语句
		vim hive.sql
		use myhive;select * from test;
	
		bin/hive -f hive.sql

### 6. Hive 的基本操作

1.数据库操作

1.1  创建数据库

```
create database if not exists myhive; 
use  myhive;
```

1.2 创建数据库并指定位置

```
create database myhive2 location '/myhive2';
```

1.3 设置数据库键值对信息

数据库可以有一些描述性的键值对信息，在创建时添加：

```
create database foo with dbproperties ('owner' = 'itcast','date = '20190120');
```

查看数据库的键值对信息：

```
describe database extended foo;
```

修改数据库的键值对信息：

```
alter database foo set dbproperties ('owner' = 'itheima');
```

1.4 查看数据库更多详细信息

```
desc database extended myhive2;
```

1.5 删除数据库

删除一个空数据库，如果数据库下面有数据表，那么就会报错

```
drop database myhive2;
```

强制删除数据库，包含数据库下面的表一起删除

```
drop database myhive cascade;
```