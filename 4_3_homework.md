# **实现步骤**

### 准备虚拟机环境 [参考](https://www.cnblogs.com/junqilian/p/11515594.html)

因为MongoDB集群实例比较多，创建虚拟机时注意需要至少10G磁盘空间，否则需要后期扩容。

登录到虚拟机，如下修改一下静态IP地址，免得IP地址冲突。

```
vi /etc/sysconfig/network-scripts/ifcfg-enp0s8
```

<img src="/Users/liqiaoqiao/Library/Application Support/typora-user-images/image-20200907213632959.png" alt="image-20200907213632959" style="zoom:50%;" />

然后重启网络

```
systemctl restart network
```

这时候就可以从主机Macbook上用ZenTermLite访问虚拟机上linux服务器了。

还可以修改一下主机名。

```
hostnamectl set-hostname mongo
```

### 下载MongoDB安装文件

下载 mongodb-linux-x86_64-4.1.3.tgz 到本机。用ZenTermLite ssh连接到虚拟机，给linux服务器安装lrzsz，即可将本机文件拖拽上传到linux服务器。

```
yum -y install lrzsz
```

修改一下名字

```
mv mongodb-linux-x86_64-4.1.3 sharding_cluster
cd sharding_cluster
```

## 搭建配置节点集群，分片节点集群，路由节点

### **1.配置并启动config节点集群 **

```
mkdir config
cd config
vi config-17011.conf
```

节点1: config-17011.conf

```
# 数据库文件位置 
dbpath=config/config1 
#日志文件位置 
logpath=config/logs/config1.log 
# 以追加方式写入日志 
logappend=true
# 是否以守护进程方式运行 
fork = true 
bind_ip=0.0.0.0
port = 17011
# 表示是一个配置服务器 
configsvr=true 
#配置服务器副本集名称 
replSet=configsvr
```

```
mkdir config1 logs
cd config
cp config-17011.conf config-17013.conf
```

节点2: config-17013.conf

```
# 数据库文件位置 
dbpath=config/config2 
#日志文件位置 
logpath=config/logs/config2.log 
# 以追加方式写入日志 
logappend=true
# 是否以守护进程方式运行 
fork = true 
bind_ip=0.0.0.0
port = 17013
# 表示是一个配置服务器 
configsvr=true 
#配置服务器副本集名称 
replSet=configsvr
```

节点3: config-17015.conf

```
# 数据库文件位置 
dbpath=config/config3 
#日志文件位置 
logpath=config/logs/config3.log 
# 以追加方式写入日志 
logappend=true
# 是否以守护进程方式运行 
fork = true 
bind_ip=0.0.0.0
port = 17015
# 表示是一个配置服务器 
configsvr=true 
#配置服务器副本集名称 
replSet=configsvr
```

启动配置节点

```
chmod -R 740 bin
```

```
[root@mongo sharding_cluster]# ./bin/mongod -f config/config-17011.conf 
./bin/mongod -f config/config-17013.conf 
./bin/mongod -f config/config-17015.conf
```

进入任意节点的mongo shell 并添加 配置节点集群 注意use admin

```
./bin/mongo --port 17011 

use admin

var cfg ={"_id":"configsvr",
"members":[ {"_id":1,"host":"192.168.56.200:17011"}, {"_id":2,"host":"192.168.56.200:17013"}, 
{"_id":3,"host":"192.168.56.200:17015"}] 
};

rs.initiate(cfg)
//重新查看集群状态
rs.status()
```

### **2.配置并启动shard集群**

shard1集群搭建37011到37017

再开启一个连接到虚拟机

```
cd sharding_cluster
mkdir shard
cd shard
mkdir shard1 shard2 shard3 shard4
cd shard1 
mkdir shard1-37011 shard1-37013 shard1-37015 shard1-37017 logs
```



```
[root@mongo shard1]# vi shard1-37011.conf
dbpath=shard/shard1/shard1-37011 
bind_ip=0.0.0.0
port=37011
fork=true 
logpath=shard/shard1/logs/shard1-37011.log 
replSet=shard1
shardsvr=true

cp shard1-37011.conf shard1-37013.conf
vi shard1-37013.conf
dbpath=shard/shard1/shard1-37013
bind_ip=0.0.0.0
port=37013
fork=true 
logpath=shard/shard1/logs/shard1-37013.log 
replSet=shard1
shardsvr=true

dbpath=shard/shard1/shard1-37015 
bind_ip=0.0.0.0
port=37015
fork=true 
logpath=shard/shard1/logs/shard1-37015.log 
replSet=shard1
shardsvr=true

dbpath=shard/shard1/shard1-37017 
bind_ip=0.0.0.0
port=37017
fork=true 
logpath=shard/shard1/logs/shard1-37017.log 
replSet=shard1
shardsvr=true
```

启动每个mongod 然后进入其中一个进行集群配置

