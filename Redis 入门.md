# Redis 入门

[toc]

## Redis简介

redis是一个基于C语言进行开发的键值对存储的非关系型数据库.是互联网上使用的最为广泛的一个NoSQl数据库.主要可以实现缓存,分布式锁,限流,地理位置信息存储等功能.

特点如下:

- 支持数据持久化
- 支持多种不同的数据结构类型之间的映射
- 支持主从模式的数据备份
- 自带发布订阅系统
- 定时器、计数器

## Redis安装

一共有四种方式可以获取Redis

1. 直接编译安装(直接在官网下载安装包进行编译安装)
2. 使用Doker进行安装
3. 使用命令进行安装
4. 在线体验,通过在线体验,可以直接使用Redis的功能

### 1.直接编译安装

提前准备GCC环境

```
yum install gcc-c++
```

下载并安装 Redis

```
wget https://download.redis.io/releases/redis-6.0.9.tar.gz --下载
tar -zxvf redis-6.0.9.tar.gz --解压
cd redis-6.0.9/
make --编译
make install --安装
```

ps: 在安装过程中可能会报错

```
cetnos make readis 报错 server.c错误
```

原因是由于自从redis 6.0.0之后,编译redis需要c11支持,C11 特性在 4.9 中被引入。Centos7 默认 gcc 版本为 4.8.5，所以需要升级gcc版本。

```
yum -y install gcc gcc-c++ make tcl
yum -y install centos-release-scl
yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
scl enable devtoolset-9 bash
```

启动redis

```
redis-server redis conf
```

启动成功页面如下

<img src="https://gitee.com/sixiwanzia/img/raw/master/20201204153427.png" alt="image-20201204153418523" style="zoom:70%;" />

### 2. 通过Docker进行安装

提前准备好Docker

启动Docker ,直接运行安装命令即可

```
docker run --name(命名) redis -d(后台运行) -p(端口映射)6379:6379 redis(需要启动的东西)
```

Docker上的redis启动成功之后可以宿主机上进行连接,但是前提是宿主机存在redis cli工具

```
redis-cli (如果有密码请加上-a 密码)
```

如果宿主机上没有安装Redis,也可以进入到Docker容器中进行连接Redis

```
Docker exec -it redis (安装有redis的容器名称) redis-cli -a密码
```

### 3. 直接安装

CentOs

```
yum install redis
```

乌班图

```
apt-get install redis
```

Mac:(建议安装在Docker中)

```
brew install redis
```

### 4. 在线体验

```
https://try.redis.io/
```



### 后台启动Redis

如果使用编译安装的Redis,运行的时候是前台运行的,如果需要更改为后台启动请按如下方式进行操作,另外redis启动是需要进入到redis根目录才可以的

1. 修改配置文件

```
vi redis
```

