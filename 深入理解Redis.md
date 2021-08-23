## 深入理解Redis

[toc]

### 1. 阻塞IO与非阻塞IO

java在JDK1.4中引入NIO,但是也有很多人习惯于使用阻塞IO,这两者有什么区别?

在阻塞模式下,如果在流中读取不到指定大小的数据量,IO就会阻塞,例如已知会有10个字节大小的数据需要被接收,但是目前只接受到4个字节,此时数据量不足10个字节,就会发送阻塞.如果是在非阻塞IO中虽然只接收到了4个字节的数据仍然会将数据返回,等剩余数据发送过来的时候再继续去接收.

显然阻塞IO的性能会低于非阻塞IO.

> 如果有一个Web服务器,使用阻塞IO来处理请求,那么每次接收到一个请求都需要开启一个新的线程.但是如果使用的是非阻塞IO基本上只需要使用几个线程池即可满足需求,因为不会发生阻塞,所以对线程的利用相当高效.

### 2. Redis 中的线程模型

**Redis 是一个单线程程序** 单线程如何解决高并发问题?

实际上,能够处理高并发的单线程应用,不仅仅只有Redis,像nodeJs,Nginx等等都是单线程的.

Redis虽然是单线程,但是运行很快,主要有如下几个原因:

1. Redis中的所有数据都是基于内存,所有计算也都是内存级别的计算

2. Redis是单线程的,所以有些时间复杂度高的指令可能会导致Redis卡顿,例如Keys

3. Redis再处理并发的客户端连接时使用的是非阻塞IO

   在使用非阻塞IO时需要解决的问题就是,线程如何知道剩余的数据过来了

   这里需要引入一个叫做**多路复用**的概念,本质上是一个事件的轮询API.这个是由操作系统所提供的

4. Redis会给每一个客户端指令通过队列来排队进行队列处理

5. Redis做出响应的时候,也会有一个响应队列,通过响应队列来将数据返回给用户

### 3. Redis 通信协议

Redis使用的是文本协议,文本协议会比较消耗流量,Redis的作者认为数据库的瓶颈不在于网络协议,而是在于内部逻辑.所以采用的是一个比较费流量的协议

这个文本协议叫做Redis Serialization Protocol简称RESP

Redis 协议将传输的数据结构分为5种最小单元,单元结束时加上回车换行符\r\n

1. 单行字符串以"+"开始 例如: +leanRedis\r\n
2. 多行字符串以"$"开始,后面跟上字符串长度  例如 $9\r\n leanRedis\r\n
3. 整数值以":"开始 例如 :1024\r\n
4. 错误消息以"-"开始
5. 数组以"*"开始,后面加上数组长度

tips: 如果是以客户端连接服务端,只能使用第五种方式进行传输数据

### 4. Redis 持久化

Redis是一个缓存工具,也叫做NoSQL数据库,既然是数据库,一定支持数据持久化操作.在Redis中数据持久化一共有两种方案:

1. 快照
2. AOF日志

#### 4.1 快照

##### 4.1.1基本原理

Redis 使用操作系统的多进程机制来实现快照持久化.Redis在持久化时,会调用Glibc函数 fork一个子进程,然后将快照持久化操作完全交给子进程,而父进程则继续处理客户端请求.在这个过程中子进程能够看到内存中的数据在子进程产生的一瞬间就固定了,再也不能改变.这也就是为什么Redis 持久化叫快照.

##### 4.1.2 基本配置

在redis 中默认情况下快照持久化的方式就是开启的.

默认情况下会生成一个叫dump.rdb文件,这个就是备份下来的文件.当Redis启动时,会自动加载该文件去恢复数据.

具体配置在redis.conf中:

![image-20210813105202146](https://gitee.com/sixiwanzia/img/raw/master/20210813105209.png)

这里表示的是快照备份的频率,第一个表示在900s之内如果有一个键被修改,则进行快照

![image-20210813105316272](https://gitee.com/sixiwanzia/img/raw/master/20210813105316.png)

这里表示的是快照执行出错,是否继续执行客户端写命令

![image-20210813105421847](https://gitee.com/sixiwanzia/img/raw/master/20210813105421.png)

这里表示是否对快照文件进行压缩

![](https://gitee.com/sixiwanzia/img/raw/master/20210813105527.png)

这里表示的是快照文件的名称

![image-20210813105652223](https://gitee.com/sixiwanzia/img/raw/master/20210813105652.png)

这里表示的文件的快照文件的存放路径

##### 4.1.3 备份流程

1. 在Redis运行过程中,我们可以通过向Redis中发送一条save命令创建一个快照.但是需要注意,save是一条阻塞命令,Redis在收到save命令开始处理备份操作之后,在处理完成之前将不会在去处理其他的请求.其他的命令都将会被挂起.
2. 我们一般可以使用bgsave命令,bgsave会fork一个子进程去处理备份的事情,不会影响父进程处理其他的请求.
3. 我们定义的备份规则,如果有规则满足也会自动触发bgsave命令
4. 另外当我执行shutdown命令,也会触发save命令.备份工作完成后,Redis才会关闭
5. Reids搭建主从复制时在从机连接上主机之后会自动发送一条 sync 同步命令,在主机收到命令后首先执行bgsave命令,生成快照完成后,才会将从机中的快照数据进行同步

#### 4.2 AOF

##### 4.2.1基本介绍

与快照持久化不同,这个AOF持久化是将被执行的命令追加到AOF文件末尾,在恢复时只需要将记录下来的命令从头到尾执行一遍即可.

##### 4.2.2 具体配置

默认情况下时没有开启的,需要手动开启.具体配置在redis.conf中:

![](https://gitee.com/sixiwanzia/img/raw/master/20210813111925.png)

这里表示开启aof配置,这个要使用的化需要将这个改为yes

![image-20210813111748351](https://gitee.com/sixiwanzia/img/raw/master/20210813111748.png)

这里配置的时aof文件的名字

![image-20210813110944102](https://gitee.com/sixiwanzia/img/raw/master/20210813110944.png)

这里配置的是备份的时机

```
appendfsync always #每条命令都进行备份

appendfsync everysec # 每秒备份一次

appendfsync no #不备份
```

![image-20210813111232406](https://gitee.com/sixiwanzia/img/raw/master/20210813111232.png)

因为AOF文件是文本文档,所以所占的空间会比较大.解决方案是每个一断时间就进行一次压缩,将文件变小一些.这里配置的就是当aof文件在进行压缩的时候是否继续进行同步操作

![image-20210813111446459](https://gitee.com/sixiwanzia/img/raw/master/20210813111446.png)

这里配置的时AOF文件压缩的时机

```
auto-aof-rewrite-percentage 100 #表示当目前的aof文件大小操作上一次重写时aof文件大小百分之多少的时候再次进行重写 
auto-aof-rewrite-min-size 64mb#这个表示如果之前没有重写过则已启动时AOF大小为基准,同时要求至少要大于在此处配置的大小
```

在开启aof备份方式的时候需要将快照备份的方式关闭

![image-20210813112146253](https://gitee.com/sixiwanzia/img/raw/master/20210813112146.png)

可以手动发送一条命令来进行备份``` bgrewriteaof``` 原理和```bgsave```差不多

### 5. Redis 事务

正常来说,一个可以商用的、功能完善的数据库都会具备非常完善的事务支持,Redis当然也支持.相对于关系型数据库中的事务模型,Redis中的事务就要显得简单许多.在使用的时候无需理解那么复杂的事务模型,当然简单也就意味着Redis中的事务模型不太严格,所以我们不能像在关系型数据库中使用事务那样在Redis中使用事务.

在关系型数据库中和事务相关的三个指令

- begin
- commit
- rollback

在Redis中也有对应的指令

- mulit
- exec
- discard

#### 5.1 Redis 事务特性

##### 5.1.1 原子性

在Redis 中的事务其实不能算作原子性如图所示,我们开启事务后一共执行了4条命令,其中有一条命令明显错误```incr k1``` 但是在提交事务之后在这条命令后面的set k3 v3 还是被执行了.这里就体现了它并不具备原子性仅仅只具备隔离性,也就是说当前的事务不会被其他事务打断.

![image-20210816112447976](https://gitee.com/sixiwanzia/img/raw/master/20210816112455.png)

由于每次事务操作涉及到的指令相对较多,为了提高执行效率,我们在使用客户端的时候我们可以使用pipeline来优化指令操作.

##### 5.1.2 乐观锁

Redis中有一个指令叫watch,watch指令可以用来监控一个key,通过这种监控,我们可以确保在事务提交之前watch监控的键的值没有被修改过.

#### 5.2 Java代码实现

```java
  public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(jedis -> {
            new TransactionTest().saveMoney(jedis,"leanRedis",1000);
        });
    }

    public Integer saveMoney(Jedis jedis, String user, Integer money) {

        while (true) {
            jedis.watch(user);
            String s = jedis.get(user);
            String ss = s==null?"0":s;
            int v = Integer.parseInt(ss) + money;
            Transaction transaction = jedis.multi();
            transaction.set(user, String.valueOf(v));
            List<Object> exec = transaction.exec();
            if (exec != null) {
                break;
            }
        }
        return Integer.parseInt(jedis.get(user));
    }
```

### 6.Redis主从同步

#### 6.1 CAP原理

在分布式环境下,CAP原理是一个非常常见的东西,所有的分布式存储系统都只能在CAP中选择两项实现.

C: consistent一致性

A: availability 可用性

P: partition tolerance 分布式容忍性

在一个分布式存储系统中上述三个只能选择其中两个满足. 在一个分布式系统中P是必定要被实现的,C和A只能选择其中一个来实现,当对数据的一致性要求较高的情况下可用性必然不能兼顾反之亦然.例如如下场景:

> 假设我们的分布式存储系统主机架构在广州,从机架构在深圳,从机从主机上复制数据,当两地之间的连接突然断开的时候,如果要满足可用性,那么此时从机是必然要提供数据服务的,但是此时的从机和主机之间的数据会存在延迟,也就不能满足数据的一致性.反之如果要满足数据的一致性,在主机和从机连接断开的时候主机和从机就不会提供服务,等到连接恢复后再提供服务,这样就保持了数据的一致性.这样就满足不了可用性了

在大部分情况下网站架构都选择的是AP.牺牲了一致性.进而使用了别的东西进行代替,在Redis中会保持最终一致.

Redis 中当搭建了主从服务之后,如果主从之间的连接断开了,Redis依然可以进行操作,相当于满足了可用性,但是此时主从之间的数据会有差异,相当于牺牲了一致性,但是Redis保证最终一致,也就是说当网络恢复后从机会追赶主机,尽量保持数据的一致.

#### 6.2 主从复制

主从复制可以在一定程度上扩展 redis 性能，redis 的主从复制和关系型数据库的主从复制类似，从机 能够精确的复制主机上的内容。实现了主从复制之后，一方面能够实现数据的读写分离，降低master的 压力，另一方面也能实现数据的备份。

##### 6.2.1 配置方式

假设我们有三个Redis实例,地址分别如下:

```
127.0.0.1:6379
127.0.0.1:6380
127.0.0.1:6381
```

即同一台服务器上三个实例，配置方式如下：

 1. 将 redis.conf 文件更名为 redis6379.conf，方便我们区分，然后把 redis6379.conf 再复制两份， 分别为 redis6380.conf 和 redis6381.conf。如下

<img src="https://gitee.com/sixiwanzia/img/raw/master/20210816161718.png" alt="image-20210816161718542" style="zoom:50%;" />

2. 打开 redis6379.conf，将如下配置均加上 6379,(默认是6379的不用修改)，如下：

```
port 6379 
pidfile /var/run/redis_6379.pid
logfile "6379.log" 
dbfilename dump6379.rdb
appendfilename "appendonly6379.aof"
```

3. 同理，分别打开 redis6380.conf 和 redis6381.conf 两个配置文件，将第二步涉及到 6379 的分别 改为 6380 和 6381。
4. 打开三个redis实例

```
[root@localhost redis-6.0.9]# redis-server redis6379.conf 
[root@localhost redis-6.0.9]# redis-server redis6380.conf 
[root@localhost redis-6.0.9]# redis-server redis6381.conf 
```

5.  输入如下命令，分别进入三个实例的控制台：

```
[root@localhost redis-6.0.9]# redis-cli -p 6379 -a "redis密码" #如果有密码的话需要加上密码
[root@localhost redis-6.0.9]# redis-cli -p 6379  -a "redis密码" #如果有密码的话需要加上密码
[root@localhost redis-6.0.9]# redis-cli -p 6379  -a "redis密码" #如果有密码的话需要加上密码
```

6. 假设在这三个实例中，6379 是主机，即 master，6380 和 6381 是从机，即 slave，那么如何配 置这种实例关系呢，很简单，分别在 6380 和 6381 上执行如下命令：

`````
127.0.0.1:6381> SLAVEOF 127.0.0.1 6379
`````

tips: 如果主机配置了密码那么需要在从机的配置文件中修改配置,将主机认证的密码添加进去否则无法连接主机

![image-20210816160607250](https://gitee.com/sixiwanzia/img/raw/master/20210816160607.png)

7. 由于一个Redis实例启动时默认是作为主机启动的,如果需要改为默认作为从机启动需要在redis的配置文件中下图位置进行修改.

![image-20210816161137287](https://gitee.com/sixiwanzia/img/raw/master/20210816161137.png)

至此,主从关系就搭建好了.我们可以使用如下命令来查看每个实例当前的状态

```
127.0.0.1:6379> INFO replication
# Replication
role:master
connected_slaves:0
master_replid:34fa30dcc3883b3b3b80bca5773b9ff338fca198
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

这里可以看到6379是作为主机拥有两个从机.如果是在两个从机上执行该命令则会看到如下信息:

```
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:3
master_sync_in_progress:0
slave_repl_offset:14
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:88d9b14e3c2348dd200b64399a598053a4a0b406
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:14
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:14
```

我们可以看到 6380 是一个从机，从机的信息以及它的主机的信息都展示出来了。 

7. 此时，我们在主机中存储一条数据，在从机中就可以 get 到这条数据了。 

##### 6.2.2 主从复制注意事项

1.  如果主机已经运行了一段时间了，并且了已经存储了一些数据了，此时从机连上来，那么从机会将 主机上所有的数据进行备份，而不是从连接的那个时间点开始备份这点和我们常用的mysql的主从备份有区别。
2.  配置了主从复制之后，主机上可读可写，但是从机只能读取不能写入（可以通过修改redis.conf  中 replica-read-only 的值让从机也可以执行写操作）。
3. 在整个主从结构运行过程中，如果主机不幸挂掉，重启之后，他依然是主机，主从复制操作也能够继续进行。 

##### 6.2.3复制原理

每一个 master 都有一个 replication ID，这是一个较大的伪随机字符串，标记了一个给定的数据集。每 个 master 也持有一个偏移量，master 将自己产生的复制流发送给 slave 时，发送多少个字节的数据， 自身的偏移量就会增加多少，目的是当有新的操作修改自己的数据集时，它可以以此更新 slave 的状 态。复制偏移量即使在没有一个 slave 连接到 master 时，也会自增，所以基本上每一对给定的 Replication ID, offset 都会标识一个 master 数据集的确切版本。当 slave 连接到 master 时，它们使用 PSYNC 命令来发送它们记录的旧的 master replication ID 和它们至今为止处理的偏移量。通过这种方 式，master 能够仅发送 slave 所需的增量部分。但是如果 master 的缓冲区中没有足够的命令积压缓冲 记录，或者如果 slave 引用了不再知道的历史记录（replication ID），则会转而进行一个全量重同步： 在这种情况下，slave 会得到一个完整的数据集副本，从头开始(参考redis官网)。  简单来说，就是以下几个步骤：  

1. slave 启动成功连接到 master 后会发送一个 sync 命令。 
2. Master 接到命令启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令。 
3.  在后台进程执行完毕之后，master 将传送整个数据文件到 slave,以完成一次完全同步。 
4.  全量复制：而 slave 服务在接收到数据库文件数据后，将其存盘并加载到内存中。 
5. 增量复制：Master 继续将新的所有收集到的修改命令依次传给 slave,完成同步。 
6. 但是只要是重新连接 master,一次完全同步（全量复制)将被自动执行。 本文接上文，所用三个 redis 实例和上文一致，这里就不再赘述三个实例搭建方式。

##### 6.2.4 接力赛

之前我们的主从模式是一个主机对应多个从机的复制,也就是一主多仆的复制模式,所有从机都直接从主机上进行数据的复制,在从机数量少的情况下这样是不会有问题,但是当从机数量多起来的时候主机的压力就会变得很大,这种场景下建议使用接力赛的模式来进行主从复制,我们可以让从机a作为从机b的主机,从机b不直接从主机复制数据转而从从机a中复制数据,这样会减小主机的压力.

##### 6.2.5 哨兵模式

我们一共介绍了两种主从模式了，但是这两种，不管是哪一种，都会存在这样一个问 题，那就是当主机宕机时，就会发生群龙无首的情况，如果在主机宕机时，能够从从机中选出一个来充 当主机，那么就不用我们每次去手动重启主机了，这就涉及到一个新的话题，那就是哨兵模式。 所谓的哨兵模式，其实并不复杂，我们还是在我们前面的基础上来搭建哨兵模式。假设现在我的 master 是 6379，两个从机分别是 6380 和 6381，两个从机都是从 6379 上复制数据。先按照上文的步 骤，我们配置好一主二仆，然后在 redis 目录下打开 sentinel.conf 文件，做如下配置：

```
sentinel monitor mymaster 127.0.0.1 6379 1
```

其中 mymaster 是给要监控的主机取的名字，随意取，后面是主机地址，最后面的 1 表示有多少个 sentinel 认为主机挂掉了，就进行切换（我这里只有一个，因此设置为1）。好了，配置完成后，输入 如下命令启动哨兵：

```
redis-sentinel sentinel.conf
```

这种启动方式是前台启动的

如果我们的redis配置了密码那么同样的需要在sentinel.conf文件离进行配置

```
sentinel auth-pass mymaster admin01*
```

当我们配置了哨兵之后,如果主机宕机了,在就算是主机重启之后也不会成为主机会由原本的主机转而变为从机
#### 6.3 Jedis操作哨兵模式

准备工作:

1. 因为所有实例都有可能升级成为主机也有可能降级成从机,因此所有实例均需要配置masterauth属性
2. 所有实例均需绑定地址 bind 实际ip地址,不要写成 bind 127.0.0.1这种

在哨兵的配置中监控的master也不要配置成127.0.0.1,需要写成局域网中的IP地址

准备工作完成后启动三个redis实例,同时启动哨兵

```java
  public static void main(String[] args) {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMaxTotal(10);
        jedisPoolConfig.setMaxWaitMillis(1000);
        String master ="mymaster";
        Set<String> sentinel= new HashSet<>();
        sentinel.add("192.168.68.131:26379");
        JedisSentinelPool sentinelPool = new JedisSentinelPool(master, sentinel,
                jedisPoolConfig, "admin01*");
        Jedis jedis = null;
        while (true){
            try {
                jedis= sentinelPool.getResource();
                String k1 = jedis.get("k1");
                System.out.println(k1);
            }catch (Exception exception){

            }finally {
                if (jedis!=null)
                    jedis.close();
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }

    }
```

#### 6.4 Spring Boot操作哨兵模式

添加Spring Boot依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

配置Redis连接

```yml
spring:
  redis:
    password: admin01*
    timeout: 5000
    sentinel:
      master: mymaster
      nodes: 192.168.68.131:26379

```

测试代码

```java

@SpringBootTest
class SentinelApplicationTests {

    @Autowired
    StringRedisTemplate redisTemplate;
    @Test
    void contextLoads() {
        while (true){
             try {
                 String k1 =redisTemplate.opsForValue().get("k1");
                 System.out.println(k1+ "时间"+new  Date().getTime());
             }catch (Exception e){

             }finally {
                 try {
                     Thread.sleep(5000);
                 } catch (InterruptedException e) {
                     e.printStackTrace();
                 }
             }
        }
    }

}
```

### 7. Redis 集群

Redis 集群架构如下图：

![image-20210818092351194](https://gitee.com/sixiwanzia/img/raw/master/20210818092358.png)

Redis 集群运行原理如下：

1. 所有的 Redis 节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽 
2. 节点的 fail 是通过集群中超过半数的节点检测失效时才生效  
3. 客户端与 Redis 节点直连,不需要中间 proxy 层，客户端不需要连接集群所有节点，连接集群中任 何一个可用节点即可 
4.  Redis-cluster 把所有的物理节点映射到 [0-16383]slot 上,cluster (簇)负责维护 node<->slot<- >value 。Redis 集群中内置了 16384 个哈希槽，当需要在 Redis 集群中放置一个key-value 时， Redis 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会 对应一个编号在 0-16383 之间的哈希槽，Redis 会根据节点数量大致均等的将哈希槽映射到不同 的节点

#### 7.1 投票机制

投票过程是集群中所有 master 参与,如果半数以上 master 节点与 master 节点通信超过 clusternode-timeout 设置的时间,认为当前 master 节点挂掉

#### 7.2 掉线判断

1. 如果集群任意 master 挂掉,且当前 master 没有 slave.集群进入 fail 状态,也可以理解成集群的 slot  映射 [0-16383] 不完整时进入 fail 状态。
2. 如果集群超过半数以上 master 挂掉，无论是否有 slave,集群进入 fail 状态，当集群不可用时,所有 对集群的操作做都不可用，收到((error) CLUSTERDOWN The cluster is down)错误。

#### 7.3 集群搭建

##### 7.3.1 集群部署

首先我们对集群做一个简单规划，假设我的集群中一共有三个节点，每个节点一个主机一个从机，这样 我一共需要 6 个 Redis 实例。首先创建 redis-cluster 文件夹，在该文件夹下分别创建 7001、7002、 7003、7004、7005、7006 文件夹，用来存放我的 Redis 配置文件，如下：

![image-20210823093516321](https://gitee.com/sixiwanzia/img/raw/master/20210823093554.png)

将Redis也在Redis-cluster文件夹下安装一份,然后将redis.conf文件拷贝到7001-7006这几个文件夹里面.拷贝完成后分别修改如下内容:

```
port 7001 #7001文件夹就写7001,7002文件夹就写7002
bind 192.168.68.131 #这里改为局域网内的Ip地址
cluster-enabled yes#开启集群
cluster-config-XXXXX7001.conf#7001文件夹下就写7001,7002文件夹下就写7002
protected no
daemonize yes#后台启动
```

修改完成后，进入到 redis 安 装目录中，分别启动各个 redis，使用刚刚修改过的配置文件，如下:

![](https://gitee.com/sixiwanzia/img/raw/master/20210823094346.png)

启动完成之后可以验证一下是否启动成功,如下:

![image-20210823093516322](https://gitee.com/sixiwanzia/img/raw/master/20210823094324.png)

这个表示各个节点都启动成功了。接下来我们就可以进行集群的创建了，首先将 redis/src 目录下的 redis-trib.rb 文件拷贝到 redis-cluster 目录下，然后在 redis-cluster 目录下执行如下命令：

```
redis-cli --cluster create 192.168.68.131:7001 192.168.68.131:7002 192.168.68.131:7003 192.168.68.131:7004 192.168.68.131:7005 192.168.68.131:7006
```

![image-20210823095855938](https://gitee.com/sixiwanzia/img/raw/master/20210823095856.png)

##### 7.3.2 查看集群信息

集群创建成功后，我们可以登录到 Redis 控制台查看集群信息，注意登录时要添加 -c 参数，表示以集 群方式连接，如下：

![image-20210823100616568](https://gitee.com/sixiwanzia/img/raw/master/20210823100616.png)

##### 7.3.3 有密码的集群配置

大体上来讲有无密码的配置实际上差不多,主要区别与需要在配置文件中配置密码:

```
requirepass redis登录密码
masterauth 主从认证密码
```

将密码配置好之后和之前一样启动redis

一般来说集群会搭配主从一起使用,这样可以提高redis性能,如果需要搭配主从一起使用创建集群时请使用如下命令

```
redis-cli --cluster create 1 192.168.68.131:7001 192.168.68.131:7002 192.168.68.131:7003 192.168.68.131:7004 192.168.68.131:7005 192.168.68.131:7006 --cluster-replicas 1 -a admin01*
```

--cluster-replicas 1表示一个主机后有多少个从机,我这里写的1,我们一共有6个redis实例在集群创建完成后会有3个主机3个从机,-a后面填写的是redis的密码

##### 7.3.4 动态添加节点

首先我们准备一个端口为 7007 的主节点并启动，准备方式和前面步骤一样，启动成功后，通过如下命 令添加主节点：

```
redis-cli --cluster add-node 192.168.68.131:7007 192.168.68.131:7001 -a admin01*
```

在节点添加完成之后我们可以使用命令查看节点信息,我们看到新加进来的界面没有分配槽,这也就意味这咋后续的数据读写过程中新加进来的这个7007的主机是不会参与其中的,如图所示:

![image-20210823105156612](https://gitee.com/sixiwanzia/img/raw/master/20210823105156.png)

为了解决这个问题我们需要手动的将槽位进行分配,参照如下步骤进行操作:

1. 输入如下命令来开始槽位的分配,后面的地址为任意一个节点地址

   ```
   redis-cli --cluster reshard 192.168.68.131:7001 -a admin01*
   ```

2. 设置需要划分多少个hash槽出来,也就是我们需要分配到新添加的节点多少槽位,这个参数请根据实际情况来设置,我这里写的是1024

   ![image-20210823105843362](https://gitee.com/sixiwanzia/img/raw/master/20210823105843.png)

3. 设置划分出来的槽位分配到哪一个节点,这里填写7007redis实例的编号

   <img src="https://gitee.com/sixiwanzia/img/raw/master/20210823110038.png" alt="image-20210823110038511" style="zoom:67%;" />

4. 设置从哪些主机中将槽位划分,因为 hash 槽目前已经全部分配完毕,要重新从已经分好的节点中拿出来一部分给 7007,必然要让另外三个节点把现有的槽位划分出来,这里我们可以输入多个节点的编号，每次输完 一个点击回车，输完所有的输入 done 表示输入完成，这样就让这几个节点让出部分 slot,如果要 让所有具有 slot 的节点都参与到此次 slot 重新分配的活动中,那么这里直接输入 all 即可，如下：

   ![image-20210823110348375](C:/Users/曾先生/AppData/Roaming/Typora/typora-user-images/image-20210823110348375.png)  输完之后进入到 slot 重新分配环节，分配完成后，通过 cluster nodes 命 令，我们可以发现 7007 已经具有 slot 了,如下:

    ![image-20210823110938214](https://gitee.com/sixiwanzia/img/raw/master/20210823110938.png)

    刚刚我们是添加主节点，我们也可以添加从节点，比如我要把 7008 作为 7007 的从节点，添加方式 如下： 

   ```
   redis-cli --cluster add-node 192.168.68.131:7008 192.168.68.131:7001 -a admin01* --cluster-slave --cluster-master-id 205ed1250c6a975ef7a530cda4f6f7c2be8eb7b8 #7007的id
   ```

##### 7.3.5 动态删除节点

需要删除集群中的某一个节点需要使用如下命令:

```
redis-cli --cluster  del-node 192.168.68.131:7007 205ed1250c6a975ef7a530cda4f6f7c2be8eb7b8 -a admin01*
```

需要注意的是在删除集群中的节点的时候如果被删除的节点不为空,也就是说被删除的节点上有槽位无论槽位中是否存在数据都会删除失败,因为删除了的话hash就不完整了,所以我们在删除节点之前需要手动将节点上的槽位分配出去否则就会出现如下所示的错误:

![image-20210823111743456](https://gitee.com/sixiwanzia/img/raw/master/20210823111743.png)

分配槽位的方式与7.3.4动态添加节点方式相同不在赘述

#### 7.4 Jedis操作 RedisCluster

Java使用jedis操作RedisCluster还是很方便的,Jedis自带有操作RedisCluster的工具代码如下所示:

```java
public class RedisCluster {
    public static void main(String[] args) {
        Set<HostAndPort> nodes = new HashSet<>();
        nodes.add(new HostAndPort("192.168.68.131",7001));
        nodes.add(new HostAndPort("192.168.68.131",7002));
        nodes.add(new HostAndPort("192.168.68.131",7003));
        nodes.add(new HostAndPort("192.168.68.131",7004));
        nodes.add(new HostAndPort("192.168.68.131",7005));
        nodes.add(new HostAndPort("192.168.68.131",7006));
        nodes.add(new HostAndPort("192.168.68.131",7007));
        JedisPoolConfig config = new JedisPoolConfig();
        //连接池最大空闲数
        config.setMaxIdle(300);
        //最大连接数
        config.setMaxTotal(30);
        //连接的最大等待时间单位ms,-1表示没有限制
        config.setMaxWaitMillis(90000);
        //空闲时检查有效性
        config.setTestOnBorrow(true);
        JedisCluster Cluster = new JedisCluster(nodes, 15000, 15000, 5, "admin01*", config);
        String set = Cluster.set("k1", "v1");
        System.out.println(set);
        String k1 = Cluster.get("k1");
        System.out.println(k1);
    }
}
```

在上述代码中使用Cluster操作和之前的操作没有区别就是一些命令操作,就不再赘述了.

### 8. Redis Steam

##### 8.1 Redis Stream基本介绍

Redis Stream 是 Redis 5.0 版本新增加的数据结构。

Redis Stream 主要用于消息队列（MQ，Message Queue），Redis 本身是有一个 Redis 发布订阅 (pub/sub) 来实现消息队列的功能，但它有个缺点就是消息无法持久化，如果出现网络断开、Redis 宕机等，消息就会被丢弃。

简单来说发布订阅 (pub/sub) 可以分发消息，但无法记录历史消息。

而 Redis Stream 提供了消息的持久化和主备复制功能，可以让任何客户端访问任何时刻的数据，并且能记住每一个客户端的访问位置，还能保证消息不丢失。

Redis Stream 的结构如下所示，它有一个消息链表，将所有加入的消息都串起来，每个消息都有一个唯一的 ID 和对应的内容：

在Stream中,有一个消息链表所有加入链表中的消息,都会被串成一条,每一条消息都会有一个唯一的ID还有对应的消息内容,消息内容实际上也就是键值对.

一个Stream上可以有很多个消费者,每一个消费者都有一个游标在Stream中的消息链表上去移动,消费者每消费一条消息,游标就移动一次.因为每个消费者都有一个属于自己的游标,这些游标互不干扰所以一条消息是可以被重复消费的.

消费者之间互相独立互不影响.

![image-20210823151406426](https://gitee.com/sixiwanzia/img/raw/master/20210823151406.png)

##### 8.2 基本命令

**消息队列相关命令:**

- **XADD** - 添加消息到末尾
- **XTRIM** - 对流进行修剪，限制长度
- **XDEL** - 删除消息
- **XLEN** - 获取流包含的元素数量，即消息长度
- **XRANGE** - 获取消息列表，会自动过滤已经删除的消息 
- **XREVRANGE** -  反向获取消息列表，ID 从大到小
- **XREAD** - 以阻塞或非阻塞方式获取消息列表

   **消费者组相关命令：**

- **XGROUP CREATE** - 创建消费者组
- **XREADGROUP GROUP** - 读取消费者组中的消息
- **XACK** - 将消息标记为"已处理"
- **XGROUP SETID** - 为消费者组设置新的最后递送消息ID
- **XGROUP DELCONSUMER** - 删除消费者
- **XGROUP DESTROY** - 删除消费者组
- **XPENDING** - 显示待处理消息的相关信息
- **XCLAIM** - 转移消息的归属权
- **XINFO** - 查看流和消费者组的相关信息；
- **XINFO GROUPS** - 打印消费者组的信息；
-   **XINFO STREAM** - 打印流信息

**XADD**

使用 XADD 向队列添加消息，如果指定的队列不存在，则创建一个队列，XADD 语法格式：

```
XADD key ID field value [field value ...]
```

- **key** ：队列名称，如果不存在就创建
- **ID** ：消息 id，我们使用 * 表示由 redis 生成，可以自定义，但是要自己保证递增性。
- **field value** ： 记录。

**实例**

```
192.168.68.131:7001> XADD leanRedis * name sixiwanzi age 1
"1628444980646-0"
192.168.68.131:7001> XADD leanRedis * name github age 10+
"1628445020846-0"
192.168.68.131:7001> XLEN leanRedis
(integer) 2
192.168.68.131:7001> Xrange leanRedis
(error) ERR wrong number of arguments for 'xrange' command
192.168.68.131:7001> Xrange leanRedis - +
1) 1) "1628444980646-0"
   2) 1) "name"
      2) "sixiwanzi"
      3) "age"
      4) "1"
2) 1) "1628445020846-0"
   2) 1) "name"
      2) "github"
      3) "age"
      4) "10+"
```

**XTRIM**

使用 XTRIM 对流进行修剪，限制长度， 语法格式：

```
XTRIM key MAXLEN [~] count
```

- **key** ：队列名称
- **MAXLEN** ：长度
- **count** ：数量

**实例**

```
192.168.68.131:6380> XADD myStreams * name sixianzi age 20+ test a
"1628461964650-0"
192.168.68.131:6380> XTRIM myStreams MAXLEN 2
(integer) 0
192.168.68.131:6380> XRANGE myStreams - +
1) 1) "1628461964650-0"
   2) 1) "name"
      2) "sixianzi"
      3) "age"
      4) "20+"
      5) "test"
      6) "a"