```
[root@mongo sharding_cluster]# ./bin/mongod -f shard/shard1/shard1-37011.conf
./bin/mongod -f shard/shard1/shard1-37013.conf
./bin/mongod -f shard/shard1/shard1-37015.conf
./bin/mongod -f shard/shard1/shard1-37017.conf

./bin/mongo --port 37011
var cfg ={"_id":"shard1", 
"protocolVersion" : 1,
"members":[ 
{"_id":1,"host":"192.168.56.200:37011"}, 
{"_id":2,"host":"192.168.56.200:37013"},
{"_id":3,"host":"192.168.56.200:37015"},
{"_id":4,"host":"192.168.56.200:37017","arbiterOnly":true} ] //增加一个特殊的仲裁节点
}; 
rs.initiate(cfg) 
rs.status()
//另一种增加一个特殊的仲裁节点的方式 注入节点 执行 
//rs.addArb("192.168.56.200:37017")
```

shard2集群搭建47011到47017

再开启一个连接到虚拟机

```
cd /root/sharding_cluster/shard/shard2 
mkdir shard2-47011 shard2-47013 shard2-47015 shard2-47017 logs
```



```
vi shard2-47011.conf
dbpath=shard/shard2/shard2-47011 
bind_ip=0.0.0.0
port=47011
fork=true 
logpath=shard/shard2/logs/shard2-47011.log 
replSet=shard2
shardsvr=true

vi shard2-47013.conf
dbpath=shard/shard2/shard2-47013 
bind_ip=0.0.0.0
port=47013
fork=true 
logpath=shard/shard2/logs/shard2-47013.log 
replSet=shard2
shardsvr=true

vi shard2-47015.conf
dbpath=shard/shard2/shard2-47015 
bind_ip=0.0.0.0
port=47015
fork=true 
logpath=shard/shard2/logs/shard2-47015.log 
replSet=shard2
shardsvr=true

vi shard2-47017.conf
dbpath=shard/shard2/shard2-47017 
bind_ip=0.0.0.0
port=47017
fork=true 
logpath=shard/shard2/logs/shard2-47017.log 
replSet=shard2
shardsvr=true
```

启动每个mongod 然后进入其中一个进行集群配置

```
[root@mongo sharding_cluster]# ./bin/mongod -f shard/shard2/shard2-47011.conf
./bin/mongod -f shard/shard2/shard2-47013.conf
./bin/mongod -f shard/shard2/shard2-47015.conf
./bin/mongod -f shard/shard2/shard2-47017.conf

./bin/mongo --port 47011

var cfg ={"_id":"shard2", 
"protocolVersion" : 1,
"members":[ 
{"_id":1,"host":"192.168.56.200:47011"}, 
{"_id":2,"host":"192.168.56.200:47013"},
{"_id":3,"host":"192.168.56.200:47015"},
{"_id":4,"host":"192.168.56.200:47017","arbiterOnly":true} ]
}; 
rs.initiate(cfg) 
rs.status()
//另一种增加一个特殊的仲裁节点的方式 注入节点 执行  
//rs.addArb("192.168.56.200:47017")
```

以此类推创建第3、4个分片节点集群

```
cd /root/sharding_cluster/shard/shard3 
mkdir shard3-57011 shard3-57013 shard3-57015 shard3-57017 logs

vi shard3-57011.conf
dbpath=shard/shard3/shard3-57011 
bind_ip=0.0.0.0
port=57011
fork=true 
logpath=shard/shard3/logs/shard3-57011.log 
replSet=shard3
shardsvr=true

vi shard3-57013.conf
dbpath=shard/shard3/shard3-57013 
bind_ip=0.0.0.0
port=57013
fork=true 
logpath=shard/shard3/logs/shard3-57013.log 
replSet=shard3
shardsvr=true

vi shard3-57015.conf
dbpath=shard/shard3/shard3-57015 
bind_ip=0.0.0.0
port=57015
fork=true 
logpath=shard/shard3/logs/shard3-57015.log 
replSet=shard3
shardsvr=true

vi shard3-57017.conf
dbpath=shard/shard3/shard3-57017 
bind_ip=0.0.0.0
port=57017
fork=true 
logpath=shard/shard3/logs/shard3-57017.log 
replSet=shard3
shardsvr=true

cd /root/sharding_cluster
./bin/mongod -f shard/shard3/shard3-57011.conf
./bin/mongod -f shard/shard3/shard3-57013.conf
./bin/mongod -f shard/shard3/shard3-57015.conf
./bin/mongod -f shard/shard3/shard3-57017.conf

var cfg ={"_id":"shard3", 
"protocolVersion" : 1,
"members":[ 
{"_id":1,"host":"192.168.56.200:57011"}, 
{"_id":2,"host":"192.168.56.200:57013"},
{"_id":3,"host":"192.168.56.200:57015"},
{"_id":4,"host":"192.168.56.200:57017","arbiterOnly":true} ]
}; 
rs.initiate(cfg) 
rs.status()
//另一种增加一个特殊的仲裁节点的方式 注入节点 执行  
//rs.addArb("192.168.56.200:57017")
```

