# 前言
>文本已收录至我的GitHub仓库，欢迎Star：https://github.com/bin392328206/six-finger                             
> **种一棵树最好的时间是十年前，其次是现在**   
>我知道很多人不玩**qq**了,但是怀旧一下,欢迎加入六脉神剑Java菜鸟学习群，群聊号码：**549684836** 鼓励大家在技术的路上写博客

![](https://user-gold-cdn.xitu.io/2019/12/2/16ec5057ac2d21d6?w=640&h=459&f=jpeg&s=32759)

## 絮叨 
> 半步神游，神游之下，天下无敌。一梦一游 便是天下。         
> Redis前面几篇的文章链接：  
>[🔥从零开始学Redis之金刚凡境](https://juejin.im/post/5dde62bf5188256ebc1ee256)   
>[🔥从零开始学Redis之自在地境](https://juejin.im/post/5de24ca25188255e8b76e1c4)      
>[🔥从零开始学Redis之逍遥天境](https://juejin.im/post/5de391046fb9a0717220fafe)   
> 上一篇的逍遥天境 讲的是Redis的内存淘汰策略 和持久化方式。那这半步神游就是带你们遨游Redis的主从HA，哨兵，和Lua脚本

## Redis主从和哨兵模式

### Redis 主从搭建（有兴趣的小伙伴自己用虚拟机搭一个玩玩）
> 1、环境说明   

|主机名称|IP地址	|redis版本和角色说明|
|--|:--:|--:|
redis-master|192.168.56.11	|redis 5.0.3（主）
redis-slave01	|192.168.56.12	|redis 5.0.3（从）
redis-slave02	|192.168.56.13	|redis 5.0.3（从）

> 2、修改主从的redis配置文件

```
[root@redis-master ~]# grep -Ev "^$|#" /usr/local/redis/redis.conf 
bind 192.168.56.11
protected-mode yes
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
dir /var/redis/

[root@redis-slave01 ~]# grep -Ev "^$|#" /usr/local/redis/redis.conf
bind 192.168.56.12
protected-mode yes
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
dir /var/redis/
replicaof 192.168.56.11 6379    #配置为master的从，如果master上有密码配置，还需要增加下面一项密码配置
masterauth 123456   #配置主的密码

[root@redis-slave02 ~]# grep -Ev "^$|#" /usr/local/redis/redis.conf
bind 192.168.56.13
protected-mode yes
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/var/log/redis.log"
dir /var/redis/
replicaof 192.168.56.11 6379    #配置为master的从
masterauth 123456   #配置主的密码
```
> 3、启动主从redis
这里需要注意的是：redis主从和mysql主从不一样，redis主从不用事先同步数据，它会自动同步过去

```
[root@redis-master ~]# systemctl start redis
[root@redis-slave01 ~]# systemctl start redis
[root@redis-slave02 ~]# systemctl start redis
[root@redis-master ~]# netstat -tulnp |grep redis
tcp        0      0 192.168.56.11:6379      0.0.0.0:*               LISTEN      1295/redis-server 1 
[root@redis-slave01 ~]# netstat -tulnp |grep redis
tcp        0      0 192.168.56.12:6379      0.0.0.0:*               LISTEN      1625/redis-server 1 
[root@redis-slave02 ~]# netstat -tulnp |grep redis
tcp        0      0 192.168.56.13:6379      0.0.0.0:*               LISTEN      1628/redis-server 1 
```
> 4、数据同步验证

```
[root@redis-master ~]# redis-cli -h 192.168.56.11   #主上写入数据
192.168.56.11:6379> KEYS *
(empty list or set)
192.168.56.11:6379> set k1 123
OK
192.168.56.11:6379> set k2 456
OK

[root@redis-slave01 ~]# redis-cli -h 192.168.56.12  #slave01上查看是否数据同步
192.168.56.12:6379> KEYS *
1) "k2"
2) "k1"
192.168.56.12:6379> get k1
"123"
192.168.56.12:6379> get k2
"456"

[root@redis-slave02 ~]# redis-cli -h 192.168.56.13  #slave02上查看是否数据同步
192.168.56.13:6379> KEYS *
1) "k2"
2) "k1"
192.168.56.13:6379> get k1
"123"
192.168.56.13:6379> get k2
"456"
```

### 主从架构
主从架构的特点
- 主服务器负责接收写请求
- 从服务器负责接收读请求
- 从服务器的数据由主服务器复制过去。主从服务器的数据是一致的

主从架构的好处
- 读写分离(主服务器负责写，从服务器负责读)
- 高可用(某一台从服务器挂了，其他从服务器还能继续接收请求，不影响服务)
- 处理更多的并发量(每台从服务器都可以接收读请求，读QPS就上去了)

主从同步
> 主从架构的特点之一：主服务器和从服务器的数据是一致的。   
> 主从同步的2种情况

- 同步(sync)
    - 将从服务器的数据库状态更新至主服务器的数据库状态
- 命令传播(command propagate)
    - 主服务器的数据库状态被修改，导致主从服务器的数据库状态不一致，让主从服务器的数据库状态重新回到一致状态。
> 完整的同步
- 从服务器向主服务器发送PSYNC命令
- 收到PSYNC命令的主服务器执行BGSAVE命令，在后台生成一个RDB文件。并用一个缓冲区来记录从现在开始执行的所有写命令。
- 当主服务器的BGSAVE命令执行完后，将生成的RDB文件发送给从服务器，从服务器接收和载入RBD文件。将自己的数据库状态更新至与主服务器执行BGSAVE命令时的状态。
- 主服务器将所有缓冲区的写命令发送给从服务器，从服务器执行这些写命令，达到数据最终一致性。

> 部分重同步
- 主从服务器的复制偏移量 主服务器每次传播N个字节，就将自己的复制偏移量加上N
- 从服务器每次收到主服务器的N个字节，就将自己的复制偏移量加上N
- 通过对比主从复制的偏移量，就很容易知道主从服务器的数据是否处于一致性的状态！

### Redis HA 方案
> HA(High Available，高可用性群集)机集群系统简称，是保证业务连续性的有效解决方案，一般有两个或两个以上的节点，且分为活动节点及备用节点。通常把正在执 行业务的称为活动节点，而作为活动节点的一个备份的则称为备用节点。当活动节点出现问题，导致正在运行的业务（任务）不能正常运行时，备用节点此时就会侦测到，并立即接续活动节点来执行业务。从而实现业务的不中断或短暂中断。

> Redis 一般以主/从方式部署（这里讨论的应用从实例主要用于备份，主实例提供读写）该方式要实现 HA 主要有如下几种方案：
- keepalived： 通过 keepalived 的虚拟 IP，提供主从的统一访问，在主出现问题时， 通过 keepalived 运行脚本将从提升为主，待主恢复后先同步后自动变为主，该方案的好处是主从切换后，应用程序不需要知道(因为访问的虚拟 IP 不变)，坏处是引入 keepalived 增加部署复杂性，在有些情况下会导致数据丢失
- zookeeper： 通过 zookeeper 来监控主从实例， 维护最新有效的 IP， 应用通过 zookeeper 取得 IP，对 Redis 进行访问，该方案需要编写大量的监控代码
- sentinel： 通过 Sentinel 监控主从实例，自动进行故障恢复，该方案有个缺陷：因为主从实例地址( IP & PORT )是不同的，当故障发生进行主从切换后，应用程序无法知道新地址，故在 Jedis2.2.2 中新增了对 Sentinel 的支持，应用通过 redis.clients.jedis.JedisSentinelPool.getResource() 取得的 Jedis 实例会及时更新到新的主实例地址


### Redis Sentinel
Redis Sentinel是官方推荐的高可用性解决方案。Sentinel是一个管理多个Redis实例的工具，它可以实现对Redis的监控、通知、自动故障转移。

Sentinel的主要功能包括主节点存活检测、主从运行情况检测、自动故障转移（failover）、主从切换。Redis的Sentinel最小配置是一主一从。 Redis的Sentinel系统可以用来管理多个Redis服务器，该系统可以执行以下四个任务：
- 监控: Sentinel会不断的检查主服务器和从服务器是否正常运行。
- 通知: 当被监控的某个Redis服务器出现问题，Sentinel通过API脚本向管理员或者其他的应用程序发送通知。
- 自动故障转移: 当主节点不能正常工作时，Sentinel会开始一次自动的故障转移操作，它会将与失效主节点是主从关系的其中一个从节点升级为新的主节点， 并且将其他的从节点指向新的主节点。
- 配置提供者: 在Redis Sentinel模式下，客户端应用在初始化时连接的是Sentinel节点集合，从中获取主节点的信息。

### Redis Sentinel的工作流程

![](https://user-gold-cdn.xitu.io/2019/12/2/16ec565266f116fa?w=660&h=266&f=jpeg&s=68734)

![](https://user-gold-cdn.xitu.io/2019/12/2/16ec565db3415ae9?w=551&h=391&f=png&s=75556)

如图所示：由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器，以及所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求 

Sentinel负责监控集群中的所有主、从Redis，当发现主故障时，Sentinel会在所有的从中选一个成为新的主。并且会把其余的从变为新主的从。同时那台有问题的旧主也会变为新主的从，也就是说当旧的主即使恢复时，并不会恢复原来的主身份，而是作为新主的一个从。

在Redis高可用架构中，Sentinel往往不是只有一个，而是有3个或者以上。目的是为了让其更加可靠，毕竟主和从切换角色这个过程还是蛮复杂的。这个是所有分布式系统都要碰到的问题一个是崩溃恢复 一个是数据同步 下面我们就来聊聊

### Redis HA方案的 崩溃恢复 和数据同步
#### 崩溃恢复
> 这个是所有分布式系统的问题 什么就是崩溃恢复呢 就我们目前的方案来说 我们是用sentinel 来保证redis的高可用 同时 sentinel 本身自己也做了HA 假设一主俩从的情况下 如果主节点挂了 怎么办，这就能造成单点故障 让整个redis 集群不可用，所以 崩溃恢复就是类似于一个Zookeeper的ZAB 算法 从其他节点中选举一个主节点，具体怎么选举一个新的主节点,这边就不扩展了，再说下去，这篇就扯不完了。
#### 数据同步

> redis的主从复制  
    >依赖于redis依赖于RDB模式下的持久化存储；采用复制RDB文件的形式进行主从节点之间的数据同步     
    >注意：  主从复制时不要开启AOF持久化模式，因为AOF优先级高于RDB模式   
    
> RDB文件两种传输方法   
    1.普通复制    
 将主节点已经到磁盘上的的ROB文件，复制到从节点上    
    2.无盘复制    
   master端直接将RDB file传到slave socket，不需要与disk进行交互    
   无磁盘diskless方式适合磁盘读写速度慢但网络带宽非常高的环境   
##  Redis Sentinel 搭建（可以自己试试）
1.环境说明

|主机名称|IP地址	|redis版本和角色说明|
|--|:--:|:--|
redis-master|192.168.56.11:6379	|redis 5.0.3（主）
redis-slave01	|192.168.56.12:6379	|redis 5.0.3（从）
redis-slave02	|192.168.56.13:6379	|redis 5.0.3（从）
redis-master|192.168.56.11:26379	|Sentinel01(主)
redis-slave01	|192.168.56.12:26379	|Sentinel02(从)
redis-slave02	|192.168.56.13:26379	|Sentinel03(从)


2.部署Sentinel

```
# 端口
port 26379

# 是否后台启动
daemonize yes

# pid文件路径
pidfile /var/run/redis-sentinel.pid

# 日志文件路径
logfile "/var/log/sentinel.log"

# 定义工作目录
dir /tmp

# 定义Redis主的别名, IP, 端口，这里的2指的是需要至少2个Sentinel认为主Redis挂了才最终会采取下一步行为
sentinel monitor mymaster 127.0.0.1 6379 2

# 如果mymaster 30秒内没有响应，则认为其主观失效
sentinel down-after-milliseconds mymaster 30000

# 如果master重新选出来后，其它slave节点能同时并行从新master同步数据的台数有多少个，显然该值越大，所有slave节点完成同步切换的整体速度越快，但如果此时正好有人在访问这些slave，可能造成读取失败，影响面会更广。最保守的设置为1，同一时间，只能有一台干这件事，这样其它slave还能继续服务，但是所有slave全部完成缓存更新同步的进程将变慢。
sentinel parallel-syncs mymaster 1

# 该参数指定一个时间段，在该时间段内没有实现故障转移成功，则会再一次发起故障转移的操作，单位毫秒
sentinel failover-timeout mymaster 180000

# 不允许使用SENTINEL SET设置notification-script和client-reconfig-script。
sentinel deny-scripts-reconfig yes
```
修改三台Sentinel的配置文件，如下

```
[root@redis-master ~]# grep -Ev "^$|#" /usr/local/redis/sentinel.conf 
port 26379
daemonize yes
pidfile "/var/run/redis-sentinel.pid"
logfile "/var/log/sentinel.log"
dir "/tmp"
sentinel monitor mymaster 192.168.56.11 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
sentinel deny-scripts-reconfig yes

[root@redis-slave01 ~]# grep -Ev "^$|#" /usr/local/redis/sentinel.conf 
port 26379
daemonize yes
pidfile "/var/run/redis-sentinel.pid"
logfile "/var/log/sentinel.log"
dir "/tmp"
sentinel monitor mymaster 192.168.56.11 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
sentinel deny-scripts-reconfig yes

[root@redis-slave02 ~]# grep -Ev "^$|#" /usr/local/redis/sentinel.conf 
port 26379
daemonize yes
pidfile "/var/run/redis-sentinel.pid"
logfile "/var/log/sentinel.log"
dir "/tmp"
sentinel monitor mymaster 192.168.56.11 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
sentinel deny-scripts-reconfig yes
```
3.启动Sentinel
启动的顺序：主Redis --> 从Redis --> Sentinel1/2/3

```
[root@redis-master ~]# redis-sentinel /usr/local/redis/sentinel.conf 
[root@redis-master ~]# ps -ef |grep redis
root      1295     1  0 14:03 ?        00:00:06 /usr/local/redis/src/redis-server 192.168.56.11:6379
root      1407     1  1 14:40 ?        00:00:00 redis-sentinel *:26379 [sentinel]
root      1412  1200  0 14:40 pts/1    00:00:00 grep --color=auto redis

[root@redis-slave01 ~]# redis-sentinel /usr/local/redis/sentinel.conf 
[root@redis-slave01 ~]# ps -ef |grep redis
root      1625     1  0 14:04 ?        00:00:06 /usr/local/redis/src/redis-server 192.168.56.12:6379
root      1715     1  1 14:41 ?        00:00:00 redis-sentinel *:26379 [sentinel]
root      1720  1574  0 14:41 pts/0    00:00:00 grep --color=auto redis

[root@redis-slave02 ~]# redis-sentinel /usr/local/redis/sentinel.conf 
[root@redis-slave02 ~]# ps -ef |grep redis
root      1628     1  0 14:07 ?        00:00:06 /usr/local/redis/src/redis-server 192.168.56.13:6379
root      1709     1  0 14:42 ?        00:00:00 redis-sentinel *:26379 [sentinel]
root      1714  1575  0 14:42 pts/0    00:00:00 grep --color=auto redis
```
4.Sentinel操作

```
[root@redis-master ~]# redis-cli -p 26379   #哨兵模式查看
127.0.0.1:26379> sentinel master mymaster   #输出被监控的主节点的状态信息
 1) "name"
 2) "mymaster"
 3) "ip"
 4) "192.168.56.11"
 5) "port"
 6) "6379"
 7) "runid"
 8) "bae06cc3bc6dcbff7c2de1510df7faf1a6eb6941"
 9) "flags"
10) "master"
......
127.0.0.1:26379> sentinel slaves mymaster   #查看mymaster的从信息，可以看到有2个从节点
1)  1) "name"
    2) "192.168.56.12:6379"
    3) "ip"
    4) "192.168.56.12"
    5) "port"
    6) "6379"
    7) "runid"
    8) "c86027e7bdd217cb584b1bd7a6fea4ba79cf6364"
    9) "flags"
   10) "slave"
......
2)  1) "name"
    2) "192.168.56.13:6379"
    3) "ip"
    4) "192.168.56.13"
    5) "port"
    6) "6379"
    7) "runid"
    8) "61597fdb615ecf8bd7fc18e143112401ed6156ec"
    9) "flags"
   10) "slave"
......

127.0.0.1:26379> sentinel sentinels mymaster    #查看其它sentinel信息
1)  1) "name"
    2) "ba12e2a4023d2e9bcad282395ba6b14030920070"
    3) "ip"
    4) "192.168.56.12"
    5) "port"
    6) "26379"
    7) "runid"
    8) "ba12e2a4023d2e9bcad282395ba6b14030920070"
    9) "flags"
   10) "sentinel"
......
2)  1) "name"
    2) "14fca3f851e9e1bd3a4a0dc8a9e34bb237648455"
    3) "ip"
    4) "192.168.56.13"
    5) "port"
    6) "26379"
    7) "runid"
    8) "14fca3f851e9e1bd3a4a0dc8a9e34bb237648455"
    9) "flags"
   10) "sentinel"
```

## Redis Lua 脚本
> 再这里我想说下，为啥我们要用Lua脚本呢？ Lua脚本的好处   
> redis对lua脚本的调用是原子性的，所以一些特殊场景，比如像实现分布式锁，我们可以放在lua中实现  
> 下面我带大家搭建一个最简单lua脚本demo
- 添加依赖

```
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
         <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
```
- 编写Lua脚本 命名为 Test.lua 放在 resources下

```
local key = KEYS[1]
    --- 获取value
    local val = KEYS[2]
    --- 获取一个参数
    local expire = ARGV[1]
    --- 如果redis找不到这个key就去插入
    if redis.call("get", key) == false then
        --- 如果插入成功，就去设置过期值
        if redis.call("set", key, val) then
            --- 由于lua脚本接收到参数都会转为String，所以要转成数字类型才能比较
            if tonumber(expire) > 0 then
                --- 设置过期时间
                redis.call("expire", key, expire)
            end
            return true
        end
        return false
    else
        return false
    end
```
- 编写配置类

```
@Configuration
public class LuaConfiguration {
    @Bean
    public DefaultRedisScript<Boolean> redisScript() {
        DefaultRedisScript<Boolean> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("Test.lua")));
        redisScript.setResultType(Boolean.class);
        return redisScript;
    }
}

```
- 测试

```
    @Test
    public void TestLua(){
        System.out.println("测试Lua开始");
        List<String> keys = Arrays.asList("testLua", "hello六脉神剑");
        Boolean execute = stringRedisTemplate.execute(redisScript, keys, "10000");
        System.out.println("测试Lua结束，并在下面打印结果");
        String testLua = stringRedisTemplate.opsForValue().get("testLua");
        System.out.println("结果是:"+testLua);
    }

```

- 结果

```
测试Lua开始
2019-12-03 10:48:01.469  INFO 246868 --- [           main] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2019-12-03 10:48:01.471  INFO 246868 --- [           main] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
测试Lua结束，并在下面打印结果
结果是:hello六脉神剑

```
### Redis使用Lua的好处
> 1.减少网络开销：本来5次网络请求的操作，可以用一个请求完成，原先5次请求的逻辑放在redis服务器上完成。使用脚本，减少了网络往返时延。

> 2.原子操作：Redis会将整个脚本作为一个整体执行，中间不会被其他命令插入。

> 3.复用：客户端发送的脚本会永久存储在Redis中，意味着其他客户端可以复用这一脚本而不需要使用代码完成同样的逻辑。
### Redis使用Lua要注意的点

> 1.Lua脚本的bug特别可怕，由于Redis的单线程特点，一旦Lua脚本出现不会返回（不是返回值）得问题，那么这个脚本就会阻塞整个redis实例。

> 2.Lua脚本应该尽量短小实现关键步骤即可。（原因同上）

> 3.Lua脚本中不应该出现常量Key，这样会导致每次执行时都会在脚本字典中新建一个条目，应该使用全局变量数组KEYS和ARGV, KEYS和ARGV的索引都从1开始

> 4.传递给lua脚本的的键和参数：传递给lua脚本的键列表应该包括可能会读取或者写入的所有键。传入全部的键使得在使用各种分片或者集群技术时，其他软件可以在应用层检查所有的数据是不是都在同一个分片里面。另外集群版redis也会对将要访问的key进行检查，如果不在同一个服务器里面，那么redis将会返回一个错误。（决定使用集群版之前应该考虑业务拆分），参数列表无所谓。。

> 5.lua脚本跟单个redis命令和事务段一样都是原子的已经进行了数据写入的lua脚本将无法中断，只能使用SHUTDOWN NOSAVE杀死Redis服务器，所以lua脚本一定要测试好。




## 结尾
> 我擦就随便写了个主从和Lua，就这么多，哎，一把辛酸一把泪。下一章是最后一章了，看看怎么写吧。

> 因为博主也是一个开发萌新 我也是一边学一边写 我有个目标就是一周 二到三篇 希望能坚持个一年吧 希望各位大佬多提意见，让我多学习，一起进步。
## 日常求赞
> 好了各位，以上就是这篇文章的全部内容了，能看到这里的人呀，都是**人才**。

> 创作不易，各位的支持和认可，就是我创作的最大动力，我们下篇文章见

>六脉神剑 | 文 【原创】如果本篇博客有任何错误，请批评指教，不胜感激 ！
