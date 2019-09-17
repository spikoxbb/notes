[TOC]

# Falcon

首先一个项目应该以cfg入口。

在一台机器上启动所有的后端组件:./open-falcon start

## 组件

### Agent

agent用于采集机器负载监控指标，比如cpu.idle、load.1min、disk.io.util等等，每隔60秒push给Transfer。agent与Transfer建立了长连接，数据发送速度比较快，agent提供了一个http接口/v1/push用于接收用户手工push的一些数据，然后通过长连接迅速转发给Transfer。

```json
{
    "debug": true,  # 控制一些debug信息的输出，生产环境通常设置为false
    "hostname": "", # agent采集了数据发给transfer，endpoint就设置为了hostname，默认通过`hostname`获取，如果配置中配置了hostname，就用配置中的
    "ip": "", # agent与hbs心跳的时候会把自己的ip地址发给hbs，agent会自动探测本机ip，如果不想让agent自动探测，可以手工修改该配置
    "plugin": {
        "enabled": false, # 默认不开启插件机制
        "dir": "./plugin",  # 把放置插件脚本的git repo clone到这个目录
        "git": "https://registry-config.tuya-inc.top:7799/devops/falcon-plugin-o", # 放置插件脚本的git repo地址
        "logs": "./var" # 插件执行的log，如果插件执行有问题，可以去这个目录看log
    },
    "heartbeat": {
        "enabled": true,  # 此处enabled要设置为true
        "addr": "falcon-gateway.tuya-in.net:6030", # hbs的地址，端口是hbs的rpc端口
        "interval": 60, # 心跳周期，单位是秒
        "timeout": 1000 # 连接hbs的超时时间，单位是毫秒
    },
    "transfer": {
        "enabled": true, 
        "addrs": [
            "falcon-gateway.tuya-in.net:8433"
        ],  # transfer的地址，端口是transfer的rpc端口, 可以支持写多个transfer的地址，agent会保证HA
        "interval": 60, # 采集周期，单位是秒，即agent一分钟采集一次数据发给transfer
        "timeout": 1000 # 连接transfer的超时时间，单位是毫秒
    },
    "http": {
        "enabled": true,  # 是否要监听http端口
        "listen": ":1988",
        "backdoor": false
    },
    "collector": {
        "ifacePrefix": ["eth", "em"], # 默认配置只会采集网卡名称前缀是eth、em的网卡流量，配置为空就会采集所有的，lo的也会采集。可以从/proc/net/dev看到各个网卡的流量信息
        "mountPoint": []
    },
    "default_tags": {
    },
    "ignore": {  # 默认采集了200多个metric，可以通过ignore设置为不采集
        "cpu.busy": true,
       #省略......
    }
}
```

#### 进程管理

```
./open-falcon start agent  启动进程
./open-falcon stop agent  停止进程
./open-falcon monitor agent  查看日志
./falcon-agent --check
```

### Transfer

transfer是数据转发服务。它接收agent上报的数据，然后按照哈希规则进行数据分片、并将分片后的数据分别push给graph&judge等组件。

```json
{
    "debug": true,
    "moduleName": "transfer",
    "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
    "minStep": 30,# 允许上报的数据最小间隔，默认为30秒
    "http": {
        "enabled": true,#表示是否开启该http端口，该端口为控制端口，主要用来对transfer发送控制命令、统计命令、debug命令等
        "listen": "0.0.0.0:6060"
    },
    "rpc": {
        "enabled": true,# 表示是否开启数据接收端口, Agent发送数据使用的就是该端口
        "listen": "0.0.0.0:8433"
    },
    "socket": {#表示是否开启该telnet方式的数据接收端口，这是为了方便用户一行行的发送数据给transfer
        "enabled": true,
        "listen": "0.0.0.0:4444",
        "timeout": 3600
    },
    "judge": {
        "enabled": true,#表示是否开启向judge发送数据
        "batch": 200,#数据转发的批量大小，可以加快发送速度，建议保持默认值
        "connTimeout": 1000,#单位是毫秒，与后端建立连接的超时时间
        "callTimeout": 5000,#单位是毫秒，发送数据给后端的超时时间
        "maxConns": 32,#连接池相关配置，最大连接数，建议保持默认
        "maxIdle": 32,#连接池相关配置，最大空闲连接数，建议保持默认
        "replicas": 500,#一致性hash算法需要的节点副本数量
        "cluster": {#后端的judge列表，其中key代表后端judge名字，value代表的是具体的ip:port
            "judge-00" : "127.0.0.1:6080"
        }
    },
    "graph": {
        "enabled": true,#表示是否开启向graph发送数据
        "batch": 200,#数据转发的批量大小，可以加快发送速度，建议保持默认值
        "connTimeout": 1000,
        "callTimeout": 5000,
        "maxConns": 32,#连接池相关配置，最大连接数，建议保持默认
        "maxIdle": 32,#连接池相关配置，最大空闲连接数，建议保持默认
        "replicas": 500,
        "cluster": {#表示后端的graph列表，其中key代表后端graph名字，value代表的是具体的ip:port(多个地址用逗号隔开, transfer会将同一份数据发送至各个地址，利用这个特性可以实现数据的多重备份)
            "graph-00" : "127.0.0.1:607000"
        }
    },
    "tsdb": {#表示是否开启向open tsdb发送数据
        "enabled": false,#数据转发的批量大小，可以加快发送速度
        "batch": 200,
        "connTimeout": 1000,
        "callTimeout": 5000,
        "maxConns": 32,
        "maxIdle": 32,
        "retry": 3,# 连接后端的重试次数和发送数据的重试次数
        "address": "127.0.0.1:8088"#tsdb地址或者tsdb集群vip地址, 通过tcp连接tsdb
    }
}
```