192.168.68.131:6380> 
```

**XDEL**

使用 XDEL 删除消息，语法格式：

```
XDEL key ID [ID ...]
```

- **key**：队列名称
- **ID** ：消息 ID

**实例**

```
192.168.68.131:6380> XADD mystrem * test v1 
"1628462187459-0"
192.168.68.131:6380> XADD mystrem * test2 v2 
"1628462192995-0"
192.168.68.131:6380> XADD mystrem * test3 v3 
"1628462197382-0"
192.168.68.131:6380> XRANGE mystrem - +
1) 1) "1628462187459-0"
   2) 1) "test"
      2) "v1"
2) 1) "1628462192995-0"
   2) 1) "test2"
      2) "v2"
3) 1) "1628462197382-0"
   2) 1) "test3"
      2) "v3"
192.168.68.131:6380> XDEL mystrem 1628462192995-0
(integer) 1
192.168.68.131:6380> XRANGE mystrem - +
1) 1) "1628462187459-0"
   2) 1) "test"
      2) "v1"
2) 1) "1628462197382-0"
   2) 1) "test3"
      2) "v3"
```

**XLEN**

使用 XLEN 获取流包含的元素数量，即消息长度，语法格式：

```
XLEN key
```

- **key**：队列名称

**XRANGE**

使用 XRANGE 获取消息列表，会自动过滤已经删除的消息 ，语法格式：

```
XRANGE key start end [COUNT count]
```

- **key** ：队列名
- **start** ：开始值， - 表示最小值
- **end** ：结束值， + 表示最大值
- **count** ：数量

如果不想使用"+"和"-",那就需要填写消息的ID而**==不是==**填写下标.

**XREVRANGE**

使用 XREVRANGE 获取消息列表，会自动过滤已经删除的消息 ，语法格式：

```
XREVRANGE key end start [COUNT count]
```

- **key** ：队列名
- **end** ：结束值， + 表示最大值
- **start** ：开始值， - 表示最小值
- **count** ：数量

用法同XRANGE命令一样

**XREAD**

使用 XREAD 以阻塞或非阻塞方式获取消息列表 ，语法格式：

```
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] id [id ...]
```

- **count** ：数量
- **milliseconds** ：可选，阻塞毫秒数，没有设置就是非阻塞模式
- **key** ：队列名
- **id** ：消息 ID

**实例**

```
# 从 Stream 头部读取两条消息
192.168.68.131:6380> XREAD count 2 streams mystrem myStreams 0-0 0-0
1) 1) "mystrem"
   2) 1) 1) "1628462187459-0"
         2) 1) "test"
            2) "v1"
      2) 1) "1628462197382-0"
         2) 1) "test3"
            2) "v3"
