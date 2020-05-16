# hive��mysql�İ�װ����ϸ��

[TOC]

��װ�汾��apache-hive-2.1.1-bin
http://archive.apache.org/dist/hive/

## 1.��װMySQL���ݿ�	

### 	1.���ذ�װMySQL

?		wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
		rpm -ivh mysql-community-release-el7-5.noarch.rpm
		yum update
		yum install -y mysql-server

### 	2.����MySQL����

?		systemctl start mysqld.service
		systemctl restart mysqld.service

### 	3.ͨ��MySQL�Դ���װ�ű���������

?		/usr/bin/mysql_secure_installation��ynyy��

### 	4.����MySQL�ͻ��˽�����Ȩ(Զ������)

 		grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;
		flush privileges;

## 2.hive��װ����

### 	1.��ѹ��װ��

?		tar -zxf /root/software/apache-hive-2.1.1-bin.tar.gz -C /usr/local/src/

### 	2.�޸�hive�������ļ�

#### 		1������hive�µ�confĿ¼������һ��hive-env.sh

```
HADOOP_HOME=/usr/local/src/hadoop-2.6.0
# Hive Configuration Directory can be controlled by:
export HIVE_CONF_DIR=/usr/local/src/apache-hive-2.1.1-bin/conf	
```

#### 		2���޸�hive-site.xml(�Լ�����)�������������

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
### 3.�ϴ�mysql��lib�����ǳ���Ҫ��

?	��mysql��lib���ϴ���hive��libĿ¼��
	mysql-connector-java-5.1.38.jar

### 4.����hive�Ļ�������

?	����slave2�Ļ�������

```
vim /etc/profile
	export HIVE_HOME=/usr/local/src/apache-hive-2.1.1-bin
	export PATH=:HIVE_HOME/bin:PATH
```

### 5.����

#### 		1������hiveĿ¼��

?		bin/hive

#### 		2��Hive Jdbc

������Ҫ����hiveserver2
ǰ̨������bin/hive --service hiveserver2
��̨������nohup bin/hive --service hiveserver2 &
��̨����metastore
bin/hive --service metastore 2>&1 >> /var/log.log &
bin/beeline ����hiveserver2 ��**���beeline����ο�����������ͼ���**��
https://blog.csdn.net/JENREY/article/details/79870864
��Ҫ�����������ļ����õ�Hadoop��etc/hadoop/core-site.xml����

##### 		ע�������ļ��е�root��������hadoop��Ⱥ���û���������������Ϊ�ģ�

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

?			��һ��������ʽ��
				bin/beeline
				!connect jdbc:hive2://bigdata3:10000
				��������û���������(����Hadoop���û���������)
			�ڶ���������ʽ��
				beeline  -u  jdbc:hive2://bigdata3:10000  -n  root

#### 		3��Hive����

?		ʹ�� �Ce  ������ֱ��ִ��hql�����
		bin/hive -e "use myhive;select * from test;"

		ʹ�� �Cf  ����ͨ��ָ���ı��ļ���ִ��hql�����
		vim hive.sql
		use myhive;select * from test;
	
		bin/hive -f hive.sql

### 6. Hive �Ļ�������

1.���ݿ����

1.1  �������ݿ�

```
create database if not exists myhive; 
use  myhive;
```

1.2 �������ݿⲢָ��λ��

```
create database myhive2 location '/myhive2';
```

1.3 �������ݿ��ֵ����Ϣ

���ݿ������һЩ�����Եļ�ֵ����Ϣ���ڴ���ʱ��ӣ�

```
create database foo with dbproperties ('owner' = 'itcast','date = '20190120');
```

�鿴���ݿ�ļ�ֵ����Ϣ��

```
describe database extended foo;
```

�޸����ݿ�ļ�ֵ����Ϣ��

```
alter database foo set dbproperties ('owner' = 'itheima');
```

1.4 �鿴���ݿ������ϸ��Ϣ

```
desc database extended myhive2;
```

1.5 ɾ�����ݿ�

ɾ��һ�������ݿ⣬������ݿ����������ݱ���ô�ͻᱨ��

```
drop database myhive2;
```

ǿ��ɾ�����ݿ⣬�������ݿ�����ı�һ��ɾ��

```
drop database myhive cascade;
```