#### 进程管理

```
./open-falcon start transfer  启动进程
./open-falcon stop transfer  停止进程
./open-falcon monitor transfer  查看日志
# 校验服务,这里假定服务开启了6060的http监听端口。检验结果为ok表明服务正常启动。
curl -s "127.0.0.1:6060/health"
```

### Graph

graph是存储绘图数据的组件。graph组件接收transfer组件推送上来的监控数据，同时处理api组件的查询请求、返回绘图数据。

```json
{
   "debug": false,
   "moduleName": "graph",
   "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
   "http": {
      "enabled": true,#表示是否开启该http端口，该端口为控制端口，主要用来对graph发送控制命令、统计命令、debug命令
      "listen": "0.0.0.0:6071"
   },
   "rpc": {
      "enabled": true,#表示是否开启该rpc端口，该端口为数据接收端口
      "listen": "0.0.0.0:6070"
   },
   "rrd": {
      "storage": "./data/6070"#历史数据的文件存储路径
   },
   "api": {
      "connect_timeout": 500,
      "request_timeout": 2000,
      "plus_api": "http://127.0.0.1:8088",
      "plus_api_token": "default-token-used-in-server-side"
   },
   "callTimeout": 5000,#RPC调用超时时间，单位ms
   "ioWorkerNum": 64,#底层io.Worker的数量
   "migrate": {#扩容graph时历史数据自动迁移
      "enabled": false,#表示graph是否处于数据迁移状态
      "concurrency": 2,#数据迁移时的并发连接数
      "replicas": 500,#一致性hash算法需要的节点副本数量，建议不要变更，保持默认即可（必须和transfer的配置中保持一致）
      "cluster": {#未扩容前老的graph实例列表
         "graph-00" : "127.0.0.1:6070"
      }
   }
}
```

#### 进程管理

```
./open-falcon start graph  启动进程
./open-falcon stop graph  停止进程
./open-falcon monitor graph  查看日志
```

###  API

api组件，提供统一的restAPI操作接口。比如：api组件接收查询请求，根据一致性哈希算法去相应的graph实例查询不同metric的数据，然后汇总拿到的数据，最后统一返回给用户。

```json
{
	"log_level": "debug",
	"moduleName": "api",
	"apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
	"database": {
		"addr": "root:root@tcp(10.0.200.5:3306)/tuya_falcon?charset=utf8&parseTime=True&loc=Local",
		"db_bug": true
	},
	"graphs": {#graph模块的部署列表信息
		"cluster": {
			"graph-00": "127.0.0.1:6070"
		},
		"max_conns": 100,
		"max_idle": 100,
		"conn_timeout": 1000,
		"call_timeout": 5000,
		"numberOfReplicas": 500
	},
	"metric_list_file": "./api/data/metric",
	"web_port": ":8080",#http监听端口
	"access_control": true,#如果设置为false，那么任何用户都可以具备管理员权限
	"signup_disable": false,
	"salt": "",#数据库加密密码的时候的salt
	"skip_auth": false,#如果设置为true，那么访问api就不需要经过认证
	"default_token": "default-token-used-in-server-side",#用于服务端各模块间的访问授权
	"gen_doc": false,
	"gen_doc_path": "doc/module.html",
	"penglai": {
		"region_api": "http://101.132.46.223:8080/api/v1/open/getMachineRegion",
		"username": "admin",
		"password": "Abcd1234",
		"proxy": "http://vpn.work.tuya-inc.com:31229"
	},
	"ela_http": "127.0.0.1:9123"
}

```

