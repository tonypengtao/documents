# codis部署
### 1、系统环境：CentOS release 6.9 (Final) * 3台
### 2、IP列表：

* 172.16.212.100
* 172.16.212.101
* 172.16.212.102

### 3、环境描述：

* go：go1.10.3.linux-amd64.tar.gz
* jdk：jdk-8u151-linux-x64.tar.gz
* zookeeper：zookeeper-3.4.10.tar.gz
* codis：codis-3.2.2

### 4、结构规划

| IP | zookeeper | codis-server | redis-sentinel | codis-proxy | codis-dashboard | codis-fe |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 100 | port:2181 | master:6379 </br> slave:6380 </br> requirepass:redis | sentinel:26379 | -- | 11080 | 9090 |
| 101 | port:2182 | master:6379 </br> slave:6380 </br> requirepass:redis | sentinel:26379 | admin_port:110800 </br> proxy_port:19000 | -- | -- |
| 102 | port:2183 | master:6379 </br> slave:6380 </br> requirepass:redis | sentinel:26379 | admin_port:110800 </br> proxy_port:19000 | -- | -- |

/srv目录结构：

```
|- /srv
|----- codis                            codis配置存放目录
|-------- data
|----------- codis
|-------------- log                     日志目录
|-------------- redis                   redis目录
|----------------- redis-6379           redis-6379配置文件及快照存放目录
|----------------- redis-6380           redis-6380配置文件及快照存放目录
|----------------- sentinel             redis哨兵配置
|-------------- run                     pid文件存放目录
|----- jdk                              jdk安装路径
|----- zookeeper-3.4.10                 zookeeper安装路径
|-------- data                              数据存放目录
|-------- logs                              日志存放目录
|-------- conf                              配置文件存放目录
```

/usr目录结构：

```
|- /usr
|----- local
|-------- go                                go安装路径
|-------- codis                             codis安装路径
```

### 5、安装jdk（每台都要安装）

1. 上传jdk-8u151-linux-x64.tar.gz至root目录下
2. 解压到jdk包到/srv目录下

`tar -xvzf /root/jdk-8u151-linux-x64.tar.gz -C /srv`

3. 重命名解压后的文件夹为jdk

`mv /srv/jdk1.8.0_151/ /srv/jdk`

4. 配置环境变量

`vim /etc/profile`

```
文件末尾添加
JAVA_HOME=/srv/jdk
CLASS_PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/lib
export PATH=$PATH:$JAVA_HOME/bin
```

5. 刷新环节变量

`source /etc/profile`

6. 测试java

`java -version`

### 6、安装go（每台都要安装）

1. 上传go1.10.3.linux-amd64.tar.gz至root目录下
2. 解压到go包到/usr/local目录下

`tar -xvzf /root/go1.10.3.linux-amd64.tar.gz -C /usr/local`

3. 配置环境变量

`vim /etc/profile`

```
文件末尾添加
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin:$JAVA_HOME/bin
```

4. 刷新环境变量

`source /etc/profile`

5. 测试

`go version`

