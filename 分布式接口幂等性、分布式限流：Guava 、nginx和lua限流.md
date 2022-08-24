# 分布式接口幂等性、分布式限流：Guava 、nginx和lua限流

**一、接口幂等性**

接口幂等性就是用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用。举个最简单的例子，那就是支付，用户购买商品后支付，支付扣款成功，但是返回结果的时候网络异常，此时钱已经扣了，用户再次点击按钮，此时会进行第二次扣款，返回结果成功，用户查询余额返发现多扣钱了，流水记录也变成了两条,这就没有保证接口的幂等性。

幂等性的核心思想：通过唯一的业务单号保障幂等性，非并发的情况下，查询业务单号有没有操作过，没有则执行操作，并发情况下，这个操作过程需要加锁。

#### 1、Update操作的幂等性

**1）根据唯一业务号去更新数据**

通过版本号的方式，来控制update的操作的幂等性，用户查询出要修改的数据，系统将数据返回给页面，将数据版本号放入隐藏域，用户修改数据，点击提交，将版本号一同提交给后台，后台使用版本号作为更新条件

```
update set version = version +1 ,xxx=${xxx} where id =xxx and version = ${version};  
```

#### 2、使用Token机制，保证update、insert操作的幂等性

**1）没有唯一业务号的update与insert操作**

进入到注册页时，后台统一生成Token， 返回前台隐藏域中，用户在页面点击提交时，将Token一同传入后台，使用Token获取分布式锁，完成Insert操作，执行成功后，不释放锁，等待过期自动释放。

### **二、分布式限流**

#### 1、分布式限流的几种维度

时间 限流基于某段时间范围或者某个时间点，也就是我们常说的“时间窗口”，比如对每分钟、每秒钟的时间窗口做限定 资源 基于可用资源的限制，比如设定最大访问次数，或最高可用连接数

上面两个维度结合起来看，限流就是在某个时间窗口对资源访问做限制，比如设定每秒最多100个访问请求。但在真正的场景里，我们不止设置一种限流规则，而是会设置多个限流规则共同作用，主要的几种限流规则如下：

**1）QPS和连接数控制**

针对上图中的连接数和QPS(query per second)限流来说，我们可以设定IP维度的限流，也可以设置基于单个服务器的限流。在真实环境中通常会设置多个维度的限流规则，比如设定同一个IP每秒访问频率小于10，连接数小于5，再设定每台机器QPS最高1000，连接数最大保持200。

更进一步，我们可以把某个服务器组或整个机房的服务器当做一个整体，设置更high-level的限流规则，这些所有限流规则都会共同作用于流量控制。

**2）传输速率**

对于“传输速率”大家都不会陌生，比如资源的下载速度。有的网站在这方面的限流逻辑做的更细致，比如普通注册用户下载速度为100k/s，购买会员后是10M/s，这背后就是基于用户组或者用户标签的限流逻辑。

**3）黑白名单**

黑白名单是各个大型企业应用里很常见的限流和放行手段，而且黑白名单往往是动态变化的。举个例子，如果某个IP在一段时间的访问次数过于频繁，被系统识别为机器人用户或流量攻击，那么这个IP就会被加入到黑名单，从而限制其对系统资源的访问，这就是我们俗称的“封IP”。

我们平时见到的爬虫程序，比如说爬知乎上的美女图片，或者爬券商系统的股票分时信息，这类爬虫程序都必须实现更换IP的功能，以防被加入黑名单。有时我们还会发现公司的网络无法访问12306这类大型公共网站，这也是因为某些公司的出网IP是同一个地址，因此在访问量过高的情况下，这个IP地址就被对方系统识别，进而被添加到了黑名单。使用家庭宽带的同学们应该知道，大部分网络运营商都会将用户分配到不同出网IP段，或者时不时动态更换用户的IP地址。

白名单就更好理解了，相当于御赐金牌在身，可以自由穿梭在各种限流规则里，畅行无阻。比如某些电商公司会将超大卖家的账号加入白名单，因为这类卖家往往有自己的一套运维系统，需要对接公司的IT系统做大量的商品发布、补货等等操作。

