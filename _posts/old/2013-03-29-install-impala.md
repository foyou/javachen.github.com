---
layout: post
title: 安装Impala过程
category: hadoop
tags: [hadoop, impala, cdh]
keywords: impala
---

与Hive类似，Impala也可以直接与HDFS和HBase库直接交互。只不过Hive和其它建立在MapReduce上的框架适合需要长时间运行的批处理任务。例如：那些批量提取，转化，加载（ETL）类型的Job，而Impala主要用于实时查询。

Hadoop集群各节点的环境设置及安装过程见[手动安装Cloudera Hadoop CDH](/2013/03/24/manual-install-Cloudera-Hadoop-CDH),参考这篇文章，hadoop各个组件和jdk版本如下：

```
	hadoop-2.0.0-cdh4.6.0
	hbase-0.94.15-cdh4.6.0
	hive-0.10.0-cdh4.6.0
	jdk1.6.0_38
```

hadoop各组件可以在[这里](http://archive.cloudera.com/cdh4/cdh/4/)下载。

集群规划为7个节点，每个节点的ip、主机名和部署的组件分配如下：

```
	192.168.0.1        desktop1     NameNode、Hive、ResourceManager、impala
	192.168.0.2        desktop2     SSNameNode
	192.168.0.3        desktop3     DataNode、HBase、NodeManager、impala
	192.168.0.4        desktop4     DataNode、HBase、NodeManager、impala
	192.168.0.5        desktop5     DataNode、HBase、NodeManager、impala
	192.168.0.6        desktop6     DataNode、HBase、NodeManager、impala
	192.168.0.7        desktop7     DataNode、HBase、NodeManager、impala
```

# install

下载 impala，目前最新版本为`1.2.4`，[下载地址](http://archive.cloudera.com/impala/redhat/6/x86_64/impala/1.2.4/)。

# 安装过程

安装前提:

1、配置`hostname`，关闭防火墙

2、先安装好java、hadoop集群以及hive，可以参考我的文章：

* [手动安装Cloudera Hadoop CDH](/2013/03/24/manual-install-Cloudera-Hadoop-CDH4.2.html)
* [手动安装Cloudera Hive CDH](/2013/03/24/manual-install-Cloudera-hive-CDH4.2.html)

impala仓库列表如下：

- Red Hat 5 repo file ([http://archive.cloudera.com/impala/redhat/5/x86_64/impala/cloudera-impala.repo](http://archive.cloudera.com/impala/redhat/5/x86_64/impala/cloudera-impala.repo)) in /etc/yum.repos.d/.
- Red Hat 6 repo file ([http://archive.cloudera.com/impala/redhat/6/x86_64/impala/cloudera-impala.repo](http://archive.cloudera.com/impala/redhat/6/x86_64/impala/cloudera-impala.repo)) in /etc/yum.repos.d/.
- SUSE repo file ([http://archive.cloudera.com/impala/sles/11/x86_64/impala/cloudera-impala.repo](http://archive.cloudera.com/impala/sles/11/x86_64/impala/cloudera-impala.repo)) in /etc/zypp/repos.d/.
- Ubuntu 10.04 list file ([http://archive.cloudera.com/impala/ubuntu/lucid/amd64/impala/cloudera.list](http://archive.cloudera.com/impala/ubuntu/lucid/amd64/impala/cloudera.list)) in /etc/apt/sources.list.d/.
- Ubuntu 12.04 list file ([http://archive.cloudera.com/impala/ubuntu/precise/amd64/impala/cloudera.list](http://archive.cloudera.com/impala/ubuntu/precise/amd64/impala/cloudera.list)) in /etc/apt/sources.list.d/.
- Debian list file ([http://archive.cloudera.com/impala/debian/squeeze/amd64/impala/cloudera.list](http://archive.cloudera.com/impala/debian/squeeze/amd64/impala/cloudera.list)) in /etc/apt/sources.list.d/.

我使用的是centos系统，在配置好yum源之后，只需要执行如下命令即可：

```
$ sudo yum install impala             # Binaries for daemons
$ sudo yum install impala-server      # Service start/stop script
$ sudo yum install impala-state-store # Service start/stop script
$ sudo yum install impala-catalog     # Service start/stop script
$ sudo yum install impala-shell
```

**你可以按照你的部署规划，在不同节点上装上不同的服务**。

# 配置Impala
## 查看安装路径

```
	[root@desktop1 conf]# find / -name impala
	/var/run/impala
	/var/lib/alternatives/impala
	/var/log/impala
	/usr/lib/impala
	/etc/alternatives/impala
	/etc/default/impala
	/etc/impala
```

## 添加配置文件

impalad的配置文件路径由环境变量`IMPALA_CONF_DIR`指定，默认为`/usr/lib/impala/conf`

在节点desktop1上 拷贝`hive-site.xml`、`core-site.xml`、`hdfs-site.xml`至`/usr/lib/impala/conf`目录下:

```
	[root@desktop1 conf]# mkdir /usr/lib/impala/conf/
	[root@desktop1 conf]# cp /opt/hadoop-2.0.0-cdh4.6.0/etc/hadoop/log4j.properties /usr/lib/impala/conf/
	[root@desktop1 conf]# cp /opt/hadoop-2.0.0-cdh4.6.0/etc/hadoop/core-site.xml /usr/lib/impala/conf/
	[root@desktop1 conf]# cp /opt/hadoop-2.0.0-cdh4.6.0/etc/hadoop/hdfs-site.xml /usr/lib/impala/conf/
	[root@desktop1 conf]# cp /opt/hive-0.10.0-cdh4.6.0/conf/hive-site.xml /usr/lib/impala/conf/
```

并作下面修改在`hdfs-site.xml`文件中添加如下内容：

```xml
	<property>
	    <name>dfs.client.read.shortcircuit</name>
	    <value>true</value>
	</property>
	 
	<property>
	    <name>dfs.domain.socket.path</name>
	    <value>/var/run/hadoop-hdfs/dn._PORT</value>
	</property>

	<property>
	  <name>dfs.datanode.hdfs-blocks-metadata.enabled</name>
	  <value>true</value>
	</property>
```

同步以上文件到其他节点

```
	[root@desktop1 ~]# scp -r /usr/lib/impala/conf desktop3:/usr/lib/impala/
	[root@desktop1 ~]# scp -r /usr/lib/impala/conf desktop4:/usr/lib/impala/
	[root@desktop1 ~]# scp -r /usr/lib/impala/conf desktop5:/usr/lib/impala/
	[root@desktop1 ~]# scp -r /usr/lib/impala/conf desktop6:/usr/lib/impala/
	[root@desktop1 ~]# scp -r /usr/lib/impala/conf desktop7:/usr/lib/impala/
```

## hadoop中添加native包

拷贝hadoop native包到hadoop安装路径下，并同步hadoop文件到其他节点：

```
	[root@desktop1 ~]# cp /usr/lib/impala/lib/*.so* /opt/hadoop-2.0.0-cdh4.6.0/lib/native/
```

## 创建socket path

在每个节点上创建/var/run/hadoop-hdfs:

```
	[root@desktop1 ~]# mkdir -p /var/run/hadoop-hdfs
	[root@desktop3 ~]# mkdir -p /var/run/hadoop-hdfs
	[root@desktop4 ~]# mkdir -p /var/run/hadoop-hdfs
	[root@desktop5 ~]# mkdir -p /var/run/hadoop-hdfs
	[root@desktop6 ~]# mkdir -p /var/run/hadoop-hdfs
	[root@desktop7 ~]# mkdir -p /var/run/hadoop-hdfs
```

拷贝postgres jdbc jar：

```
	cp /opt/hive-0.10.0-cdh4.6.0/lib/postgresql-9.1-903.jdbc* /usr/lib/impala/lib/
```

# 启动服务

1. 在hive所在节点启动statestored（默认端口为24000）:

```
	GLOG_v=1 nohup statestored -state_store_port=24000 &
```

如果statestore正常启动，可以在`/tmp/statestored.INFO`查看。如果出现异常，可以查看`/tmp/statestored.ERROR`定位错误信息。

2. 在所有impalad节点上：

```
	HADOOP_CONF_DIR="/usr/lib/impala/conf" nohup impalad -state_store_host=desktop1 -nn=desktop1 \
		-nn_port=8020 -hostname=desktop3 -ipaddress=192.168.0.3 &
```

注意： 其中的`-hostname`和`-ipaddress`表示当前启动impalad实例所在机器的主机名和ip地址。

如果impalad正常启动，可以在`/tmp/ impalad.INFO`查看。如果出现异常，可以查看`/tmp/impalad.ERROR`定位错误信息。

# 使用shell

使用`impala-shell`启动Impala Shell，分别连接各Impalad主机(我这里是：desktop3、desktop4、desktop5、desktop6、desktop7)，刷新元数据，之后就可以执行shell命令。相关的命令如下(可以在任意节点执行)：

```
	>impala-shell
	[Not connected] >connect desktop3:21000
	[desktop3:21000] >refresh
	[desktop3:21000] >connect desktop4:21000
	[desktop4:21000] >refresh
```

# 注意：

1. 如果hive使用mysql或postgres数据库作为metastore的存储，则需要拷贝相应的jdbc jar到`/usr/lib/impala/lib`目录下
2. 如果出现`E0325 11:04:19.937718  7239 statestored-main.cc:52] Could not start webserver on port: 25010`，可能是已经启动了statestored进程

# 参考文章
* [Impala安装文档完整版](http://yuntai.1kapp.com/?p=904)
* [Impala入门笔记](http://tech.uc.cn/?p=817)
* [Installing and Using Cloudera Impala](https://ccp.cloudera.com/display/IMPALA10BETADOC/Installing+and+Using+Cloudera+Impala)