2) 1) "myStreams"
   2) 1) 1) "1628461964650-0"
         2) 1) "name"
            2) "sixianzi"
            3) "age"
            4) "20+"
            5) "test"
            6) "a"
```

**XGROUP CREATE**

使用 XGROUP CREATE 创建消费者组，语法格式：

```
XGROUP [CREATE key groupname id-or-$] [SETID key groupname id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername]
```

- **key** ：队列名称，如果不存在就创建
- **groupname** ：组名。
- **$** ： 表示从尾部开始消费，只接受新消息，当前 Stream 消息会全部忽略。

从头开始消费:

```
XGROUP CREATE mystream consumer-group-name 0-0  
```

从尾部开始消费:

```
XGROUP CREATE mystream consumer-group-name $
```

**XREADGROUP GROUP**

使用 XREADGROUP GROUP 读取消费组中的消息，语法格式：

```
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...]
```

- **group** ：消费组名
- **consumer** ：消费者名。
- **count** ： 读取数量。
- **milliseconds** ： 阻塞毫秒数。
- **key** ： 队列名。
- **ID** ： 消息 ID。

```
XREADGROUP GROUP consumer-group-name consumer-name COUNT 1 STREAMS mystream >
```

">"表示从游标当前位置的下一个位置开始读取

**实例**

```
192.168.68.131:7001> XREADGROUP group cl c count 1 streams leanRedis >
1) 1) "leanRedis"
   2) 1) 1) "1628445810407-0"
         2) 1) "name"
            2) "baidu"
            3) "age"
            4) "10+"