#### 进程管理

```
./open-falcon start api  启动进程
./open-falcon stop api  停止进程
./open-falcon monitor api  查看日志
```

### HBS

所有agent都会连到HBS，每分钟发一次心跳请求。

Portal的数据库中有一个host表，维护了所有机器的信息，比如hostname、ip等等。这个表是从公司CMDB中同步过来的。agent发送心跳信息给HBS的时候，会把hostname、ip、agent version、plugin version等信息告诉HBS，HBS负责更新host表。

除了要采集这些基础信息之外,agent如何知道自己应该采集哪些端口和进程呢？向HBS要，HBS去读取Portal的数据库，返回给agent。

Judge需要获取所有的报警策略,既然HBS无论如何都要访问Portal的数据库了，那就让HBS去获取所有的报警策略缓存在内存里，然后Judge去向HBS请求。这样一来，对Portal DB的压力就会大大减小。

```json
{
    "debug": true,
    "moduleName": "hbs",
    "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
    "database": {#Portal的数据库地址
        "addr" : "root:@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true",
        "max_idle": 20,
        "max_conn": 20
    },
    "hosts": "",
    "listen": ":6030",# hbs监听的rpc地址
    "trustable": [""],
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6031"# hbs监听的http地址
    }
}

```

#### 进程管理

```
./open-falcon start hbs  启动进程
./open-falcon stop hbs  停止进程
./open-falcon monitor hbs  查看日志
```

### Judge

Judge用于告警判断，agent将数据push给Transfer，Transfer不但会转发给Graph组件来绘图，还会转发给Judge用于判断是否触发告警。

因为监控系统数据量比较大，一台机器显然是搞不定的，所以必须要有个数据分片方案。Transfer通过一致性哈希来分片，每个Judge就只需要处理一小部分数据就可以了。

Judge监听了一个http端口，提供了一个http接口：/count，访问之，可以得悉当前Judge实例处理了多少数据量。

```json
{
    "debug": true,
    "moduleName": "judge",
    "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
    "debugHost": "nil",
    "remain": 11, #指定了judge内存中针对某个数据存多少个点，比如host01这个机器的cpu.idle的值在内存中最多存多少个，配置报警的时候比如all(#3)，这个#后面的数字不能超过remain-1，一般维持默认就够用了.
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6081"
    },
    "rpc": {
        "enabled": true,
        "listen": "0.0.0.0:6080"
    },
    "hbs": {
        "servers": ["127.0.0.1:6030"],
        "timeout": 300,
        "interval": 60
    },
    "alarm": {
        "enabled": true,
        "minInterval": 300,# 连续两个报警之间至少相隔的秒数，维持默认即可
        "queuePattern": "event:p%v",
        "redis": {
            "dsn": "127.0.0.1:6379",
            "password": "",
            "maxIdle": 5,
            "connTimeout": 5000,
            "readTimeout": 5000,
            "writeTimeout": 5000
        }
    },
    "redis": {
        "addr": "127.0.0.1:6379",
        "password": "",
        "maxIdle": 5,
        "connTimeout": 5000,
        "readTimeout": 5000,
        "writeTimeout": 5000
    }
}

```

#### 进程管理

```
./open-falcon start judge  启动进程
./open-falcon stop judge  停止进程
./open-falcon monitor judge  查看日志
```

### Alarm

alarm模块是处理报警event的，judge产生的报警event写入redis，alarm从redis读取处理，并进行不同渠道的发送。

在配置报警策略的时候配置了报警级别，比如P0/P1/P2等等，每个及别的报警都会对应不同的redis队列 alarm去读取这个数据的时候我们希望先读取P0的数据，再读取P1的数据，最后读取P5的数据，因为我们希望先处理优先级高的。于是：用了redis的brpop指令。

已经发送的告警信息，alarm会写入MySQL中保存，这样用户就可以在dashboard中查阅历史报警，同时针对同一个策略发出的多条报警，在MySQL存储的时候，会聚类；历史报警保存的周期，是可配置的，默认为7天。