**4）分布式环境**

所谓的分布式限流，其实道理很简单，一句话就可以解释清楚。分布式区别于单机限流的场景，它把整个分布式环境中所有服务器当做一个整体来考量。比如说针对IP的限流，我们限制了1个IP每秒最多10个访问，不管来自这个IP的请求落在了哪台机器上，只要是访问了集群中的服务节点，那么都会受到限流规则的制约。

从上面的例子不难看出，我们必须将限流信息保存在一个“中心化”的组件上，这样它就可以获取到集群中所有机器的访问状态，目前有两个比较主流的限流方案：

网关层限流

- 将限流规则应用在所有流量的入口处

中间件限流

- 将限流信息存储在分布式环境中某个中间件里（比如Redis缓存），每个组件都可以从这里获取到当前时刻的流量统计，从而决定是拒绝服务还是放行流量

#### 2、限流方案常用算法讲解

##### 1）令牌桶算法

Token Bucket令牌桶算法是目前应用最为广泛的限流算法，顾名思义，它有以下两个关键角色：

- 令牌 获取到令牌的Request才会被处理，其他Requests要么排队要么被直接丢弃
- 桶 用来装令牌的地方，所有Request都从这个桶里面获取令牌

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAm8H0a8AQx38EIe3lIEdywfDFGbCUpcygn8f4QoXugbuLnqSpAISDqnwtxMI7NQNHRs9Vk5nbx9A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图片

**令牌生成**

这个流程涉及到令牌生成器和令牌桶，前面我们提到过令牌桶是一个装令牌的地方，既然是个桶那么必然有一个容量，也就是说令牌桶所能容纳的令牌数量是一个固定的数值。

对于令牌生成器来说，它会根据一个预定的速率向桶中添加令牌，比如我们可以配置让它以每秒100个请求的速率发放令牌，或者每分钟50个。注意这里的发放速度是匀速，也就是说这50个令牌并非是在每个时间窗口刚开始的时候一次性发放，而是会在这个时间窗口内匀速发放。

在令牌发放器就是一个水龙头，假如在下面接水的桶子满了，那么自然这个水（令牌）就流到了外面。在令牌发放过程中也一样，令牌桶的容量是有限的，如果当前已经放满了额定容量的令牌，那么新来的令牌就会被丢弃掉。

**令牌获取**

每个访问请求到来后，必须获取到一个令牌才能执行后面的逻辑。假如令牌的数量少，而访问请求较多的情况下，一部分请求自然无法获取到令牌，那么这个时候我们可以设置一个“缓冲队列”来暂存这些多余的令牌。

缓冲队列其实是一个可选的选项，并不是所有应用了令牌桶算法的程序都会实现队列。当有缓存队列存在的情况下，那些暂时没有获取到令牌的请求将被放到这个队列中排队，直到新的令牌产生后，再从队列头部拿出一个请求来匹配令牌。

当队列已满的情况下，这部分访问请求将被丢弃。在实际应用中我们还可以给这个队列加一系列的特效，比如设置队列中请求的存活时间，或者将队列改造为PriorityQueue，根据某种优先级排序，而不是先进先出。算法是死的，人是活的，先进的生产力来自于不断的创造，在技术领域尤其如此。

##### 2）漏桶算法

Leaky Bucket

