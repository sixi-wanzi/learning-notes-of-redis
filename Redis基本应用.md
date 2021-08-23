## Redis基本应用以及常见场景

[toc]

### 1. Redis做分布式锁

#### 1.1 分布式锁简介,以及简单实现

分布式锁也算是Redis使用之中比较常见的一个使用场景了.

分布式锁的应用场景一般如下:

> 例如一个简单的用户操作,一个线程去修改用户的状态,首先从数据库中读取用户状态,然后再内存中进行修改,修改完成后再存入数据库.在单线程中是没有问题的,但是在多线程的环境中,由于读取,修改,存储是三个操作,而且也不是原子操作.所以在多线程中这样会出问题.(多个线程同时操作修改同一条数据的情景,如果不加锁肯定不行)

对于这种问题,我们可以使用分布式锁来进行限制程序的并发执行

**分布式锁的实现思路十分简单,就是当一个线程进行操作的时候先占位,这个时候当别的线程试图操作的时候,发现已经有线程在位了就会放弃或者稍后再试**

在Redis中,占位一般使用setnx命令进行占位,先进行操作的线程先占位,线程的操作执行完成之后在调用del指令进行释放位置.

```java
 public static void main(String[] args) {
        Redis redis = new Redis();
     //首先连接Redis,然后进行操作,这里是用的连接池,具体方法可以参照Redis入门文档
        redis.excute(jedis -> {
            Long setnx = jedis.setnx("k1", "v1");
            if (setnx==1){
                //没人占位
               jedis.set("name", "test1");
                String name = jedis.get("name");
                System.out.println(name);
                jedis.del("k1");//释放位置
            }else{
                //有人占位,停止操作,或者暂缓操作
            }
        });
    }
```

这样就完成了一个简单的分布式锁,但是上面的代码还是存在问题的,如果在业务执行的过程中产生了异常,或者程序异常退出,这样会导致del指令无法呗调用,这样k1无法释放,后来的请求会全部阻塞,锁也永远得不到释放.要解决这个问题我们可以给锁添加一个过期时间,确保这个锁在一定的时间之后可以被释放.改进后的代码如下:

```java
 public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(jedis -> {
            Long setnx = jedis.setnx("k1", "v1");
            if (setnx == 1) {
                //没人占位
                //给锁添加过期时间,防止在运行过程中抛出异常导致锁无法得到释放
                jedis.expire("k1", 5);
                jedis.set("name", "test1");
                String name = jedis.get("name");
                System.out.println(name);
                jedis.del("k1");//释放位置
            } else {
                //有人占位,停止操作,或者暂缓操作
            }
        });
    }
```

但是这样改进之后还是存在问题的,就是在获取锁和设置过期时间之间如果服务器突然离线了,或者突然挂掉了,这个时候锁被占用无法及时释放,同样也会造成死锁.因为获取锁和设置过期时间是两个操作并不具备原子性.其实Redi的作者同样也发现了这个问题所以在Redis2.8开始set的指令就添加了一个拓展参数,这样就可以让setnx和expire指令通过一个命令一起执行.

对上述代码再次改进

``` java
public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(jedis -> {
            String set = jedis.set("k1", "v1", new SetParams().nx().ex(5));
            if ( set!=null &&"OK".equals(set)) {
                //没人占位
                //给锁添加过期时间,防止在运行过程中抛出异常导致锁无法得到释放
                jedis.expire("k1", 5);
                jedis.set("name", "test1");
                String name = jedis.get("name");
                System.out.println(name);
                jedis.del("k1");//释放位置
            } else {
                //有人占位,停止操作,或者暂缓操作
            }
        });
    }
```

#### 1.2解决超时问题

为了防止业务代码在执行的时候抛出异常导致死锁,我们给每一个锁都加上了超时时间,超时之后锁会被自动释放,但是这也带来了一个新的问题:如果要执行的任务非常耗时,可能就会出现紊乱.例如:

> 第一个线程首先获取到锁设置的超时时间为5s,开始执行业务代码,但是业务代码比较耗时,执行了8s超过了事先设置的超时时间,这样会在第一个线程的任务还没有执行成功,锁就被释放了.这个时候第二个线程获取到锁,开始执行,在第二个线程执行3s的时候第一个线程就会释放锁,而且释放的锁是第2个线程的锁因为第一个线程的锁早就因为超时而导致释放了.释放之后第三个线程就会获取到锁并开始执行

对于这个问题,可以从两个角度入手:

- 尽量避免获取锁之后执行耗时操作.
- 可以在锁上进行操作,将锁的Value设置为一个随机字符串,那么每次释放锁的时候都去比较随机字符串是否一致,如果一致再去释放,否则就不释放