192.168.68.131:7001> XREADGROUP group cl c count 1 streams leanRedis >
1) 1) "leanRedis"
   2) 1) 1) "1628445821232-0"
         2) 1) "name"
            2) "github"
            3) "age"
            4) "20+"
192.168.68.131:7001> XREADGROUP group cl c count 1 streams leanRedis >
1) 1) "leanRedis"
   2) 1) 1) "1628445830022-0"
         2) 1) "name"
            2) "sixianzi"
            3) "age"
            4) "20+"
192.168.68.131:7001> XREADGROUP group cl c count 1 streams leanRedis >
(nil)
```

### 9. Redis 服务器

Redis 服务器命令主要是用于管理 redis 服务。

**实例**

以下实例演示了如何获取 redis 服务器的统计信息：

```
redis 127.0.0.1:6379> INFO

# Server
redis_version:2.8.13
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:c2238b38b1edb0e2
redis_mode:standalone
os:Linux 3.5.0-48-generic x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:4.7.2
process_id:3856
run_id:0e61abd297771de3fe812a3c21027732ac9f41fe
tcp_port:6379
uptime_in_seconds:11554
uptime_in_days:0
hz:10
lru_clock:16651447
config_file:

# Clients
connected_clients:1
client-longest_output_list:0
client-biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:589016
used_memory_human:575.21K
used_memory_rss:2461696
used_memory_peak:667312
used_memory_peak_human:651.67K
used_memory_lua:33792
mem_fragmentation_ratio:4.18
mem_allocator:jemalloc-3.6.0