![图片](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAm8H0a8AQx38EIe3lIEdywV2z4moFZkz7II8aOylfO5DGl8WkfrPYO0pFOL55SEhStfj2iavXAa9w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图片

漏桶算法的前半段和令牌桶类似，但是操作的对象不同，令牌桶是将令牌放入桶里，而漏桶是将访问请求的数据包放到桶里。同样的是，如果桶满了，那么后面新来的数据包将被丢弃。

漏桶算法的后半程是有鲜明特色的，它永远只会以一个恒定的速率将数据包从桶内流出。打个比方，如果我设置了漏桶可以存放100个数据包，然后流出速度是1s一个，那么不管数据包以什么速率流入桶里，也不管桶里有多少数据包，漏桶能保证这些数据包永远以1s一个的恒定速度被处理。

**漏桶 vs 令牌桶的区别**

根据它们各自的特点不难看出来，这两种算法都有一个“恒定”的速率和“不定”的速率。令牌桶是以恒定速率创建令牌，但是访问请求获取令牌的速率“不定”，反正有多少令牌发多少，令牌没了就干等。而漏桶是以“恒定”的速率处理请求，但是这些请求流入桶的速率是“不定”的。

从这两个特点来说，漏桶的天然特性决定了它不会发生突发流量，就算每秒1000个请求到来，那么它对后台服务输出的访问速率永远恒定。而令牌桶则不同，其特性可以“预存”一定量的令牌，因此在应对突发流量的时候可以在短时间消耗所有令牌，其突发流量处理效率会比漏桶高，但是导向后台系统的压力也会相应增多。

#### 3、分布式限流的主流方案

这里主要讲nginx和lua的限流，gateway和hystrix放在后面springcloud中讲

##### 1）Guava RateLimiter客户端限流

1.引入maven

```
<dependency>  
    <groupId>com.google.guava</groupId>  
    <artifactId>guava</artifactId>  
    <version>18.0</version>  
</dependency>  
```

2.编写Controller

```
@RestController  
@Slf4j  
public class Controller{  
    //每秒钟可以创建两个令牌  
    RateLimiter limiter = RateLimiter.create(2.0);  
      
    //非阻塞限流  
    @GetMapping("/tryAcquire")  
    public String tryAcquire(Integer count){  
        //count 每次消耗的令牌  
        if(limiter.tryAcquire(count)){  
            log.info("成功，允许通过，速率为{}",limiter.getRate());  
            return "success";  
        }else{  
            log.info("错误，不允许通过，速率为{}",limiter.getRate());  
            return "fail";  
        }  
    }  
      
    //限定时间的非阻塞限流  
    @GetMapping("/tryAcquireWithTimeout")  
    public String tryAcquireWithTimeout(Integer count, Integer timeout){  
        //count 每次消耗的令牌  timeout 超时等待的时间  
        if(limiter.tryAcquire(count,timeout,TimeUnit.SECONDS)){  
            log.info("成功，允许通过，速率为{}",limiter.getRate());  
            return "success";  
        }else{  
            log.info("错误，不允许通过，速率为{}",limiter.getRate());  
            return "fail";  
        }  
    }  
      
    //同步阻塞限流  
    @GetMapping("/acquire")  
    public String acquire(Integer count){  
        limiter.acquire(count);  
        log.info("成功，允许通过，速率为{}",limiter.getRate());  
        return "success";  
    }  
}  
```

##### 2）基于Nginx的限流

**1.iP限流**

1.编写Controller

```
@RestController  
@Slf4j  
public class Controller{  
    //nginx测试使用  
    @GetMapping("/nginx")  
    public String nginx(){  
        log.info("Nginx success");  
    }  
}  
```

2.修改host文件，添加一个网址域名

```
127.0.0.1   www.test.com  
```

3.修改nginx，将步骤2中的域名，添加到路由规则当中

打开nginx的配置文件

```
vim /usr/local/nginx/conf/nginx.conf  
```

添加一个服务

```
#根据IP地址限制速度  
#1）$binary_remote_addr   binary_目的是缩写内存占用，remote_addr表示通过IP地址来限流  
#2）zone=iplimit:20m   iplimit是一块内存区域（记录访问频率信息），20m是指这块内存区域的大小  
#3）rate=1r/s  每秒放行1个请求  
limit_req_zone $binary_remote_addr zone=iplimit:20m rate=1r/s;  
  
server{  
    server_name www.test.com;  
    location /access-limit/ {  
        proxy_pass http://127.0.0.1:8080/;  
          
        #基于ip地址的限制  
        #1）zone=iplimit 引用limit_rep_zone中的zone变量  
        #2）burst=2  设置一个大小为2的缓冲区域，当大量请求到来，请求数量超过限流频率时，将其放入缓冲区域  
        #3）nodelay   缓冲区满了以后，直接返回503异常  
        limit_req zone=iplimit burst=2 nodelay;  
    }  
}  
```

4.访问地址，测试是否限流

> www.test.com/access-limit/nginx

**2.多维度限流**

1.修改nginx配置

```
#根据IP地址限制速度  
limit_req_zone $binary_remote_addr zone=iplimit:20m rate=10r/s;  
#根据服务器级别做限流  
limit_req_zone $server_name zone=serverlimit:10m rate=1r/s;  
#根据ip地址的链接数量做限流  
limit_conn_zone $binary_remote_addr zone=perip:20m;  
#根据服务器的连接数做限流  
limit_conn_zone $server_name zone=perserver:20m;  
  
  
server{  
    server_name www.test.com;  
    location /access-limit/ {  
        proxy_pass http://127.0.0.1:8080/;  
          
        #基于ip地址的限制  
        limit_req zone=iplimit burst=2 nodelay;  
        #基于服务器级别做限流  
        limit_req zone=serverlimit burst=2 nodelay;  
        #基于ip地址的链接数量做限流  最多保持100个链接  
        limit_conn zone=perip 100;  
        #基于服务器的连接数做限流 最多保持100个链接  
        limit_conn zone=perserver 1;  
        #配置request的异常返回504（默认为503）  
        limit_req_status 504;  
        limit_conn_status 504;  
    }  
      
     location /download/ {  
        #前100m不限制速度  
        limit_rate_affer 100m;  
        #限制速度为256k  
        limit_rate 256k;  
     }  
}  
```

##### 3）基于Redis+Lua的分布式限流

**1.Lua脚本**

Lua是一个很小巧精致的语言，它的诞生（1993年）甚至比JDK 1.0还要早。Lua是由标准的C语言编写的，它的源码部分不过2万多行C代码，甚至一个完整的Lua解释器也就200k的大小。

Lua往大了说是一个新的编程语言，往小了说就是一个脚本语言。对于有编程经验的同学，拿到一个Lua脚本大体上就能把业务逻辑猜的八九不离十了。

Redis内置了Lua解释器，执行过程保证原子性

**2.Lua安装**

安装Lua：

1.参考`http://www.lua.org/ftp/`教程，下载5.3.5_1版本，本地安装

如果你使用的是Mac，那建议用brew工具直接执行brew install lua就可以顺利安装，有关brew工具的安装可以参考`https://brew.sh/`网站，使用brew安装后的目录在`/usr/local/Cellar/lua/5.3.5_1`

2.安装IDEA插件，在IDEA->Preferences面板，Plugins，里面Browse repositories，在里面搜索lua，然后就选择同名插件lua。安装好后重启IDEA

3.配置Lua SDK的位置：`IDEA->File->Project Structure`,选择添加Lua，路径指向Lua SDK的bin文件夹

4.都配置好之后，在项目中右键创建Module，左侧栏选择lua，点下一步，选择lua的sdk，下一步，输入lua项目名，完成

**3.编写hello lua**

```
print 'Hello Lua'  
```

**4.编写模拟限流**

```
-- 模拟限流  
  
-- 用作限流的key  
local key = 'my key'  
  
-- 限流的最大阈值  
local limit = 2  
  
-- 当前限流大小  
local currentLimit = 2  
  
-- 是否超过限流标准  
if currentLimit + 1 > limit then  
    print 'reject'  
    return false  
else  
    print 'accept'  
    return true  
end  
```

**5.限流组件封装**

1.添加maven

```
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-data-redis</artifactId>  
</dependency>  
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-aop</artifactId>  
</dependency>  
<dependency>  
    <groupId>com.google.guava</groupId>  
    <artifactId>guava</artifactId>  
    <version>18.0</version>  
</dependency>  
```

2.添加Spring配置

不是重要内容就随便写点，主要就是把reids配置一下

```
server.port=8080  
  
spring.redis.database=0  
spring.redis.host=localhost  
spring.redis.port=6376  
```

3.编写限流脚本

lua脚本放在resource目录下就可以了

```
-- 获取方法签名特征  
local methodKey = KEYS[1]  
redis.log(redis.LOG_DEBUG,'key is',methodKey)  
  
-- 调用脚本传入的限流大小  
local limit = tonumber(ARGV[1])  
  
-- 获取当前流量大小  
local count = tonumber(redis.call('get',methodKey) or "0")  
  
--是否超出限流值  
if count + 1 >limit then  
    -- 拒绝访问  
    return false  
else  
    -- 没有超过阈值  
    -- 设置当前访问数量+1  
    redis.call('INCRBY',methodKey,1)  
    -- 设置过期时间  
    redis.call('EXPIRE',methodKey,1)  
    -- 放行  
    return true  
end  
```

4.使用`spring-data-redis`组件集成Lua和Redis

创建限流类

```
@Service  
@Slf4j  
public class AccessLimiter{  
    @Autowired  
    private StringRedisTemplate stringRedisTemplate;  
    @Autowired  
    private RedisScript<Boolean> rateLimitLua;  
  
    public void limitAccess(String key,Integer limit){  
        boolean acquired = stringRedisTemplate.execute(  
            rateLimitLua,//lua脚本的真身  
            Lists.newArrayList(key),//lua脚本中的key列表  
            limit.toString()//lua脚本的value列表  
        );  
  
        if(!acquired){  
            log.error("Your access is blocked,key={}",key);  
            throw new RuntimeException("Your access is blocked");  
        }  
    }  
}  
```

创建配置类

```
@Configuration  
public class RedisConfiguration{  
    public RedisTemplate<String,String> redisTemplate(RedisConnectionFactory factory){  
        return new StringRedisTemplate(factory);  
    }  
      
    public DefaultRedisScript loadRedisScript(){  
        DefaultRedisScript redisScript = new DefaultRedisScript();  
        redisScript.setLocation(new ClassPathResource("rateLimiter.lua"));  
        redisScript.setResultType(java.lang.Boolean.class);  
        return redisScript;  
    }  
}  
```

5.在Controller中添加测试方法验证限流效果

```
@RestController  
@Slf4j  
public class Controller{  
    @Autowired  
    private AccessLimiter accessLimiter;  
      
    @GetMapping("test")  
    public String test(){  
        accessLimiter.limitAccess("ratelimiter-test",1);  
        return "success";  
    }  
}   
```

**6.编写限流注解**

1.新增注解

```
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface AccessLimiterAop{  
    int limit();  
      
    String methodKey() default "";  
}  
```

2.新增切面

```
@Slf4j  
@Aspect  
@Component  
public class AccessLimiterAspect{  
    @Autowired  
    private AccessLimiter  accessLimiter;  
  
    //根据注解的位置，自己修改  
    @Pointcut("@annotation(com.gyx.demo.annotation.AccessLimiter)")  
    public void cut(){  
        log.info("cut");  
    }  
      
    @Before("cut()")  
    public void before(JoinPoint joinPoint){  
        //获取方法签名，作为methodkey  
        MethodSignature signature =(MethodSignature) joinPoint.getSignature();  
        Method method = signature.getMethod();  
        AccessLimiterAop annotation = method.getAnnotation(AccessLimiterAop.class);  
          
        if(annotation == null){  
            return;  
        }  
        String key = annotation.methodKey();  
        Integer limit = annotation.limit();  
        //如果没有设置methodKey，就自动添加一个  
        if(StringUtils.isEmpty(key)){  
            Class[] type = method.getParameterType();  
            key = method.getName();  
            if (type != null){  
                String paramTypes=Arrays.stream(type)  
                    .map(Class::getName)  
                    .collect(Collectors.joining(","));  
                    key += "#"+paramTypes;  
            }  
        }  
          
        //调用redis  
        return accessLimiter.limitAccess(key,limit);  
    }  
}  
```

3.在Controller中添加测试方法验证限流效果

```
@RestController  
@Slf4j  
public class Controller{  
    @Autowired  
    private AccessLimiter accessLimiter;  
      
    @GetMapping("test")  
    @AccessLImiterAop(limit =1)  
    public String test(){  
        return "success";  
    }  
} 
```