![image-20201204163649096](https://gitee.com/sixiwanzia/img/raw/master/20201204163649.png)

2. 配置完成后保存退出在通过命令启动redis,此时就是在后台启动了

   ![image-20201204163939234](https://gitee.com/sixiwanzia/img/raw/master/20201204163939.png)
   
   ![image-20210805164143337](https://gitee.com/sixiwanzia/img/raw/master/20210805164150.png)

## Redis中的基本数据类型

在redis中常说的基本数据类型有五种分别是 String、Hash、List、Set、ZSet

### 1. String

String是Redis中最简单的一种数据结构,在Redis中所有的Key都是字符串,但是不同的Key对应的Value则具备不同的数据结构,我们所说的五种不同的数据类型,主要是指Value的数据类型不同.

Redis中的字符串实际上是动态字符串,(内部是可以修改的)类似于Java中的StringBuffer,它是采用分配冗余空间的方式,来减少内存的频繁分配.在redis内部结构中实际分配的内存会大于需要的内存,当字符串小于1M的时候,扩容都是在现有的空间基础上加倍,如果大于1M那么每次扩容则会扩大1M空间,最大是521M

常见的字符串操作命令

- set

  给一个Key赋值

  ![image-20201204165605245](https://gitee.com/sixiwanzia/img/raw/master/20201204165605.png)

- append

  在使用append命令的时候如果Key已经存在那么会直接在对应的Value中追加值反之就创建键值对.

  ![image-20201204165209147](https://gitee.com/sixiwanzia/img/raw/master/20201204165209.png)

- decr

  可以实现对Value的减一操作,前提是Value为数字.不是数字会报错,如果Value不存在会给一个默认值0,然后在默认值的基础上减一

  ![image-20201204165428407](https://gitee.com/sixiwanzia/img/raw/master/20201204165428.png)

- decrby

  这个和decr类似,但是可以设置步长

  ![image-20201204165723820](https://gitee.com/sixiwanzia/img/raw/master/20201204165723.png)

- get 

  用来获取一个key的value

- getrange 

  用来返回一个key中的Value的子串,类似JAVA中的substring方法,但是不会修改本身的值.这个命令的第二个和第三个参数就是截取的起始和终止位置,其中-1表示最后一个字符串,-2表示倒数第2个以此类推

  ![image-20201204170122132](https://gitee.com/sixiwanzia/img/raw/master/20201204170122.png)

- getset

  这个命令表示获取并更新一个key的Value

  ![image-20201204170404182](https://gitee.com/sixiwanzia/img/raw/master/20201204170404.png)

- incr 

  这个命令表示给某一个key的value自增

- incrby

  这个命令表示给某一个key的value自增可以设置步长

- incrbyfloat

  和incrby类似,但是可以设置浮点数

![image-20201204170727133](https://gitee.com/sixiwanzia/img/raw/master/20201204170727.png)

- mget和mset

  批量获取和批量存储

  ![image-20201205093309341](https://gitee.com/sixiwanzia/img/raw/master/20201205093316.png)

- ttl

  查看key的有效期(-1表示永不过期,-2表示已经过期)

  ![image-20201205093534746](https://gitee.com/sixiwanzia/img/raw/master/20201205093534.png)

- setex

  在给key设置value的同时设置过期时间,单位是s

  ![image-20201205093754323](https://gitee.com/sixiwanzia/img/raw/master/20201205093754.png)

- psetsx

  和setex类似,但是这个命令的时间单位是ms

  ![image-20201205093931198](https://gitee.com/sixiwanzia/img/raw/master/20201205093931.png)

- setnx

  默认情况下set命令会覆盖已存在的key,但是setnx不会

  ![image-20201205094210033](https://gitee.com/sixiwanzia/img/raw/master/20201205094210.png)

- msetnx 

  批量设置,但是在批量设置的时候只要有一个key存在,那么整个msetnx操作都会失败

  ![image-20201205094511296](https://gitee.com/sixiwanzia/img/raw/master/20201205094511.png)

- setrange

  覆盖一个已经存在key的value,这个命令可以设置偏移量,就是从第几个字符开始覆盖,如果设置的偏移量大于原本字符串的长度,不足的地方会用0补齐

  ![image-20201205095105146](https://gitee.com/sixiwanzia/img/raw/master/20201205095105.png)

- strlen

  查看字符串长度

  ![image-20201205095148059](https://gitee.com/sixiwanzia/img/raw/master/20201205095148.png)
 #### 1.1 StringBit相关命令

在redis中string的存储都是以2进制的方式进行存储的.例如 ``` set   k1 a``` a对应的ASCII码是97,97的二进制表示为01100001,Bit相关的命令是直接操作这个二进制码

- getbit

  返回key对应的value在offset处对应的bit值.

  ![image-20201205100333673](https://gitee.com/sixiwanzia/img/raw/master/20201205100333.png)

- setbit

  修改key对应的value在offet处的bit值

  ![image-20201205100538069](https://gitee.com/sixiwanzia/img/raw/master/20201205100538.png)

- bitcount 

  统计二进制数据中1的个数,可以设置始末位置,但是这里的石墨位置不是值二进制中的始末位置,指定的是key对应的value的始末位置

  ![image-20201205100848419](https://gitee.com/sixiwanzia/img/raw/master/20201205100848.png)



### 2. List

List实际上指的就是key对应的value的数据类型是列表

- lpush

  表示将value的值按照从左到右的方式依次插入表头位置.里面的数据结构就像栈的结构一样

- lrange

  返回列表中指定区间内的元素

- rpush

  和lpush功能基本类似,不同的是,将数据从右往左一次插入表头位置

  ![image-20201205101843506](https://gitee.com/sixiwanzia/img/raw/master/20201205101843.png)

- rpop

  移除并返回列表的尾元素

- lpop

  移除并返回列表的头元素

  ![image-20201205102142156](https://gitee.com/sixiwanzia/img/raw/master/20201205102142.png)
  
- lindex

  返回列表下标为index的元素,不会移除数据

  ![image-20201207095228375](https://gitee.com/sixiwanzia/img/raw/master/20201207095235.png)

- ltrim

  可以对一个列表进行修剪,只保留指定区间的数据

  ![image-20201207095421846](https://gitee.com/sixiwanzia/img/raw/master/20201207095421.png)

- blpop

  阻塞式的弹出,相当于lpop的阻塞版.当列表里面没有元素时会阻塞,知道列表中有元素可以弹出.或者到达了指定的时间,这个时间单位是s

### 3.Set

**==set和list的主要区别是集合内的元素不可重复==**

- sadd

  添加元素到一个key中,可以添加多个不重复的value,但是如果指定的是元素是重复的,就只会添加一个

- smembers

  获取一个key下的所有元素

  ![image-20201207100630022](https://gitee.com/sixiwanzia/img/raw/master/20201207100630.png)

- srem

  移除指定的元素

  ![image-20201207100902883](https://gitee.com/sixiwanzia/img/raw/master/20201207100902.png)

- sismemeber

  返回某一个成员是否在集合中,返回1表示存在

  ![image-20201207101021370](https://gitee.com/sixiwanzia/img/raw/master/20201207101021.png)

- scard

  返回集合的数量

  ![image-20201207101325206](https://gitee.com/sixiwanzia/img/raw/master/20201207101325.png)

-  SRANDMEMBER

  随机返回一个元素,元素个数可以指定,如果指定的长度大于集合长度,则会返回集合中全部元素

  ![image-20201207101315206](https://gitee.com/sixiwanzia/img/raw/master/20201207101315.png)

- spop

  和SRANDMEMBER类似,spop的作用是随机返回并且移除一个元素

  ![image-20201207101613455](https://gitee.com/sixiwanzia/img/raw/master/20201207101613.png)

- smove

  将指定集合中的某一个元素移到另一个指定的集合中去

  ![image-20201207144239521](https://gitee.com/sixiwanzia/img/raw/master/20201207144239.png)

- sdiff

  返回一个集合和给定集合的差集元素(以第一个集合为准,除去第二个集合中的所有元素,返回剩下的元素)

  ![image-20201207145850355](https://gitee.com/sixiwanzia/img/raw/master/20201207145850.png)

- sinter

  返回两个指定集合的交集

  ![image-20201207150333480](https://gitee.com/sixiwanzia/img/raw/master/20201207150333.png)

- sdiffstore

  和sdiff类似,不同的是会将计算出来的结果保存在一个新的集合中
  
  ![image-20201218102503287](https://gitee.com/sixiwanzia/img/raw/master/20201218102510.png)
  
- sinterstore

  和 sinter类似,不同的是会将计算出来的结果保存在一个新的集合中

- sunion

  求并集

- sunionstore

  求并集并且将结果保存在新的集合中

  ![image-20201218102816185](https://gitee.com/sixiwanzia/img/raw/master/20201218102816.png)

### 4.Hash

hash这种数据结构其实有点类似微缩版的redis,这种数据结构里面的value又像key value这种结构.在hash结构中key是一个字符串,而value是一个键值对

- hset

  添加值

  ![image-20201218103305223](https://gitee.com/sixiwanzia/img/raw/master/20201218103305.png)

- hget

  获取值

  ![image-20201218103349646](https://gitee.com/sixiwanzia/img/raw/master/20201218103349.png)

- hmget

  批量获取值

- hmset

  批量添加值

  ![image-20201218103559671](https://gitee.com/sixiwanzia/img/raw/master/20201218103559.png)

- hdel

  删除一个指定的field,删除不是指定key,删除的是指定key中对应的value中指定的key所对应的值

  ![image-20201218103839934](https://gitee.com/sixiwanzia/img/raw/master/20201218103839.png)

- hsetnx

  默认情况下如果key和field相同则会覆盖掉原有的value,而hsetnx不会

  ![image-20201218104239107](https://gitee.com/sixiwanzia/img/raw/master/20201218104239.png)

- hvals

  获取所有的value

- hkeys

  获取所有的key
  
- hgetall

  获取所有的key 和value

![image-20201218104620760](https://gitee.com/sixiwanzia/img/raw/master/20201218104620.png)

 - hexists

   返回field是否存在

   ![image-20201218105118374](https://gitee.com/sixiwanzia/img/raw/master/20201218105118.png)

 - hincrby

   给指定的value自增需要指定步长

 - hincrbyfloat

   可以自增一个浮点数

   ![image-20201218105237482](https://gitee.com/sixiwanzia/img/raw/master/20201218105237.png)

- hstrlen   

  返回指定key中 field的长度

![image-20201218105533184](https://gitee.com/sixiwanzia/img/raw/master/20201218105533.png)


### 5. ZSet

这个类似于set,但是这个是一个有序的集合,这种数据结构中有一个叫分数的概念,也就是根据这个来给集合中的元素进行排序

- zadd

将指定的元素添加到有序集合中

- zscor

可以返回有序集合中member的 scor值(也就是分数)

- zrange

返回集合中的一组元素,带上withscor参数可以和分数一起返回

- zrevrange

和zrange一样,但是返回的顺序是倒序

![image-20201218110357213](https://gitee.com/sixiwanzia/img/raw/master/20201218110357.png)

- zcard

可以返回元素个数

![image-20201218110456792](https://gitee.com/sixiwanzia/img/raw/master/20201218110456.png)

- zcount

返回score(分数)在某一个区间的元素,分数区间前不加"("是闭区间,加上是开区间

![image-20201218110708507](https://gitee.com/sixiwanzia/img/raw/master/20201218110708.png)

- zrangebyscore

按照scroe的范围返回元素,开闭区间原则同zcount命令,也可以加上withscroe连同scroe一起返回

![image-20201218110935841](https://gitee.com/sixiwanzia/img/raw/master/20201218110935.png)

- zrank

可以返回元素的从小到大的排名

- zrevrank

可以返回元素的从大到小的排名

![image-20201218111205192](https://gitee.com/sixiwanzia/img/raw/master/20201218111205.png)

- zincrby 

给score自增需要指定增加量

![image-20201218111531466](https://gitee.com/sixiwanzia/img/raw/master/20201218111531.png)

- zinterstore

可以计算两个集合的交集,并且将结果放在一个新的集合中(需要指定有几个集合要求交集),在新的集合中会将对应的score求和

![image-20201218111945392](https://gitee.com/sixiwanzia/img/raw/master/20201218111945.png)

- zrem

将一个元素弹出

![image-20201218112036728](https://gitee.com/sixiwanzia/img/raw/master/20201218112036.png)

- zlexcount

计算成员数量,这里是使用的元素来进行统计这里参数使用 "-" 和 "+" 表示从第一个到最后一个元素之间一共有多少个元素,如果想指定元素需要使用"[元素"表示

![image-20201218113317052](https://gitee.com/sixiwanzia/img/raw/master/20201218113317.png)

- zrangebylex

这个会返回指定区间内的成员

![image-20201218113429176](https://gitee.com/sixiwanzia/img/raw/master/20201218113429.png)



## 针对key相关的操作命令

针对于key进行操作的命令是可以对任意redis中任意一种数据类型进行操作,不管是哪一种数据类型都可以使用,常见的命令有一下几个

- del

删除一个key  value键值对

![image-20201218113800753](https://gitee.com/sixiwanzia/img/raw/master/20201218113800.png)

- dump

序列化给定的key,并返回序列化之后的值

![image-20201218113946802](https://gitee.com/sixiwanzia/img/raw/master/20201218113946.png)

- exists

判断一个key是否存在

![image-20201218114032276](https://gitee.com/sixiwanzia/img/raw/master/20201218114032.png)

- ttl

查看一个key的有效期(-1,表示永不过期,-2表示已经过期或不存在)

![image-20201218114223626](C:/Users/曾先生/AppData/Roaming/Typora/typora-user-images/image-20201218114223626.png)

- expire

给一个key设置有效期,单位是秒,**如果key在过期之前被重新set,过期时间会失效**

![image-20201218115006088](https://gitee.com/sixiwanzia/img/raw/master/20201218115006.png)

- persist

移除一个key的过期时间

- keys *

查看所有的key(*是通配符,可以使用别的符号来匹配,而且也是支持正则表达式的)

![image-20201218114557988](https://gitee.com/sixiwanzia/img/raw/master/20201218114558.png)

- pttl

和ttl一样,但是这里返回的是毫秒

## 特殊说明

1. 五种数据类型除字符串外,在第一次使用的时候,如果容器不存在,会自动进行创建一个
2. 五种数据类型除字符串外,如果里面没有元素,那么会立即删除容器,进行释放内存

## Redis 的JAVA客户端

### 1. 开启Redis远程连接

Redis 默认是不支持远程连接的,需要手动开启,开启方法,直接修改Redis的配置文件即可,一共需要修改两个地方:

- 注释bind :127.0.0.1

![image-20201218150947039](https://gitee.com/sixiwanzia/img/raw/master/20201218150947.png)

- 开启密码校验

![image-20201218151737033](https://gitee.com/sixiwanzia/img/raw/master/20201218151737.png)

改完之后保存退出,启动Redis.

### 2. Jedis

#### 2.1通过Jedis来操作

通过Jedis来操作Redis比较简单,Jedis的github地址是[https://github.com/xetorthio/jedis/](https://github.com/xetorthio/jedis/)

2.1.1 首先创建一个普通的Maven项目
        2.1.2 项目创建成功之后添加Jedis依赖

``` xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.3.0</version>
    <type>jar</type>
    <scope>compile</scope>
</dependency>
```
2.1.3 创建一个测试用的方法

```java
public static void main(String[] args) {
        //1.构造一个Jedis对象,如果使用默认端口6379则可以不填写
        Jedis jedis = new Jedis("192.168.68.131",6379);
        //2.密码认证
        jedis.auth("redis连接远程密码");
        //3.测试连接是否成功
        String ping = jedis.ping();
        //4.返回pong表示连接成功
        System.out.println(ping);
    }
```

2.1.4 对于Jedis来说一旦连接上Redis服务端,即可使用实例化后的对象,Jedis里的方法API和Redis命令基本一致,后面的操作相对简单.这里不在过多赘述.

#### 2.2 连接池优化

在实际应用中,Jedis示例一般是通过连接池来进行获取,原因是Jedis对象并不是线程安全的,在多线陈环境中使用会变得危险.所以当我们使用Jedis对象时从连接池获取Jedis,使用完成后再归还给连接池

```JAVA
  public static void main(String[] args) {
        //1. 构造一个Jedis连接池
        JedisPool jedisPool = new JedisPool("192.168.68.131",6379);
        //2. 从连接池中获取一个Jedis连接
        Jedis jedis = jedisPool.getResource();
        //3.jedis操作
        jedis.auth("admin01*");
        String ping = jedis.ping();
        System.out.println(ping);
        //归还连接
        jedis.close();
    }
```

这样写的话,有可能会在第3步的时候,如果抛出异常,会导致当前的Jedis连接无法归还,导致这个Jedis连接的浪费

```java
 public static void main(String[] args) {
        Jedis jedis = null;
        
            //1. 构造一个Jedis连接池
            JedisPool jedisPool = new JedisPool("192.168.68.131", 6379);
            //2. 从连接池中获取一个Jedis连接
            jedis = jedisPool.getResource();
     try {
            //3.jedis操作
            jedis.auth("admin01*");
            String ping = jedis.ping();
            System.out.println(ping);
        } catch (Exception e) {
            e.printStackTrace(); 
        } finally {
            //归还连接
            if (jedis != null)
                jedis.close();
        }
    }
```

通过finally我们可以确保Jedis是一定为会关闭.

利用JDK1.7中的try-with-resource可以进行改造

```java
public static void main(String[] args) {
        JedisPool jedisPool = new JedisPool("192.168.68.131", 6379);
        try(Jedis jedis= jedisPool.getResource()) {
             jedis.auth("admin01*");
            //3.jedis操作
            String ping = jedis.ping();
            System.out.println(ping);
        } 
    }
```

这段代码和上面的作用是一样的

但是上面这段代码无法实现强约束,可以进一步的改进,可以通过使用接口来进行强约束比如

``` java
public interface CallWithJedis {
    void callJedis(Jedis jedis);
}


public class Redis {
    private JedisPool pool;
    public Redis(){
        GenericObjectPoolConfig config = new GenericObjectPoolConfig();
        //连接池最大空闲数
        config.setMaxIdle(300);
        //最大连接数
        config.setMaxTotal(30);
        //连接的最大等待时间单位ms,-1表示没有限制
        config.setMaxWaitMillis(90000);
        //空闲时检查有效性
        config.setTestOnBorrow(true);
        /**
         * 1.Redis 地址
         * 2. 端口
         * 3. 连接超时时间
         * 4. 密码
         */
        pool = new JedisPool(config,"192.168.68.131 ",6379,30000,"admin01*");
    }
    public void excute(CallWithJedis callWithJedis){
        try (Jedis jedis = pool.getResource()){
            callWithJedis.callJedis(jedis);
        }
    }
}
 new Redis().excute(jedis -> {
           
            System.out.println(jedis.ping());
        });
```

但是如果是在分布式环境中进行操作,需要注意的是要多一个连接失败进行重试的一个机制

### 3. Lettuce

github地址[https://github.com/lettuce-io/lettuce-core](https://github.com/lettuce-io/lettuce-core)

Lettuce和Jedis的不同

1. Jedis的实现是直接连接Redis的,那么就会造成在多个线程之间共享一个Jedis实例,这是非线程安全的,如果需要在多线程环境中使用Jedis.那么就需要使用到连接池,这样每个线程都有自己的Jedis实例,但是这样不可避免的会造成物理资源的浪费.
2. lettuce是基于Netty NIO框架来构建的,其方法调用是异步的。所以克服了Jedis中线程不安全的问题,Lettuce支持同步,异步,响应式的调用,多个线程是可以共享同一个连接实例

#### 使用Lettuce

首先创建一个普通的Maven项目,添加Lettuce依赖.

```xml
<dependency>
        <groupId>io.lettuce</groupId>
        <artifactId>lettuce-core</artifactId>
        <version>5.2.2.RELEASE</version>
    </dependency>
```

接下来就编写一个简单的测试案例

```java
public static void main(String[] args) {
        RedisClient redisClient = RedisClient.create("redis://admin01*@192.168.68.131");
        StatefulRedisConnection<String, String> connection = redisClient.connect();
        RedisAsyncCommands<String, String> async = connection.async();
        async.set("name","test");
        RedisFuture<String> name = async.get("name");
        try {
            System.out.println(name.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
```

需要注意的是这里的密码是直接写在连接里面的```redis://这里写你的redis密码@这里写Reids地址```