# Persistence
loading:0
rdb_changes_since_last_save:3
rdb_bgsave_in_progress:0
rdb_last_save_time:1409158561
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:0
rdb_current_bgsave_time_sec:-1
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok

# Stats
total_connections_received:24
total_commands_processed:294
instantaneous_ops_per_sec:0
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:41
keyspace_misses:82
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:264

# Replication
role:master
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# CPU
used_cpu_sys:10.49
used_cpu_user:4.96
used_cpu_sys_children:0.00
used_cpu_user_children:0.01

# Keyspace
db0:keys=94,expires=1,avg_ttl=41638810
db1:keys=1,expires=0,avg_ttl=0
db3:keys=1,expires=0,avg_ttl=0
```

**Redis 服务器命令**

下表列出了 redis 服务器的相关命令:

| 序号 | 命令及描述                                                   |
| ---- | :----------------------------------------------------------- |
| 1    | [BGREWRITEAOF](https://www.runoob.com/redis/server-bgrewriteaof.html)  异步执行一个 AOF（AppendOnly File） 文件重写操作 |
| 2    | [BGSAVE](https://www.runoob.com/redis/server-bgsave.html)  在后台异步保存当前数据库的数据到磁盘 |
| 3    | [CLIENT KILL [ip:port\] [ID client-id]](https://www.runoob.com/redis/server-client-kill.html)   关闭客户端连接 |
| 4    | [CLIENT LIST](https://www.runoob.com/redis/server-client-list.html)  获取连接到服务器的客户端连接列表 |
| 5    | [CLIENT GETNAME](https://www.runoob.com/redis/server-client-getname.html)  获取连接的名称 |
| 6    | [CLIENT PAUSE timeout](https://www.runoob.com/redis/server-client-pause.html)  在指定时间内终止运行来自客户端的命令 |
| 7    | [CLIENT SETNAME connection-name](https://www.runoob.com/redis/server-client-setname.html)  设置当前连接的名称 |
| 8    | [CLUSTER SLOTS](https://www.runoob.com/redis/server-cluster-slots.html)  获取集群节点的映射数组 |
| 9    | [COMMAND](https://www.runoob.com/redis/server-command.html)  获取 Redis 命令详情数组 |
| 10   | [COMMAND COUNT](https://www.runoob.com/redis/server-command-count.html)  获取 Redis 命令总数 |
| 11   | [COMMAND GETKEYS](https://www.runoob.com/redis/server-command-getkeys.html)  获取给定命令的所有键 |
| 12   | [TIME](https://www.runoob.com/redis/server-time.html)  返回当前服务器时间 |
| 13   | [COMMAND INFO command-name [command-name ...]](https://www.runoob.com/redis/server-command-info.html)  获取指定 Redis 命令描述的数组 |
| 14   | [CONFIG GET parameter](https://www.runoob.com/redis/server-config-get.html)  获取指定配置参数的值 |
| 15   | [CONFIG REWRITE](https://www.runoob.com/redis/server-config-rewrite.html)  对启动 Redis 服务器时所指定的 redis.conf 配置文件进行改写 |
| 16   | [CONFIG SET parameter value](https://www.runoob.com/redis/server-config-set.html)  修改 redis 配置参数，无需重启 |
| 17   | [CONFIG RESETSTAT](https://www.runoob.com/redis/server-config-resetstat.html)  重置 INFO 命令中的某些统计数据 |
| 18   | [DBSIZE](https://www.runoob.com/redis/server-dbsize.html)  返回当前数据库的 key 的数量 |
| 19   | [DEBUG OBJECT key](https://www.runoob.com/redis/server-debug-object.html)  获取 key 的调试信息 |
| 20   | [DEBUG SEGFAULT](https://www.runoob.com/redis/server-debug-segfault.html)  让 Redis 服务崩溃 |
| 21   | [FLUSHALL](https://www.runoob.com/redis/server-flushall.html)  删除所有数据库的所有key |
| 22   | [FLUSHDB](https://www.runoob.com/redis/server-flushdb.html)  删除当前数据库的所有key |
| 23   | [INFO [section]](https://www.runoob.com/redis/server-info.html)  获取 Redis 服务器的各种信息和统计数值 |
| 24   | [LASTSAVE](https://www.runoob.com/redis/server-lastsave.html)  返回最近一次 Redis 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示 |
| 25   | [MONITOR](https://www.runoob.com/redis/server-monitor.html)  实时打印出 Redis 服务器接收到的命令，调试用 |
| 26   | [ROLE](https://www.runoob.com/redis/server-role.html)  返回主从实例所属的角色 |
| 27   | [SAVE](https://www.runoob.com/redis/server-save.html)  同步保存数据到硬盘 |
| 28   | [SHUTDOWN [NOSAVE\] [SAVE]](https://www.runoob.com/redis/server-shutdown.html)  异步保存数据到硬盘，并关闭服务器 |
| 29   | [SLAVEOF host port](https://www.runoob.com/redis/server-slaveof.html)  将当前服务器转变为指定服务器的从属服务器(slave server) |
| 30   | [SLOWLOG subcommand [argument]](https://www.runoob.com/redis/server-showlog.html)  管理 redis 的慢日志 |
| 31   | [SYNC](https://www.runoob.com/redis/server-sync.html)   用于复制功能(replication)的内部命令 |

### 10 Redis过期策略

Redis中所有的key都是可以设置过期时间.

在Redis中所有被设置了过期时间的key都会存放在一个独立的字典中,删除这些key有俩个不同的策略:

- 定时遍历字典将已到期的key进行删除.Redis中默认每秒进行10次过期扫描,每次从字典中随机取20个key,删除其中过期的key,如果过期的key的比例操作25%那么再取20个进行扫描,以此类推.
- 在客户端访问的时候查看key是否过期,如果已经过期,则删除

需要注意的是在主从架构中,从机是不会进行key的过期扫描的,主机的key过期之后,会自动同步到从机上.

### 11 Redis 淘汰策略

**LUR**

这个东西其实和Redis没有太大的关系,很多地方都有使用.

在Redis中,因为Redis是基于内存的,当Redis所使用的内存超出物理限制的时候,内存中的数据会和磁盘产生频繁的交换,这种交换会使得Redis的性能极具下降,在实际的开发过程中,我们不允许Redis出现交换(swap)行为.

当Redis实际内存超过物理内存的时候Redis提供了几种策略:

- noeviction:  默认的策略 此时写操作将停止,删除和读取进行读取
- volatile-lru: 最近最少使用算法,淘汰设置了过期时间的key,最近最少使用的key会被淘汰,如果一个key没有设置过期时间,则不会被淘汰
- volatile-ttl: 淘汰设置了过期时间的key,根据设置的key的ttl的值进行淘汰.ttl越小越优先被淘汰如果一个key没有设置过期时间,则不会被淘汰
- volatile-random: 淘汰设置了过期时间的key,随机进行移除.如果一个key没有设置过期时间,则不会被淘汰
- allkeys-lru: 最近最少使用算法,淘汰所有的key,最近最少使用的key会被淘汰.
- allkeys-random: 淘汰所有的key,随机将key进行移除.

### 12. lazy free

Redis是一个单线程程序,如果直接删除一个很大的key可能会造成卡顿,Redis提供了一些异步删除的指令如下:

- UNLINK key (redis4.0后有效)
- FLUSHDB async
- FLUSHALL async

配置文件中可以配置一些异步删除在redis.conf文件搜索如下字符即可:

replica-lazy-flush 

lazy-free-eviction

### 13. 加密通信

Redis本身不支持加密通信,如果需要做加密通信那么需要借助其他的第三方工具,Redis官方推荐的是spiped

使用步骤如下:

1. 安装

   ```
   wget https://github.com/Tarsnap/spiped/archive/refs/tags/1.6.1.tar.gz
   tar -zxvf 1.6.1.tar.gz
   cd spiped-1.6.1
   make
   make install
   ```

2. 生成随机密钥文件

   ```
   dd if=/dev/urandom bs=32 count=1 of=redis.key#这条命令需要在安装了spiped的文件夹下运行
   ```

3. 启动spiped

   ```
   spiped -d -s '[192.168.65.128]:6479' -t '[192.168.65.128]:6379' -k redis.key
   //监控本地的6479端口，在6479端口接收到的消息转发到6379，
   //利用redis.key秘钥文件启动
   ```

   ```  
   //将128上的redis.key复制到129上
   scp ./spiped-1.6.1/redis.key 192.168.65.129:/root
   //在129上执行
   spiped -e -s '[192.168.65.129]:6379' -t '[192.168.65.128]:6479' -k /root/redis.key
   redis-cli -a admin01*  -h 192.168.65.128
   ```

   