```json
           
{
    "log_level": "debug",
    "moduleName": "alarm",
    "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:9913"
    },
    "redis": {
        "addr": "127.0.0.1:6379",
        "password": "",
        "maxIdle": 5,
        "highQueues": [
            "event:p0",
            "event:p1",
            "event:p2"
        ],
        "lowQueues": [
            "event:p3",
            "event:p4",
            "event:p5",
            "event:p6"
        ],
        "userIMQueue": "/queue/user/im",
        "userSmsQueue": "/queue/user/sms",
        "userMailQueue": "/queue/user/mail",
        "syncUserFlag": "/sync/userFlag",
        "syncGroupFlag": "/sync/groupFlag",
        "cleanCaseFlag": "/clean/caseFlag"
    },
    "api": {
        "im": "http://10.0.200.3:7003/qywx.do?do=message&opt=sendMessage&agentId=1000006",
        "sms": "http://127.0.0.1:10086/sms",
        "mail": "http://127.0.0.1:10086/mail",
        "dashboard": "http://127.0.0.1:8081",
        "dashboard_api":"http://127.0.0.1:8080",
        "plus_api":"http://127.0.0.1:8089",#falcon-plus api模块的运行地址
        "plus_api_token": "default-token-used-in-server-side",#用于和falcon-plus api模块服务端之间的通信认证token
        "penglai_ops_api":"http://192.168.12.92:8080/api/v1/open/getPicById?username=admin&passwd=admin"
    },
    "database": {
        "addr": "root:root@tcp(10.0.200.5:3306)/tuya_falcon?charset=utf8&parseTime=True&loc=Local",
        "max_idle": 10,
        "max_conn": 100
    },
    "worker": {
        "im": 10,
        "sms": 10,
        "mail": 50
    },
    "housekeeper": {
        "event_retention_days": 7,#报警历史信息的保留天数
        "event_delete_batch": 100
    },
    "cron": {
        "user": "0 0 0 * * 1-5",
        "user_api":"http://10.0.200.3:7003/qywx.do?do=employee&opt=getEmployeeAllOnTheJob",
        "group": "0 0 */2 * * *",
        "app_api":"http://192.168.12.92:8080/api/v1/open/getAppList?username=admin&passwd=admin",
        "host_api":"http://192.168.12.92:8080/api/v1/open/getMachineListById?username=admin&passwd=admin"
    }
}
```

#### 进程管理

```
./open-falcon start alarm  启动进程
./open-falcon stop alarm  停止进程
./open-falcon monitor alarm  查看日志
```

如果某个核心服务挂了，可能会造成大面积报警，为了减少报警短信数量，我们做了报警合并功能。把报警信息写入dashboard模块，然后dashboard返回一个url地址给alarm，alarm将这个url链接发给用户，这样用户只要收到一条短信（里边是个url地址），点击url进去就是多条报警内容。

highQueues中配置的几个event队列中的事件是不会做报警合并的，因为那些是高优先级的报警，报警合并只是针对lowQueues中的事件。如果所有的事件都不想做报警合并，就把所有的event队列都配置到highQueues中即可

### Task

task是监控系统一个必要的辅助模块。定时任务，实现了如下几个功能：

- index更新。包括图表索引的全量更新 和 垃圾索引清理。
- falcon服务组件的自身状态数据采集。定时任务了采集了transfer、graph、task这三个服务的内部状态数据。
- falcon自检控任务。

```json
{
    "debug": false,#表示是否开启该http端口，该端口为控制端口，主要用来对task发送控制命令、统计命令、debug命令等
    "moduleName": "task",
    "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
    "http": {
        "enable": true,
        "listen": "0.0.0.0:8002"
    },
    "database":{
        "addr": "root:root@tcp(127.0.0.1:3306)/graph?loc=Local&parseTime=true",
        "max_idle": 4
    },
    "index": {#表示是否开启索引更新任务
        "enable": false,
        "dsn": "root:root@tcp(127.0.0.1:3306)/graph?loc=Local&parseTime=true",#索引服务的MySQL的连接信息
        "maxIdle": 4,
        "autoDelete": false,#是否自动删除垃圾索引
        "cluster":{#后端graph索引更新的定时任务描述。一条记录的形如: "graph地址:执行周期描述"，通过设置不同的执行周期，来实现负载在时间上的均衡。
            "test.hostname01:6071" : "0 0 0 ? * 0-5",
            "test.hostname02:6071" : "0 30 0 ? * 0-5"
        }
    },
    "collector" : {
        "enable": true,#表示是否开启falcon的自身状态采集任务
        "destUrl" : "http://127.0.0.1:1988/v1/push",#监控数据的push地址,默认为本机的1988接口
        "srcUrlFmt" : "http://%s/statistics/all",# 监控数据采集的url格式, %s将由机器名或域名替换
        "cluster" : [#falcon后端服务列表，用具体的"module,hostname:port"表示，module取值可以为graph、transfer、task等
            "transfer,test.hostname:6060",
            "graph,test.hostname:6071",
            "task,test.hostname:8001"
        ]
    }
}
```

#### 进程管理