对于第二种方案由于在释放锁的时候需要去查看锁的value,然后比较value的值是否正确,最后释放资源.很明显这三个步骤不具备原子性,为了解决这个问题,我们需要引入Lua脚本.

Lua脚本的优势:

- 使用方便,Redis中内置了对Lua脚本的支持
- Lua脚本可以在Redis服务端原子的执行命令
- 由于网络在很大程度上影响到Redis性能,而使用Lua脚本可以让多个命令一次执行,可以有效的降低网络给Redis带来的性能问题.

在Redis中使用Lua脚本大致上有两种思路:

1. 在Redis服务端写好脚本,然后再Java客户端直接进行调用(推荐使用这种方法)
2. 可以直接再Java端写Lua脚本,写好Lua脚本后,发送到Redis上去执
   行.

解决超时问题的具体方案如下:

1. 在Redis服务端创建Lua脚本内容如下:

``` lua
if redis.call("get",KEYS[1])==ARGV[1] then
   return redis.call("del",KEYS[1])
else
   return 0
end
```

2. 给Lua脚本求一个SHA1和(也就是算出一个文件的唯一标志符)命令如下:

```centos
 cat ../../home/admin/lua/relesewherevalueequal.lua | redis-cli -a 这里写你的Redis密码 script load --pipe
```

script load这个命令会在Redis服务器中缓存Lua脚本,并返回脚本的SHA1校验和然后再JAVA调用的时候传入SHA1校验和作为参数,这样Redis服务端就知道需要执行的是哪一个脚本.