![-w243](http://pic.itsuantou.com/15337339729718.jpg)

### 7、安装zookeeper（每台都要安装）

1. 上传zookeeper-3.4.10.tar.gz至root目录下
2. 解压到zookeeper包到/srv目录下

`tar -xvzf /root/zookeeper-3.4.10.tar.gz -C /srv`

3. 配置环境变量

`vim /etc/profile`

```
文件末尾添加
export ZOOKEEPER_HOME=/srv/zookeeper-3.4.10
export PATH=$PATH:$GOROOT/bin:$JAVA_HOME/bin:$ZOOKEEPER_HOME/bin
```

4. 创建data、logs文件夹

```
mkdir -p /srv/zookeeper-3.4.10/data
mkdir -p /srv/zookeeper-3.4.10/logs
```

5. 复制默认配置文件为zoo.cfg，并修改该文件

`cp /srv/zookeeper-3.4.10/conf/zoo_sample.cfg /srv/zookeeper-3.4.10/conf/zoo.cfg`

修改文件

`vim /srv/zookeeper-3.4.10/conf/zoo.cfg`

修改dataDir路径为：/srv/zookeeper-3.4.10/data
修改dataLogDir路径为：/srv/zookeeper-3.4.10/logs
配置集群IP地址和端口：
server.1=172.16.212.100:2881:3881
server.2=172.16.212.101:2882:3882
server.3=172.16.212.102:2883:3883

![-w546](http://pic.itsuantou.com/15337348160475.jpg)

* tickTime：这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
* dataDir：顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。
* clientPort：这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。
* initLimit：这个配置项是用来配置 Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 5个心跳的时间（也就是tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒
* syncLimit：这个配置项标识 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 2*2000=4 秒
* server.A=B：C：D：其中 A 是一个数字，表示这个是第几号服务器；B 是这个服务器的 ip 地址；C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口。如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号。


6. 在/srv/zookeeper-3.4.10/data 下建立一个myid文件，文件内容为 1、2、3

172.16.212.100执行下面语句

`echo "1" >> /srv/zookeeper-3.4.10/data/myid`

172.16.212.101执行下面语句

`echo "2" >> /srv/zookeeper-3.4.10/data/myid`

172.16.212.102执行下面语句

`echo "3" >> /srv/zookeeper-3.4.10/data/myid`

7. 依次启动服务

`zkServer.sh start`

查看状态

`zkServer.sh status`

![-w386](http://pic.itsuantou.com/15337843693581.jpg)

### 8、安装codis（每台都要安装）

1. 上传3.2.2.zip源码包至root目录下或者直接下载

```
cd /root
wget https://github.com/CodisLabs/codis/archive/3.2.2.zip
```

2. 解压

`unzip 3.2.2.zip`
 
3. 创建codis文件夹并把解压后的文件复制到文件夹中

```
mkdir -p /usr/local/codis/src/github.com/CodisLabs
cp -fr /root/codis-3.2.2/ /usr/local/codis/src/github.com/CodisLabs/codis
编译源码
cd /usr/local/codis/src/github.com/CodisLabs/codis
make
```

在/usr/local/codis/src/github.com/CodisLabs/codis/bin文件夹内生成codis-admin、codis-dashboard、codis-fe、codis-ha、codis-proxy、codis-server六个可执行文件
4. 配置环境变量

`vim /etc/profile`

```
文件末尾添加
export CODIS=/usr/local/codis/src/github.com/CodisLabs/codis
export PATH=$PATH:$GOROOT/bin:$JAVA_HOME/bin:$ZOOKEEPER_HOME/bin:$CODIS/bin:$CODIS/admin
```

5. 刷新环境变量

`source /etc/profile`

### 9、配置codis-server（每台都要安装）

Codis安装完成后，就可以为Codis创建一些标准的目录，用来存储Codis的脚本目录、配置文件目录、日志目录、PID目录、Redis配置目录等。codis-server就是redis-server程序,属于codis优化版本,配合codis集群使用
1. 创建文件夹

```
mkdir -p /srv/codis/data/codis/log
mkdir -p /srv/codis/data/codis/redis/redis-6379
mkdir -p /srv/codis/data/codis/redis/redis-6380
mkdir -p /srv/codis/data/codis/redis/sentinel
mkdir -p /srv/codis/data/codis/run
```

2. 增加配置文件
可以复制/usr/local/codis/src/github.com/CodisLabs/codis/config下的redis.conf
配置主库：/srv/codis/data/codis/redis/redis-6379/redis.conf

```
##允许后台运行
daemonize yes
##设置端口,最好是非默认端口
port 6379
##绑定登录IP,安全考虑,最好是内网
bind 0.0.0.0
##命名并指定当前redis的PID路径,用以区分多个redis
pidfile "/srv/codis/data/codis/run/redis-6379.pid"
##命名并指定当前redis日志文件路径
logfile "/srv/codis/data/codis/log/redis-6379.log"
##指定RDB文件名,用以备份数据到硬盘并区分不同redis,当使用内存超过可用内存的45%时触发快照功能
dbfilename "dump_6379.rdb"
##指定当前redis的根目录,用来存放RDB/AOF文件
dir "/srv/codis/data/codis/redis/redis-6379"
##当前redis的认证密钥,redis运行速度非常快,这个密码要足够强大,
##所有codis-proxy集群相关的redis-server认证密码必须全部一致
requirepass "redis"
##当前redis的最大容量限制,建议设置为可用内存的45%内,最高能设置为系统可用内存的95%,
##可用config set maxmemory 去在线修改,但重启失效,需要使用config rewrite命令去刷新配置文件
##注意,使用codis集群,必须配置容量大小限制,不然无法启动
maxmemory 100000kb
##LRU的策略,有四种,看情况选择
maxmemory-policy allkeys-lru
##如果做故障切换，不论主从节点都要填写密码且要保持一致
masterauth "redis"
```

配置从库

```
##允许后台运行
daemonize yes
##设置端口,最好是非默认端口
port 6380
##绑定登录IP,安全考虑,最好是内网
bind 0.0.0.0
##命名并指定当前redis的PID路径,用以区分多个redis
pidfile "/srv/codis/data/codis/run/redis-6380.pid"
##命名并指定当前redis日志文件路径
logfile "/srv/codis/data/codis/log/redis-6380.log"
##指定RDB文件名,用以备份数据到硬盘并区分不同redis,当使用内存超过可用内存的45%时触发快照功能
dbfilename "dump_6380.rdb"
##指定当前redis的根目录,用来存放RDB/AOF文件
dir "/srv/codis/data/codis/redis/redis-6380"
##当前redis的认证密钥,redis运行速度非常快,这个密码要足够强大,
##所有codis-proxy集群相关的redis-server认证密码必须全部一致
requirepass "redis"
##当前redis的最大容量限制,建议设置为可用内存的45%内,最高能设置为系统可用内存的95%,
##可用config set maxmemory 去在线修改,但重启失效,需要使用config rewrite命令去刷新配置文件
##注意,使用codis集群,必须配置容量大小限制,不然无法启动
maxmemory 100000kb
##LRU的策略,有四种,看情况选择
maxmemory-policy allkeys-lru
##如果做故障切换，不论主从节点都要填写密码且要保持一致
masterauth "redis"
#配置主节点信息
slaveof 172.16.212.100 6379
```

除了端口号不同带来的文件名不同.实际上从库配置只是多了最后一行,指定了主库地址
3. 启动服务

```
codis-server /srv/codis/data/codis/redis/redis-6379/redis.conf
codis-server /srv/codis/data/codis/redis/redis-6380/redis.conf
```

4. 检查

`ss -ntplu |grep codis-server`

![-w770](http://pic.itsuantou.com/15337976058998.jpg)

### 10、配置redis-sentinel（每台都要安装）

正确来说,redis-sentinel是要配置主从架构才能生效,但是在codis集群中并不一样,因为他的配置由zookeeper来维护,所以,这里codis使用的redis-sentinel只需要配置一些基本配置就可以了
1. 配置

`vim /srv/codis/data/codis/redis/sentinel/sentinel_conf`

创建文件并增加配置内容

```
bind 0.0.0.0
protected-mode no
port 26379
dir "/srv/codis/data/codis/redis/sentinel"
pidfile "/srv/codis/data/codis/run/sentinel_26379.pid"
logfile "/srv/codis/data/codis/log/sentinel_26379.log"
daemonize yes
```

2. 启动

`redis-sentinel /srv/codis/data/codis/redis/sentinel/sentinel_conf`

3. 检查

`redis-cli -p 26379 -c info Sentinel`

![-w577](http://pic.itsuantou.com/15337979602927.jpg)

**注意:没有配置好codis-dashboard会没最后那几行,因为还不受zookeeper控制,所以是正常的,配置好之后才会自动加载进来.**

### 11、配置codis-proxy

配置机器：
* 172.16.212.101
* 172.16.212.102

1. 生成默认的配置文件并修改

`codis-proxy --default-config | tee /srv/codis/data/codis/conf/proxy.conf`

```
#项目名称,会登记在zookeeper里,如果你想一套zookeeper管理多套codis,就必须区分好
product_name = "codis-demo"
#设置登录dashboard的密码(与真实redis中requirepass一致)
product_auth = "redis"
#客户端(redis-cli)的登录密码(与真实redis中requirepass不一致),是登录codis的密码
session_auth = "codis"
#管理的端口,0.0.0.0即对所有ip开放,基于安全考虑,可以限制内网
admin_addr = "0.0.0.0:11080"
#用那种方式通信,假如你的网络支持tcp6的话就可以设别的
proto_type = "tcp4"
#客户端(redis-cli)访问代理的端口,0.0.0.0即对所有ip开放
proxy_addr = "0.0.0.0:19000"
#外部配置存储类型,我们用的就是zookeeper,当然也是还有其他可以支持,这里不展开说
jodis_name = "zookeeper"
#配置zookeeper的连接地址,这里是三台就填三台
jodis_addr = "172.16.212.100:2181,172.16.212.101:2182,172.16.212.102:2183"
#zookeeper的密码,假如有的话
jodis_auth = ""
#codis代理的最大连接数,默认是1000,并发大要调大
proxy_max_clients = 100000
#假如并发太大,你可能需要调这个pipeline参数,大多数情况默认10000就够了
session_max_pipeline = 100000
#并发大也可以改以下参数
backend_max_pipeline = 204800
session_recv_bufsize = "256kb"
session_recv_timeout = "0s"
```

2. 启动codis-proxy

`codis-proxy --ncpu=1 --config=/data/redis/data/config/proxy.conf --log=/data/redis/data/logs/proxy.log &`

3. 检查

`ss -ntplu |grep codis-proxy`

![-w797](http://pic.itsuantou.com/15337996942614.jpg)

--ncpu    指定使用多少个cpu,我的是虚拟机,所以就1了,如果你是多核,那就填多个
--config    指定配置文件,就是刚才的配置文件
--log    指定输出日志文件
配置并启动成功,但是暂时还用不了,因为还需要配置codis-dashboard才能最终完成.

### 12、配置codis-dashboard

配置机器：
* 172.16.212.100

1. 修改dashboard.toml配置文件

`vim /usr/local/codis/src/github.com/CodisLabs/codis/config/dashboard.toml`

```
主要修改这几行
# Set Coordinator, only accept "zookeeper" & "etcd" & "filesystem".
coordinator_name = "zookeeper"
coordinator_addr = "192.168.188.120:2181,192.168.188.121:2181,192.168.188.122:2181"

# Set Codis Product Name/Auth.
product_name = "codis-demo"
product_auth = "redis"

# Set bind address for admin(rpc), tcp only.
admin_addr = "0.0.0.0:18080"
```

2. 启动脚本
启动前需要修改脚本zookeeper地址池与product名称

`vim /usr/local/codis/src/github.com/CodisLabs/codis/admin/codis-dashboard-admin.sh`

```
#coordinator_name = "filesystem"
#coordinator_addr = "/tmp/codis"
coordinator_name = "zookeeper"
coordinator_addr = "172.16.212.100:2181,172.16.212.101:2182,172.16.212.102:2183"
#coordinator_auth = ""

# Set Codis Product Name/Auth.
product_name = "codis-demo"
product_auth = "redis"

# Set bind address for admin(rpc), tcp only.
admin_addr = "0.0.0.0:18080"
```

启动

`codis-dashboard-admin.sh start`

3. 检查

`ss -ntplu |grep codis-dashboard`

![-w797](http://pic.itsuantou.com/15338053327325.jpg)

### 13、配置codis-fe

配置机器：
* 172.16.212.100

1. 修改启动脚本

`vim /usr/local/codis/src/github.com/CodisLabs/codis/admin/codis-fe-admin.sh`

启动前需要修改脚本zookeeper地址池

```
#COORDINATOR_NAME="filesystem"
#COORDINATOR_ADDR="/tmp/codis"
#CODIS_FE_ADDR="0.0.0.0:9090"
COORDINATOR_NAME="zookeeper"
COORDINATOR_ADDR="172.16.212.100:2181,172.16.212.101:2182,172.16.212.102:2183"
CODIS_FE_ADDR="0.0.0.0:9090"
```

2. 启动

`codis-fe-admin.sh start`

3. 检查

`netstat -tupnl |grep codis-fe`

![-w687](http://pic.itsuantou.com/15338059710023.jpg)

### 14、完成配置

至此codis的部署已经完成，请登录管理端进行配置使用
dashboard：http://172.16.212.100:9090

![](http://pic.itsuantou.com/15338064329972.jpg)

### 15、Codis常见问题修复

##### 1、codis-dashboard异常退出的修复

当codis-dashboard启动时，会在外部存储上存放一条数据，用于存储 dashboard 信息，同时作为LOCK存在。当codis-dashboard安全退出时，会主动删除该数据。

当codis-dashboard异常退出时（大多数情况zookeeper连接异常时会异常退出），由于之前LOCK未安全删除，重启往往会失败。因此codis-admin提供了强制删除工具：

1. 确认codis-dashboard进程已经退出（很重要）；
2. 然后运行codis-admin删除LOCK：

`codis-admin --remove-lock --product=codis_demo --zookeeper=172.16.212.100:2181`

正常关闭codis-dashboard：

`codisc -admin --dashboard=172.16.212.100:18080 --shutdown`

##### 2、codis-proxy异常退出的修复

通常codis-proxy都是通过codis-dashboard进行移除，移除过程中codis-dashboard为了安全会向codis-proxy发送offline指令，成功后才会将proxy 信息从外部存储中移除。如果codis-proxy异常退出，该操作会失败。此时可以使用codis-admin工具进行移除：

1. 确认codis-proxy进程已经退出（很重要）；
2. 运行codis-admin删除proxy，首先查看proxy状态

`codis-admin --dashboard=172.16.212.100:18080 --proxy-status`

3. 根据查看到的信息，强制删除报错的codis-proxy

`codis-admin --dashboard=172.16.212.100:18080 --remove-proxy --token=6a2db3c9ac07ba8857d4bc79ca6d191c  --force`

选项–force表示，无论offline操作是否成功，都从外部存储中将该节点删除。所以操作前，一定要确认该codis-proxy进程已经退出
4. codis-proxy正常关闭

`codis-admin --proxy=172.16.212.100:11080 --auth="xxxxx" --shutdown`

### 16、dashboard的使用
1、Proxy
添加codis-proxy地址和端口
1. 先添加codis-proxy的地址和管理端口，上面设置的是11080。
2. 点击左方的橙色按钮，然后就添加完毕。
3. 看到下方出现该有的codis-proxy地址就算完成了，然后看到右方的SYNC字样的颜色是绿色，则代表配置正常。
4. 如果要删除记录，点击最右方的红色按钮即可。

![](http://pic.itsuantou.com/15338071625266.jpg)

2、添加真实redis-server(也是codis-server)地址和端口
1. 先创建一个组，准备把相关的一组主从放进去
2. 点击按钮生成这个分组
3. 添加真实redis-server地址，并选定一个分组，例如刚才创建的分组1
4. 点击按钮生成配置
5. 可以看到配置已经登记好，注意sync状态
6. 点击重新平衡所有slots数据块(任何添加和删除新旧节点都需要点击这个)

在旧版本中slots需要手动配置，但是3.2版本之后就改成自动分配了，所以已经不需要配置，点击一下就可以了，当然你也可以手动去分配

![](http://pic.itsuantou.com/15338073231701.jpg)

3、配置sentinel的地址和端口
1. 添加真实的sentinel地址和端口
2. 点击按钮添加
3. 查看状态,这里有点不一样,他会自动添加当前主从组架构由多少台,控制切换

![](http://pic.itsuantou.com/15338075215499.jpg)
