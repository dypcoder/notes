# elasticsearch7.0安装

- 环境列表
   * [elasticsearch-7.x](https://www.elastic.co/cn/downloads/elasticsearch "elasticsearch-7.x")
   执行命令：`wget https://artifacts.elastic.co/downloads/elasticsearch/${elasticsearch-7.x}-linux-x86_64.tar.gz`
   * [node-v10.15.3](https://nodejs.org/en/download/)(这边我原本有node 8.6.x的直接升级为10.15.3了)升级过程参考 [这个大  哥等博客](https://www.cnblogs.com/jackson0714/p/node.html)
   * [elasticsearch-head](https://nodejs.org/en/download/)
   执行命令：`wget https://github.com/mobz/elasticsearch-head/archive/master.zip`
  

------------

## 安装 ElasticSearch

#### 1. 解压并移动到安装目录
 我这里下载的版本是7.0.1
```Bash
  [root@localhost ~]# tar -zxvf  elasticsearch-7.0.1-linux-x86_64.tar.gz
  [root@localhost ~]# mv elasticsearch-7.0.1 /usr/local/elasticsearch-7.0.1
```
#### 2. 创建用户并赋予ElasticSearch目录权限
```Bash
  [root@localhost ~]# groupadd es  
  [root@localhost ~]# useradd  es -g es -p es
  [root@localhost ~]# chown  es /usr/local/elasticsearch-7.0.1/ -R
```
#### 3.修改ElasticSearch配置信息
```Bash
  [root@localhost ~]# vi /usr/local/elasticsearch-7.0.1/config/elasticsearch.yml
```
下面为我的配置信息
```Bash
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: my-es
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
node.name: node-1
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
#path.data: /path/to/data
#
# Path to log files:
#
#path.logs: /path/to/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#这里配置为0.0.0.0可以让外网访问，当让也可以指定ip
network.host: 0.0.0.0
#
# Set a custom port for HTTP:
#浏览器访问的端口号
http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
#discovery.seed_hosts: ["host1", "host2"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
# 这里不配置会报这个错误信息：  the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
cluster.initial_master_nodes: ["node-1"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
#gateway.recover_after_nodes: 3
#
# For more information, consult the gateway module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
#下面这两个不加的话外部浏览器无法访问
#
http.cors.enabled: true
http.cors.allow-origin: "*"
```

------------
#### 4.修改系统配置信息
 ***以下操作都在ROOT用户下***
 查看硬限制命令
 ```Bash
  [root@localhost ~]# ulimit -Hn
```
修改硬限制
 ```Bash
  [root@localhost ~]# vim /etc/security/limits.conf 
```
在文件尾部加入然后保存
 ```Bash
  *                hard    nofile          65536
```
编辑xx--nproc.conf文件 我的是20--nproc.conf
 ```Bash
[root@localhost ~]# vim /etc/security/limits.d/20-nproc.conf
```
编辑xx--nproc.conf文件 我的是20--nproc.conf
 ```Bash
[root@localhost ~]#  vim /etc/security/limits.d/20-nproc.conf
```
将 * soft nproc xxx 修改为
 ```Bash
 *          soft    nproc     4096
```
修改sysctl.conf文件配置
 ```Bash
[root@localhost ~]# vi /etc/sysctl.conf
```
在其中加入并保存
 ```Bash
vm.max_map_count=262144
```
然后执行命令
```Bash
[root@localhost ~]# sysctl -p
```
**最后退出当钱登陆的root用户使以上配置生效**


#### 5.启动

切换为之前为es建立的用户 启动
```Bash
  [root@localhost bin]# sudo su es
  [root@localhost ~]# cd /usr/local/elasticsearch-7.0.1/bin/
  [root@localhost ~]#./elasticsearch
```
后台启动
```Bash
  [root@localhost ~]#./elasticsearch -d   
```
如果按照以上配置的话直接启动应该是没问题

#### 5.部分报错
```diff
- error_1: [1]: max file descriptors [xxxx] for elasticsearch process is too low, increase to at least [65535]
- error_2: [2]: max number of threads [xxxx] for user [es] is too low, increase to at least [4096]
+ 参照第4步操作
```
```diff
- error_3: can not run elasticsearch as root
+ 切换为为非root用户操作
```
```diff
- error_4: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
+ 查看上面等配置文件步骤
```
```diff
- error_5:  Exception in thread "main" java.nio.file.AccessDeniedException: /usr/local/elasticsearch-6.2.2/config/jvm.options 
+ 切换为root用户
+ 运行如下语句 es为之前创建的用户 /usr/local/elasticsearch-7.0.1 为ElasticSearch的路径
chown es /usr/local/elasticsearch-7.0.1 -R
```
```diff
- error_6:  Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x0000000085330000, 2060255232, 0) failed; error='Cannot allocate memory' (errno=12)
+ 修改 /usr/local/elasticsearch-7.0.1/config/jvm.options
[root@localhost ~]# vi /usr/local/elasticsearch-7.0.1/config/jvm.options

#-Xms2g
#-Xmx2g
-Xms512m
-Xmx512m
```

------------

## 安装 Elasticsearch-head插件

1.安装Grunt工具
```Bash
  [root@localhost ~]# npm install -g grunt-cli
  [root@localhost ~]#  npm install grunt --save-dev
  [root@localhost local]# grunt -version
   grunt-cli v1.3.2
```
2.安装Elasticsearch-head插件
```Bash
  [root@localhost ~]# wget https://github.com/mobz/elasticsearch-head/archive/master.zip
  [root@localhost ~]# yum install unzip
  [root@localhost ~]# unzip master.zip
  [root@localhost ~]# mv elasticsearch-head-master/ /usr/local/elasticsearch-head
  [root@localhost ~]# cd /usr/local/elasticsearch-head/
  [root@localhost elasticsearch-head]# vim Gruntfile.js
```
3.找到connect: {server: {   修改文件的内容为下面这样
```javascript
 connect: {
                        server: {
                                options: {
                                        hostname: '*',
                                        port: 9100,
                                        base: '.',
                                        keepalive: true
                                }
                        }
                }
```
4.修改app.js的localhost为ip地址
用 /app.App查找到对应位置
修改为入图所示
![image_1](https://github.com/dypcoder/notes/blob/master/image/tempsnip.png "image_1")

5.然后在Elasticsearch-head文件夹下运行
```Bash
 [root@localhost elasticsearch-head]# npm install
```
可能会遇到  Error: Command failed: tar jxf /tmp/phantomjs/phantomjs-2.1.1-linux-x86_64.tar.bz2
这个报错 
运行
```Bash
[root@localhost elasticsearch-head]# yum install bzip2
```

安装bzip2
6.在/usr/local/elasticsearch-head 目录下 运行

```Bash
[root@localhost elasticsearch-head]# yum install bzip2
```
启动 elasticsearch-head

出现下图代表成功
![image_2](https://github.com/dypcoder/notes/blob/master/image/grunt.PNG "image_2")