![image-20210107095044178](https://gitee.com/sixiwanzia/img/raw/master/20210107095051.png)

3. 在JAVA中调用Lua脚本(也可以使用jedis.eval("脚本内容","key数组","其他参数[]")方法,将脚本内容拼成字符串来执行)

``` java
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(jedis -> {
            //1.先获取随机字符串
            String value = UUID.randomUUID().toString();
            //获取锁以及锁的过期时间
            String k1 = jedis.set("k1", value, new SetParams().nx().ex(5));
            //判断是否拿到锁
            if (k1 != null && "OK".equals(k1)) {
                //4.具体的业务逻辑
                jedis.set("test", "测试");
                String test = jedis.get("test");
                System.out.println(test);
                //5.释放锁
                jedis.evalsha("b8059ba43af6ffe8bed3db65bac35d452f8115d8", Arrays.asList("k1"),
                        Arrays.asList(value));
            } else {
                System.out.println("没有获取到锁");
            }

        });
    }
```



### 2. Redis做消息队列

我们平时说到的消息队列,一般都是指的RabbitMQ、RocketMQ、ActiveMQ以及大数据中的Kafka,这些都是java中比较常见而且非常专业的消息中间件.当然作为专业的消息中间件,他里边提供了很多功能.但是当我们需要使用消息中间件的时候,并不是每次都需要专业的消息中间件.因为不是每次使用的时候对应的业务场景很复杂.比如我们只有一个消息队列,一个消费者的时候这种情况我们就没有必要使用那些专业的消息中间件.这种情况下我们就可以使用Redis来做消息队列,**当然Redis的消息队列不是很专业有很多高级特性都没有,仅仅适用于简单场景,如果对于消息可靠性有极高的要求,那么Redis的消息队列将不在适用**

#### 2.1 消息队列

Redis做消息队列一般的做法是使用Redis中的List数据结构进行实现的,使用lpush/rpush来进行入队,然后使用lpop/rpop来实现出队.

![image-20210121152158342](https://gitee.com/sixiwanzia/img/raw/master/20210121152205.png)

在客户端(例如JAVA端)常见的做法是维护一个死循环来不停的从队列中读取消息,并处理,如果有消息,则会直接获取到,如果没有消息就会陷入死循环,直到下一次有消息进入,但是这种情况回造成大量的资源浪费,这时我们可以使用blpop/brpop

#### 2.2延迟消息队列

延迟队列可以通过使用zset来实现,因为zset中有score,将时间作为score把value存到redis中,然后通过轮询去读取消息出来.

1. 首先 如果消息是string,即可直接发送,如果消息是对象则需要将对象进行序列化.可以使用JSON来实现序列化和反序列化.(需要在项目中添加JSON依赖)

```xml
 <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.10.3</version>
 </dependency>
```

2. 构造一个消息对象

````java
public class JavaMessage {
    private String id;
    private Object data;

    @Override
    public String toString() {
        return "JavaMessage{" +
                "id='" + id + '\'' +
                ", data=" + data +
                '}';
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }
}

````

3. 封装一个消息队列

```java
public class DelayMsgQueue {
    private Jedis jedis;
    private String queue;

    public DelayMsgQueue(Jedis jedis, String queue) {
        this.jedis = jedis;
        this.queue = queue;
    }

    /**
     * 消息入队方法
     *
     * @param data 要发送的消息内容
     */
    public void queue(Object data) {
        //构造一个消息对象
        JavaMessage javaMessage = new JavaMessage();
        javaMessage.setId(UUID.randomUUID().toString());
        javaMessage.setData(data);
        //序列化
        try {
            String s = new ObjectMapper().writeValueAsString(javaMessage);
            System.out.println("msg public" + new Date());
            //发送消息 score 延迟5s
            jedis.zadd(queue, System.currentTimeMillis() + 5000, s);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
    }

    /**
     * 消息消费
     */
    public void loop() {
        while (!Thread.interrupted()) {
            //读取score在0到当前时间之间的数据
            Set<String> strings = jedis.zrangeByScore(queue, 0, System.currentTimeMillis(), 0, 1);
            if (strings.isEmpty()) {
                //如果没有读取到消息,暂缓读取
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                   break;
                }
                continue;
            }
            //如果读取到了数据那么直接获取消息
            String next = strings.iterator().next();
            if (jedis.zrem(queue,next)>0){
                //获取到消息,进行相应的处理
                try {
                    JavaMessage javaMessage = new ObjectMapper().readValue(next, JavaMessage.class);
                    System.out.println("receiver msg"+javaMessage);
                } catch (JsonProcessingException e) {
                    e.printStackTrace();
                }

            }
        }
    }
}
```

4. 测试

```java
public class DelayMsgTest {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(jedis -> {
            //构造消息队列
            DelayMsgQueue queue = new DelayMsgQueue(jedis, "MyDelayQueue");
            //构造消息生产者
            Thread producer = new Thread(){
                @Override
                public void run() {

                    for (int i = 0; i < 5; i++) {
                        queue.queue("Test>>>>>>"+i);
                    }
                }
            };
            //构造消息消费者
            Thread consumer = new Thread(){
                @Override
                public void run() {
                    queue.loop();
                }
            };
            producer.start();
            consumer.start();
            try {
                Thread.sleep(7000);
                consumer.interrupt();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
}
```

![image-20210204102515583](https://gitee.com/sixiwanzia/img/raw/master/20210204102522.png)

### 3. Redis操作位图

常见的应用场景:

网站的签到记录,如果使用String类型来进行存储则会需要很多个Key/Value对操作起来较为麻烦.所以一般使用位图来进行操作.位图的操作比较简单,每天的签到记录占用一个位,一年也只有365个位,可以有效减少存储时所需要的空间. 

对于位图的操作可以直接操作对应的字符串(get/set),也可以操作位(getbit/setbit)

#### 3.1 基本操作

##### 3.1.1 零存整取z

在操作的时候直接操作位,获取数据的时候直接获取字符串.

例如:

>存储一个Java字符串
>
>| 字符 | ASCII | 二进制    |
>| :--: | :---: | --------- |
>|  J   |  74   | 01001010  |
>|  a   |  97   | 0110 0001 |
>|  v   |  118  | 0111 0110 |
>
>存储数据

![image-20210205094442320](https://gitee.com/sixiwanzia/img/raw/master/20210205094449.png)



##### 3.1.2 整存零取

在存储和操作的时候直接针对字符串,但是获取数据的时候是获取的数据对应的位

![image-20210205154531990](https://gitee.com/sixiwanzia/img/raw/master/20210205154532.png)

##### 3.1.3 统计

例如:

某个网站中的某一用户的签到记录是"0110 0001"(1表示已经签到,0表示未签到),如果需要统计该用户一共的签到天数,只需要统计这个字符串中1出现的次数即可.

可以使用的命令bitcount.这个命令可以指定统计的开始和结束的位置,但是需要注意的是这个起始位置是指的这个位数据对应的字符串的起始位置.比如这样一个2进制数据(010010100110 00010111 01100110 0001)在使用bitcount指定起始和结束位置的时候这个起始和结束位置实际上对应的位置是JAVA这个字符串的位置.

还有一个命令是bitpops,这个命令是用来查找指定范围内第一个出现的0或者1(这个命令指定的范围规则依旧和bitcount命令规则一样)

##### 3.1.4 Bit批处理

在Redis 3.2之后新增bitfiled可以对bit进行批量的操做

**GET:**

BITFIELD name get u4 0表示获取name中的位从0开始获取4个位并返回一个无符号数

- u表示无符号数字
- i表示有符号数字,获取的位数据中第一个位表示符号1表示负号

BITFIELD 可以一次进行多个操作

![image-20210206104448659](https://gitee.com/sixiwanzia/img/raw/master/20210206104455.png)

**SET:**

 BITFIELD name set u8(u表示无符号,8代表长度) 8(偏移量可以理解为起始位置) 98(Value) 

![image-20210206110847047](https://gitee.com/sixiwanzia/img/raw/master/20210206110847.png)

**INCRBY:**

这个命令是用于对指定范围内的数据进行自增操作,有可能会导致溢出(包含向上和向下溢出)Redis针对针对溢出的默认处理方案是折返,当8位无符号数255加1溢出会变为0,8位有符号数127溢出会变成-128.

可以修改默认的溢出策略,可以改为fail,表示执行失败.也可以改为sat,表示一直停留在最大/最小值

### 4.HyperLogLog

一般来说我们评估一个网站的访问量有几个主要的参数:

- pv  Page View  网站的浏览量 
- uv User View 访问的用户

pv 或者uv可以自己制作,也可以借助第三方工具来进行制作.比如cnzz,有盟等.如果自己实现的话PV可以通过redis计数器来进行实现.uv由于涉及到去重的问题,首先需要在前端给每个用生成一个用户ID,无论是登录用户还是未登录用户都需要有唯一标志符,这个ID会和请求一起被后端接收.在后端会使用Set集合的sadd来存储这个ID,最后通过scard统计集合大小.这样就可以得出UV数据.但是当千万级别数据量的UV的时候,需要用到的存储空间非常大,而且类似UV统计这种一般不需要特别精确,所以我们可以使用HyperLogLog

Redis中提供的HyperLogLog就是专门用来解决这个问题的.HyperLogLog提供了一个相对不那么准确的去重方案,虽然会有误差但是在可以接收的范围.官方给的误差范围在0.81%,这个精确度对于UV的统计是够用的.

HyperLogLog主要提供了两个命令: pfadd(添加) pfcount(统计).

![image-20210206114131213](https://gitee.com/sixiwanzia/img/raw/master/20210206114131.png)

pfadd类似sadd用来添加数据,在添加过程中,重复数据会自动去重

pfcount用来统计数据.

我们可以使用JAVA代码来多添加几个元素来验证是否会有误差

```java
public class HyperLogLog {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(jedis -> {
            for (int i = 0; i < 10000; i++) {
                //这里是为了制作重复数据所以会有u+0和u+1
                jedis.pfadd("uv","u"+i,"u"+1);
            }
            long uv = jedis.pfcount("uv");
            //理论值是10001
            System.out.println(uv);
        });
    }
}
```

![image-20210206114905057](https://gitee.com/sixiwanzia/img/raw/master/20210206114905.png)

理论值是10001 但是实际上只有9997明显存在误差.

除了上述命令外还有一个是pfmerge,合并多个统计结果,在合并的过程中会自动去重多个集合中重复的元素

![image-20210206115246465](https://gitee.com/sixiwanzia/img/raw/master/20210206115246.png)

### 5. 布隆过滤器

HyperLogLog只能用来进行估算,会存在一定的误差,HyperLogLog主要提供两个方法: pfadd和pfcount.HyperLogLog没有判断集合中是否包含某个元素的方法.但是这种判断集合中是否存在某个元素的业务需求却比较常见.例如我们在刷短视频的时候推送的短视频是会有相似但是不会有重复的短视频,这就涉及到在推送的时候进行去重.解决方案很多,例如可以将用户的浏览历史记录下来,然后每次推送的时候去比较该条消息是否以及推送过给用户,但是这种方案效率低不推荐使用这种方案.可以使用Bloom Filter

#### 5.1 Bloom Filter介绍

Bloom Filter专门用来解决上诉出现的去重问题,使用Bloom Filter不会像使用缓存那么浪费空间,但是会存在一定误差.

Bloom Filter相当于是一个不太精确的set集合.可以使用contains方法来判断集合中是否存在某个元素,但是这个判断并不会特别精确.一般来说通过contains判断某个值不存在那么一定是不存在的,但是判断某个元素是否存在则有可能不存在. 

#### 5.2 Bloom Filter原理

每个Bloom Filter在redis中都会对应一个大型的位数组以及几个不同的无偏hash函数.它的add操作有如下几个步骤:

1. 根据几个不同的hash函数给元素进行hash算一个整数索引值
2. 拿到索引值后对位数组的长度进行取模得到一个位置,每一个hash函数都会得到一个位置
3. 将位数组中对应的位置设置为1

![image-20210207100041465](https://gitee.com/sixiwanzia/img/raw/master/20210207100041.png)

当判断元素是否存在时,依然先对元素进行hsah运算,将运算结果和位数组进行取模,然后去查看位数组对应的位置是否有相应的数据,如果有则该元素可能存在(因为对应的位置也可能时其他元素存储进来的),如果没有则表示一定不存在

Bloom Filter的精确度和所使用的位数组长度成负相关的关系位数组长度越长误判的概率越低当然所需要的空间也越大

#### 5.3 Bloom Filter的安装

Bloom Filter有两种安装方式(官网:https://oss.redislabs.com/redisbloom/Quick_Start/)

1. Docker:

   ```
   docker run -p 6379:6379 --name redis-redisbloom redislabs/rebloom:latest
   ```

2. 自己进行编译安装

   ```
   /*编译*/
   cd redis-6.0.9
   git clone https://github.com/RedisBloom/RedisBloom.git
   cd RedisBloom
   make
   /*运行需要在redis的安装目录下执行*/
   redis-server --loadmodule ./RedisBloom/redisbloom.so
   /*后台启动*/
   redis-server --loadmodule ./RedisBloom/redisbloom.so
   ```

   安装完成后可执行bf.add命令,可以测试安装是否成功

   如果想要默认直接后台启动,我们可以将需要加载的模块在redis.conf中提前配置好即可

   ```
   loadmodule /root/redis-6.0.9/RedisBloom/redisbloom.so
   ```

   ![image-20210207115809394](https://gitee.com/sixiwanzia/img/raw/master/20210207115809.png)

#### 5.4 基本使用

在Bloom Filter中是不支持删除操作的,主要是因为在Bloom Filter是将数据进行hash运算然后存储到位数组中,如果支持删除数据可能会导致删除错误的数据,进而导致数据不准确的问题出现.Bloom Filter中主要是添加(bf.add/bf.madd)和判断是否存在(bf.exists/bf.mexists)

使用jedis操作Bloom Filter

1. 添加Bloom Filter依赖(这个依赖是在jedis3之后才支持的,所以添加的jedis依赖版本需要在3以上)

``` xml
    <dependency>
        <groupId>com.redislabs</groupId>
        <artifactId>jrebloom</artifactId>
        <version>1.2.0</version>
    </dependency>
```

2. 进行测试

```java
    GenericObjectPoolConfig config = new GenericObjectPoolConfig();
        config.setMaxIdle(300);
        config.setMaxTotal(30);
        config.setMaxWaitMillis(90000);
        config.setTestOnBorrow(true);
        JedisPool pool = new JedisPool(config, "192.168.68.131", 6379, 30000,
                "admin01*");
        Client client = new Client(pool);
        //存入数据
        for (int i = 0; i < 100000; i++) {
            client.add("name","test"+i);

        }
        //检查数据是否存在
        boolean exists = client.exists("name", "test9");
        System.out.println("------------>"+exists);
```

默认情况下使用布隆过滤器错误率是0.01,默认的元素大小是100.当元素长度操过100个时错误率会明显上升,但是这俩参数是可以配置的(bf.reserve)

![image-20210219103014306](https://gitee.com/sixiwanzia/img/raw/master/20210219103021.png)

第一个参数是key在设置的时候需要保证key不存在,第二个参数是错误率,错误率与布隆过滤器占用的空间大小成反比错误率越低,所需空间越大,第三个参数是预估长度,如果实际长度超过预估长度则会导致错误率上升

#### 5.5 典型应用场景

解决Redis穿透或者叫缓存击穿的问题

> 假设有一亿条用户数据,(只有在数据量很大的时候说缓存击穿才有意义),现在查询用户需要在数据库中查询,效率低而且数据库压力大,所以会把请求首先在Redis中处理(获取已经缓存在Redis中的用户),Redis中查询不到的数据再到数据库中查询.
>
> 此时假设有一个恶意请求,这个请求携带了很多不存在的用户,针对这哥请求Redis无法拦截,所以会直接到数据库中查询数据.如果一次有大量的恶意请求就会将我们的缓存击穿,甚至是数据库.进而会引起雪崩效应.

为了解决这个问题,我们就可以使用布隆过滤器,我们可以将需要缓存的大量的数据存储到布隆过滤器中.当接收到请求时首先在布隆过滤器中进行一次查询,如果存在再到数据库中查询.否则直接过滤.

### 6. 限流

####  6.1 预备知识

Pipeline (管道)本质上是由客户端提供的一种操作.Pipeline通过调整指定列表的读写顺序可以大幅度压缩IO时间,提高效率

#### 6.2简单限流

``` java
public class RateLimiter {
    private Jedis jedis;
    public RateLimiter(Jedis jedis) {
        this.jedis = jedis;
    }


    /**
     * 限流方法
     * @param user 操作用户,相当于被限流用户
     * @param action 具体操作
     * @param period 时间窗口限流周期 单位s
     * @param maxCount 限流次数
     * @return
     */
    public boolean isAllowed(String user,String action,int period,int maxCount){
       //1. 数据使用zset保存 首先生成key
        String key = user+"-"+action;
        //2. 获取当前时间
        long newTime = System.currentTimeMillis();
        //3. 建立管道
        Pipeline pipelined = jedis.pipelined();
        pipelined.multi();
        //4.将当前操作存储
        pipelined.zadd(key,newTime,String.valueOf(newTime));
        //5.移除时间窗之外的数据
        pipelined.zremrangeByScore(key,0,newTime-period*1000);
        //6. 统计剩下的key
        Response<Long> zcard = pipelined.zcard(key);
        //7. 将当前key设置过期时间,过期时间就是时间窗,可以不设置过期时间,第5步会自动移除
        pipelined.expire(key,period+1);
        // 关闭管道
        pipelined.exec();
        pipelined.close();
        //8. 返回比较时间窗内的操作数
        return  zcard.get()<=maxCount;
    }

    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(j->{
            RateLimiter rateLimiter = new RateLimiter(j);
            for (int i = 0; i < 20; i++) {
                System.out.println(rateLimiter.isAllowed("lean", "sixi", 5, 3));
            }
        });
    }
}

```

仅限在数据量较小情况下使用

#### 6.3深入限流操作

在Redis4.0开始,提供了一个Redis-cell,这个模块使用了==**漏斗算法**==,提供了一个十分方便的限流指令.

漏斗算法: 见名知意,就像名字一样请求从漏斗大口进,然后从小口进入到系统.最终被系统所接收到的请求的数量说固定的.简单的实现方式可以考虑使用队列进行处理,固定队列容量超出容量后的请求可以直接丢弃,获取请求只需要固定好速率去取即可.

使用漏斗算法首先需要安装 Redis-cell模块

https://github.com/brandur/redis-cell/releases

安装步骤:

由于Redis-cell是使用rust进行开发的,直接下载安装后还需要编译比较麻烦,建议下载编译好的文件

```
# 设置下载位置
mkdir redis-cell

# 在该位置 下载包
wget https://github.com/brandur/redis-cell/releases/download/v0.2.5/redis-cell-v0.2.5-x86_64-unknown-linux-gnu.tar.gz

# 解压
tar -zxvf redis-cell/releases/download/v0.2.5/redis-cell-v0.2.5-x86_64-unknown-linux-gnu.tar.gz

# pwd 复制位置 修改redis.conf文件 加载新模块
vim redis.conf
# :/loadm 就能找到了
loadmodule pwd刚才的地址/libredis_cell.so
# esc :wq
# 重启redis
redis-server redis.conf
# 登陆进来 输入CL.THROTTLE命令 若是存在 说明安装成功
redis-cli -a 密码
CL.THROTTLE
```

tips: 安装模块Redis-cell需要glibc支持,以本机为例是需要glibc 2.18,如果没有需要先进行安装

```
#1.获取压缩文件 
wget  http://ftp.gnu.org/gnu/glibc/glibc-2.17.tar.gz 
#2. 解压
tar -zxvf glibc-2.14.tar.gz
#3.由于编译需要在不同的目录下进行建议创建一个新的文件夹
mkdir bulid
#4. 进入编译文件夹
cd bulid\
#5.编译(../config 这里建议填写绝对路径,就是configure的路径)
../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
#6. 编译
make
# 7. 安装
make install

```

如果仍未成功加载redis-cell模块可以考虑采用低版本redis-cell 目前本机采用的是redis-cell V0.2.4

CL.THROTTLE这个命令一共有5个参数

1. 第一个参数是key
2. 第二个参数是漏斗的容量,只要小于该容量就不会被限制
3. 第三个参数是时间窗内可以操作的次数
4. 第四个参数是时间窗
5. 第五个参数是每次漏出数量

执行完成后返回值也有5个

1. 第一个返回值是0的话表示运行,1表示拒绝
2. 第二参数是漏斗的容量
3. 第三个参数是漏斗的剩余空间
4. 第四个参数是如果拒绝多长时间后可以重试
5. 第五个参数是多长时间后漏斗会置空

实例代码:

由于redis-cell属于第三方的模块jedis是不会自带有这个模块,需要自行添加依赖,但是lettuce可以支持自定义命令相对而言会比较方便

#### 6.4 lettuce拓展

首先定义一个命令接口:

```java
public interface RedisCommandInterface extends Commands {
    @Command("CL.THROTTLE ?0 ?1 ?2 ?3 ?4")
    List<Object> throttle(String key,Long init,Long count, Long period,Long quota);
}
```

这样就相当于已经自定义了一个命令

调用:

```java
 public static void main(String[] args) {
        RedisClient redisClient = RedisClient.create("redis://admin01*@192.168.68.131");
        StatefulRedisConnection<String, String> connect = redisClient.connect();
        RedisCommandFactory factory= new RedisCommandFactory(connect);
        RedisCommandInterface commandInterface = factory.getCommands(RedisCommandInterface.class);
        System.out.println(commandInterface.throttle("lean-redis-cell", 10L, 10L, 6L, 1L));

    }
```

### 7. GeoHash 算法介绍

Redis从3.2开始提供了Geo模块,使用的算法叫GeoHash,这个算法是通用的不仅仅再Redis中有

#### 7.1 GeoHash 

核心思想: Geohash算法就是将经纬度编码,将二维变一维,编码成一维字符串,给地址位置分区的一种算法.

经纬度的划分: 

以经过伦敦格林尼治天文台旧址的经线为0度经纬线,向东就是东经,向西就是西经.如果我们将西经定义为负数那么经度范围是[-180,180].

纬度北纬90度到南纬90度,如果将南纬定义为负那么纬度的范围就是[-90,90]

接下来以本初子午线和赤道为界,我们可以将地球上的任意一个点都分配到一个二维坐标中

GeoHash算法就是基于这样的思想不停的对某一个范围的地理位置做分割,划分的次数阅读区域就阅读,区域面积就越小也就约精确.

##### 7.1.1 GeoHash具体算法:

以北京天安门广场为例(39.9053908600,116.3980007200):

1. 纬度的范围再-90到90之间,中间值是0,对于39.9053908600这个值落在0到90之间因此得到的值为1
2. 在0到90之间的中间值为45,39.9053908600这个值落在0到45之间因此得到0
3. 在0到45的中间值为22.5,39.9053908600这个值落在22.5到45之间因此得到1
4. 递归上述过程39.9053908600总是属于某个区间[a,b]。随着每次迭代区间[a,b]总在缩小，并越来越逼近39.9053908600

这样我们在前三步计算后得到的纬度二进制数据是101

按照同样步骤,可以得到经度二进制是110

接下来将经纬度进行合并,经度占偶数位,纬度占奇数位,所以这个例子中得到的二进制数据是: 111001

然后按照base32(0-9,b-z,去掉a i l 0)对合并后的二进制数据进行编码,编码时需要先将二进制数据转换成10进制在进行编码.

可以在geoHash.org上将编码后得到的字符串进行解析

总结: 如果给定的纬度x（39.928167）属于左区间，则记录0，如果属于右区间则记录1，序列的长度跟给定的区间划分次数有关

##### 7.1.2 GeoHash特点:

1. 用字符串表示经纬度

2. GeoHash表示的是一个区域而不是一个点,方便保护用户隐私,也方便缓存
3. 编码格式是有规律的,假设一个地址编码是123,另一个地址编码是123456,从字符串上就可以看出来123456是处于123之中的

缺点:

1. 由于GeoHash是将区域划分为一个个规则矩形，并对每个矩形进行编码，这样在查询附近POI信息时会导致以下问题，比如红色的点是我们的位  置，绿色的两个点分别是附近的两个餐馆，但是在查询的时候会发现距离较远餐馆的GeoHash编码与我们一样（因为在同一个GeoHash区域块上），而 较近餐馆的GeoHash编码与我们不一致。这个问题往往产生在边界处。解决的思路很简单，我们查询时，除了使用定位点的GeoHash编码进行匹配外，还使用周围8个区域的GeoHash编码，这样可以避免这个问题。

![image-20210809105408366](https://gitee.com/sixiwanzia/img/raw/master/20210809105415.png)

2. 我们已经知道现有的GeoHash算法使用的是Peano空间填充曲线，这种曲线会产生突变，造成了编码虽然相似但距离可能相差很大的问题，因此在查询附近餐馆时候，首先筛选GeoHash编码相似的POI点，然后进行实际距离计算。

#### 7.2 在Redis中使用

添加地址:

```lua
GEOADD city 116.3980007200 39.9053908600 beijing	
GEOADD city 114.0592002900 22.5536230800 shenzhen	
```

查看两个地址之间的距离:

```
GEODIST city beijing shenzhen km
"1942.5435"
```

获取元素的位置:

```
GEOPOS city beijing
   1) "116.39800339937210083"
   2) "39.90539144357683909"
```

获取元素Hash值:

```
GEOHASH city beijing
"wx4g08w3y00"
```

查看附件地址:

```
# 以北京为中心方圆两百公里内找出3个按照从远近进行排列,事先已经在city这个key中存储了 北京,长沙,遵义三个元素
GEORADIUSBYMEMBER city beijing 200 km count 3 asc 
 "beijing"
 "zhunyi"
 # 可以附加其他参数
  GEORADIUSBYMEMBER city beijing 200 km withdist withhash withcoord count 3 asc
  #可以直接使用经纬度进行查找
  GEORADIUS city 116 39 200 km withcoord withdist withhash count 3 
1) 1) "beijing"
   2) "106.3461"
   3) (integer) 4069885364415061
   4) 1) "116.39800339937210083"
      2) "39.90539144357683909"
2) 1) "zhunyi"
   2) "107.3581"
   3) (integer) 4069885642983710
   4) 1) "116.41338318586349487"
      2) "39.9109247398676743"
```

### 8.Redis之Scan

#### 8.1 简单介绍

scan实际上是keys的一个升级版,keys主要是用来查询key的,在查询过程中可以使用通配符.keys虽然使用较为方便,但是没有分页,同时redis是单线程,keys使用的是遍历的方式进行查找,当key的数量很多的时候就会影响效率.

为了解决keys存在的问题,在redis2.8开始,引入了scan,scan具备了keys的功能,但是不会阻塞线程,而且可以控制每次返回的数据数量

#### 8.2 基本用法

首先准备测试数据：

```java
public static void main(String[] args) {
        Redis redis = new Redis();
        redis.excute(jedis -> {
            for (int i = 0; i < 10000; i++) {
                jedis.set("k"+i,"v"+i);
            }
        });
    }
```

Scan一共提供了三个参数 ,第一个是cursor(游标),第二个是key,第三个是limit.

Redis使用了Hash表作为底层实现，原因不外乎高效且实现简单。说到Hash表，很多Java程序员第一反应就是HashMap。没错，Redis底层key的存储结构就是类似于HashMap那样数组+链表的结构。其中第一维的数组大小为2n(n>=0)。每次扩容数组长度扩大一倍。

scan命令就是对这个一维数组进行遍历。每次返回的游标值也都是这个数组的索引。limit参数表示遍历多少个数组的元素，将这些元素下挂接的符合条件的结果都返回。因为每个元素下挂接的链表大小不同，所以每次返回的结果数量也就不同。

```
scan 0 match k8* count 1000 
```

#### 8.3 原理

Scan的遍历顺序 :

假设目前有三条数据 

```
127.0.0.1:6379> keys *
1) "db_number"
2) "key1"
3) "myKey"
127.0.0.1:6379> scan 0 MATCH * COUNT 1
1) "2"
2) 1) "db_number"
127.0.0.1:6379> scan 2 MATCH * COUNT 1
1) "1"
2) 1) "myKey"
127.0.0.1:6379> scan 1 MATCH * COUNT 1
1) "3"
2) 1) "key1"
127.0.0.1:6379> scan 3 MATCH * COUNT 1
1) "0"
2) (empty list or set)
```

在使用scan命令遍历游标时返回的顺序是 0->2->1->3,这样子看上去确实没有规律可言,但是当我们把它转换为二进制数据 00->10->01->11,这样一看我们会发现每次这个序列总是会在高位的地方加一.普通的二进制加法是从右往左进行、相加.而这个序列是从左往右相加、进位的.这一点我们在redis的源码中也得到印证.

```
v = rev(v);
v++;
v = rev(v);
```

意思就是,将游标倒置,在自增后,再将游标倒置,这也就是我们所说的高位进一的操作.

没有按照自然数大小的顺序进行遍历的原因是为了解决字典扩容和字典缩容的问题

![image-20210810110815066](https://gitee.com/sixiwanzia/img/raw/master/20210810110815.png)

我们来看一下在SCAN遍历过程中,发生扩容时,遍历会如何进行.加入我们原始的数组有4个元素,也就是索引有两位,这时需要把它扩充成3位,原来挂接在xx下的所有元素被分配到0xx和1xx下.在上图中,当我们即将遍历10时,dict进行了rehash,这时,scan命令会从010开始遍历,而000和100（原00下挂接的元素）不会再被重复遍历。

再来看看缩容的情况。假设dict从3位缩容到2位，当即将遍历110时，dict发生了缩容，这时scan会遍历10。这时010下挂接的元素会被重复遍历，但010之前的元素都不会被重复遍历了。所以，缩容时还是可能会有些重复元素出现的。

#### 8.4 其他指令

scan 是一系列的指令，除了遍历所有的 key 之外，也可以遍历某一个类型的 key，对应的命令有：

 zscan-->zset

 hscan-->hash

 sscan-->se