```
./control start 启动进程
./control stop  停止进程
# 校验服务,这里假定服务开启了8002的http监听端口。检验结果为ok表明服务正常启动。
curl -s "127.0.0.1:8002/health"
```

服务启动后，可以通过日志查看服务的运行状态，日志文件地址为./var/app.log。可以通过调试脚本`./test/debug`查看服务器的内部状态数据，如 运行 `bash ./test/debug` 可以得到服务器内部状态的统计信息。

#### 过期索引清除

监控数据停止上报后，该数据对应的索引也会停止更新、变为过期索引。过期索引，影响视听，部分用户希望删除之。

原来的方案，是: 通过task模块，有数据上报的索引、每天被更新一次，7天未被更新的索引、清除之。但是，很多用户不能正确配置graph实例的http接口，导致正常上报的监控数据的索引 无法被更新；7天后，合法索引被task模块误删除。为了解决上述问题停掉了task模块自动删除过期索引的功能(autoDelete=false)；如果你确定配置的index.cluster正确无误，可以自行打开该功能。

手动删除过期索引的方法:

1. 进行一次索引数据的全量更新。方法为: 针对每个graph实例，运行`curl -s "127.0.0.1:6071/index/updateAll"`，异步地触发graph实例的索引全量更新(这里假设graph的http监听端口为6071)，等待所有的graph实例完成索引全量更新后 进行第2步操作。单个graph实例，索引全量更新的耗时，因counter数量、mysql数据库性能而不同，一般耗时不大于30min。
2. 待索引全量更新完成后，发起过期索引删除 `curl -s "$Hostname.Of.Task:$Http.Port/index/delete"`。运行索引删除前，请务必**确保索引全量更新已完成**。

### Nodata

nodata用于检测监控数据的上报异常。nodata和实时报警judge模块协同工作.配置了nodata的采集项超时未上报数据，nodata生成一条默认的模拟数据；用户配置相应的报警策略，收到mock数据就产生报警。

```json
{
    "debug": true,
    "moduleName": "nodata",
    "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
    "extra_region": "nx",
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6090"
    },
    "plus_api":{
        "connectTimeout": 500,
        "requestTimeout": 2000,
        "addr": "http://127.0.0.1:8080",#falcon-plus api模块的运行地址
        "token": "default-token-used-in-server-side"#用于和falcon-plus api模块的交互认证token
    },
    "database": {
        "enabled": true,
        "addr": "root:@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true&wait_timeout=604800",
        "max_idle": 4
    },
    "collector":{
        "enabled": true,
        "batch": 200,
        "concurrent": 10
    },
    "sender":{
        "enabled": true,
        "connectTimeout": 500,
        "requestTimeout": 2000,
        "transferAddr": "127.0.0.1:6060",#transfer的http监听地址,一般形如"domain.transfer.service:6060"
        "batch": 500
    },
    "redis": {
        "addr": "127.0.0.1:6379",
        "password": "",
        "maxIdle": 5,
        "Flag": "/falcon/nodata/flag"
    }
}

```

#### 进程管理

```
./open-falcon start nodata  启动进程
./open-falcon stop nodata  停止进程
./open-falcon monitor nodata  查看日志
```

### Aggregator

集群聚合模块。聚合某集群下的所有机器的某个指标的值，提供一种集群视角的监控体验。

```json
{
    "debug": true,
    "moduleName": "aggregator",
    "apolloURL":"http://in-mousika-config.tuya-inc.top:8181/configs/falcon-backend-s/default/",
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6055"
    },
    "database": {
        "addr": "root:@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true",
        "max_idle": 10,
        "ids": [1, -1],
        "interval": 55
    },
    "api": {
        "connect_timeout": 500,
        "request_timeout": 2000,
        "plus_api": "http://127.0.0.1:8080",#falcon-plus api模块的运行地址
        "plus_api_token": "default-token-used-in-server-side",#和falcon-plus api 模块交互的认证token
        "push_api": "http://127.0.0.1:1988/v1/push" #push数据的http接口，这是agent提供的接口
    },
    "redis": {
        "addr": "127.0.0.1:6379",
        "password": "",
        "maxIdle": 5,
        "maxWorker":5,
        "flag": "/falcon/aggregator/flag",
        "queue":"/falcon/aggregator/queue"
    }
}
```

#### 进程管理

```
./open-falcon start aggregator  启动进程
./open-falcon stop aggregator  停止进程
./open-falcon monitor aggregator  查看日志
```

## 源码相关解读

[报警发送原理](Sender.md)

[HBS原理](HBS.md)

[报警判断原理](judge.md)

[报警原理](Alarm.md)

[发送数据点原理](Transfer.md)

[agent原理](Agent.md)