```
cd /root/sharding_cluster/shard/shard4 
mkdir shard4-58011 shard4-58013 shard4-58015 shard4-58017 logs

vi shard4-58011.conf
dbpath=shard/shard4/shard4-58011 
bind_ip=0.0.0.0
port=58011
fork=true 
logpath=shard/shard4/logs/shard4-58011.log 
replSet=shard4
shardsvr=true

vi shard4-58013.conf
dbpath=shard/shard4/shard4-58013 
bind_ip=0.0.0.0
port=58013
fork=true 
logpath=shard/shard4/logs/shard4-58013.log 
replSet=shard4
shardsvr=true

vi shard4-58015.conf
dbpath=shard/shard4/shard4-58015 
bind_ip=0.0.0.0
port=58015
fork=true 
logpath=shard/shard4/logs/shard4-58015.log 
replSet=shard4
shardsvr=true

vi shard4-58017.conf
dbpath=shard/shard4/shard4-58017 
bind_ip=0.0.0.0
port=58017
fork=true 
logpath=shard/shard4/logs/shard4-58017.log 
replSet=shard4
shardsvr=true

cd /root/sharding_cluster
./bin/mongod -f shard/shard4/shard4-58011.conf
./bin/mongod -f shard/shard4/shard4-58013.conf
./bin/mongod -f shard/shard4/shard4-58015.conf
./bin/mongod -f shard/shard4/shard4-58017.conf

./bin/mongo --port 58011

var cfg ={"_id":"shard4", 
"protocolVersion" : 1,
"members":[ 
{"_id":1,"host":"192.168.56.200:58011"}, 
{"_id":2,"host":"192.168.56.200:58013"},
{"_id":3,"host":"192.168.56.200:58015"},
{"_id":4,"host":"192.168.56.200:58017","arbiterOnly":true} ]
}; 
rs.initiate(cfg) 
rs.status()
//另一种增加一个特殊的仲裁节点的方式 注入节点 执行  
//rs.addArb("192.168.56.200:57017")
```



### **3.配置和启动路由节点**

```
mkdir route/logs -p
vi route-27017.conf
```

```
port=27017
bind_ip=0.0.0.0
fork=true
logpath=route/logs/route.log
configdb=configsvr/192.168.56.200:17011,192.168.56.200:17013,192.168.56.200:17015
```

启动路由节点使用mongos (注意不是mongod)

```
cd ..
./bin/mongos -f route/route-27017.conf
```

### 4. **mongos(路由)中添加分片节点**

进入路由mongos

```
./bin/mongo --port 27017
sh.status() sh.addShard("shard1/192.168.56.200:37011,192.168.56.200:37013,192.168.56.200:37015,192.168.56.200:37017"); sh.addShard("shard2/192.168.56.200:47011,192.168.56.200:47013,192.168.56.200:47015,192.168.56.200:47017");
sh.addShard("shard3/192.168.56.200:57011,192.168.56.200:57013,192.168.56.200:57015,192.168.56.200:57017");
sh.addShard("shard4/192.168.56.200:58011,192.168.56.200:58013,192.168.56.200:58015,192.168.56.200:58017");
sh.status()
```

### 5.**开启数据库和集合分片(指定片键)**

继续使用mongos完成分片开启和分片大小设置

```
为数据库开启分片功能
sh.enableSharding("lg_resume")
为指定集合开启分片功能
sh.shardCollection("lg_resume.lg_resume_datas",{"片键字段名如 name":索引说明})
```

### **6.** **向集合中插入数据测试**

通过路由循环向集合中添加数

```
use lg_resume;
for(var i=1;i<= 1000;i++){
db.lg_resume_datas.insert({"name":"test"+i, salary:(Math.random()*20000).toFixed(2)});
}
```

### 7.**验证分片效果** 

分别进入 shard1, shard2, shard3, shard4 中的数据库进行验证

## 权限控制

##### 1. 切换到admin创建管理员

MongoDB 服务端开启安全检查之前，至少需要有一个管理员账号，admin 数据库中的用户都被视为管 理员

```
use admin;
db.createUser( {
  user:"root",
  pwd:"123456",
  roles:[{role:"root",db:"admin"}]
} )
```

##### 2. 切换到lg_resume创建普通用户

```
>show dbs
admin 0.000GB
config 0.000GB
local 0.000GB
lg_resume 0.000GB
> use lg_resume
switched to lg_resume
> db.lg_resume_datas.insert({name:"testdb1"}) 
> show tables
lg_resume_datas
> db.lg_resume_datas.find()
...

> db.createUser({
... user:"lagou_gx",
... pwd:"abc321",
... roles:[{role:"readWrite",db:"lg_resume"}] 
... })
```

接着从客户端关闭 MongoDB 服务端，之后服务端会以安全认证方式进行启动

```
> use lg_resume
switched to db lg_resume
> db.shutdownServer() 
server should be down...
```

**3. MongoDB** **安全认证方式启动
** mongod --dbpath=数据库路径 --port=端口 --auth

也可以在配置文件中 加入 auth=true

## 编写Spring Boot项目进行测试

```
spring.data.mongodb.username=lagou_gx
spring.data.mongodb.password=abc321 
#spring.data.mongodb.uri=mongodb://账号:密码@IP:端口/数据库名
```

