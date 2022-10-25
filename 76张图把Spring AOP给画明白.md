# 76张图把Spring AOP给画明白



![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ZOiaCAdqusS8wVcGwMav6T622aMmdliaej73IfFElVK6n36FGHJv1iaTg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

# 1. 基础知识

## 1.1 什么是 AOP ？

AOP 的全称是 “Aspect Oriented Programming”，即**面向切面编程**。

在 AOP 的思想里面，周边功能（比如性能统计，日志，事务管理等）被定义为**切面**，核心功能和切面功能分别独立进行开发，然后**把核心功能和切面功能“编织”在一起，这就叫 AOP。**

AOP 能够将那些与业务无关，却为业务模块所共同调用的逻辑封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。

## 1.2 AOP 基础概念

- **连接点(Join point)**：能够被拦截的地方，Spring AOP 是基于动态代理的，所以是方法拦截的，每个成员方法都可以称之为连接点；
- **切点(Poincut)**：每个方法都可以称之为连接点，我们具体定位到某一个方法就成为切点；
- **增强/通知(Advice)**：表示添加到切点的一段逻辑代码，并定位连接点的方位信息，简单来说就定义了是干什么的，具体是在哪干；
- **织入(Weaving)**：将增强/通知添加到目标类的具体连接点上的过程；
- **引入/引介(Introduction)**：允许我们向现有的类添加新方法或属性，是一种特殊的增强；
- **切面(Aspect)**：切面由切点和增强/通知组成，它既包括了横切逻辑的定义、也包括了连接点的定义。

上面的解释偏官方，下面用“方言”再给大家解释一遍。

- 切入点（Pointcut）：在哪些类，哪些方法上切入（**where**）；
- 通知（Advice）：在方法执行的什么时机（**when**：方法前/方法后/方法前后）做什么（**what**：增强的功能）；
- 切面（Aspect）：切面 = 切入点 + 通知，通俗点就是在什么时机，什么地方，做什么增强；
- 织入（Weaving）：把切面加入到对象，并创建出代理对象的过程，这个由 Spring 来完成。



**5 种通知的分类：**

- **前置通知(Before Advice)**：在目标方法被调用前调用通知功能；
- **后置通知(After Advice)**：在目标方法被调用之后调用通知功能；
- **返回通知(After-returning)**：在目标方法成功执行之后调用通知功能；
- **异常通知(After-throwing)**：在目标方法抛出异常之后调用通知功能；
- **环绕通知(Around)**：把整个目标方法包裹起来，在被调用前和调用之后分别调用通知功能。

## 1.3 AOP 简单示例

新建 Louzai 类：

```
@Data
@Service
public class Louzai {

    public void everyDay() {
        System.out.println("睡觉");
    }
}
```

添加 LouzaiAspect 切面：

```
@Aspect
@Component
public class LouzaiAspect {
    
    @Pointcut("execution(* com.java.Louzai.everyDay())")
    private void myPointCut() {
    }

    // 前置通知
    @Before("myPointCut()")
    public void myBefore() {
        System.out.println("吃饭");
    }

    // 后置通知
    @AfterReturning(value = "myPointCut()")
    public void myAfterReturning() {
        System.out.println("打豆豆。。。");
    }
}
```

applicationContext.xml 添加：

```
<!--启用@Autowired等注解-->
<context:annotation-config/>
<context:component-scan base-package="com" />
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

程序入口：

```
public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context =new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
        Louzai louzai = (Louzai) context.getBean("louzai");
        louzai.everyDay();
    }
}
```

输出：

```
吃饭
睡觉
打豆豆。。。
```

这个示例非常简单，“睡觉” 加了前置和后置通知，但是 Spring 在内部是如何工作的呢？

## 1.4 Spring AOP 工作流程

为了方便大家能更好看懂后面的源码，我先整体介绍一下源码的执行流程，让大家有一个整体的认识，否则容易被绕进去。

整个 Spring AOP 源码，其实分为 3 块，我们会结合上面的示例，给大家进行讲解。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4sppibN4qOsEOpUqXyM9l9MhMjY0Vgdy6Gdhe1ZJgiaCACMcXUjmqgricg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

第一块就是**前置处理**，我们在创建 Louzai Bean 的前置处理中，会遍历程序所有的切面信息，然后**将切面信息保存在缓存中**，比如示例中 LouzaiAspect 的所有切面信息。

第二块就是**后置处理**，我们在创建 Louzai Bean 的后置处理器中，里面会做两件事情：

- **获取 Louzai 的切面方法**：首先会从缓存中拿到所有的切面信息，和 Louzai 的所有方法进行匹配，然后找到 Louzai 所有需要进行 AOP 的方法。
- **创建 AOP 代理对象**：结合 Louzai 需要进行 AOP 的方法，选择 Cglib 或 JDK，创建 AOP 代理对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4WRpo2GS2EMVJXstSczS7oSMoyITaqvAdzniatlCcpxuvlPrML9vQ08Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

第三块就是**执行切面**，通过“责任链 + 递归”，去执行切面。

# 2. 源码解读

> 注意：Spring 的版本是 5.2.15.RELEASE，否则和我的代码不一样！！！

除了原理部分，上面的知识都不难，下面才是我们的重头戏，让你跟着楼仔，走一遍代码流程。

## 2.1 代码入口

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ka2MC8zvJBEfzU83j3Uacbnd9HQm9483CibcVAhtyx2Mvn7lW772NZA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4gdIvbzzKlicJMl95eyfwUcobBvvGlRMlYGKKsgMM2giaXYumrF8jKgow/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这里需要多跑几次，把前面的 beanName 跳过去，只看 louzai。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4qGdtcTDS347xexkHQpMmRdsfAQUic0yJFbOWc4wDxs4lZwF0asHenSQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4rBIY0onVkU3mowDYzRLTnd0TQRoIyicjE8wWjl7pTYt8tv3CObRB6Bw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

进入 doGetBean()，进入创建 Bean 的逻辑。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4LrUR2C7VXh7sJ9bXrtmD7YnE2yM9qH94PyIjia42iafMxFZf3VTh9jGA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.2 前置处理

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ZtDnyiaLMKkGd8l3A0mibvdBARndupXeoOh0UGOibIFfsPdLjab2WHCYw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

主要就是遍历切面，放入缓存。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4hDT04uib91GMPkN8cHKKMjKpRwdAFmicdjOqOn9eSxro3cWXTXqUPzAA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4TdlA7SmhXmPR6eYy682zGAxXHBbZOBbFUCAwdA7jDoVbSGuHItv6RA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4DPoLbtzjz2pNOKml0KLzhx2tsnLgeRbYYDg7e7F38FmD83FWed13lQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O48elQ9Nq51kCdgxuxXg76ohVia0xet3TeoLf7ADg8smRU9ibnURhtDdIg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4a8QGN7GczQYhrfaDbuRyUB8Wu3iaFN5SSNqrYZicC5MeFsdXKkweElLQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ay47vSwAnK9wD2iaaksicNnTL0iaT16DmXPbcYWUs29pXxyUtc5HPHQcw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**这里是重点！敲黑板！！！**

1. 我们会先遍历所有的类；
2. 判断是否切面，只有切面才会进入后面逻辑；
3. 获取每个 Aspect 的切面列表；
4. 保存 Aspect 的切面列表到缓存 advisorsCache 中。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4K7oH7ibhlaHwhicVMUPoeWT6Rt1Cg9LtdKUicqHyb87Cia3EDryWyf7PuA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

到这里，获取切面信息的流程就结束了，**因为后续对切面数据的获取，都是从缓存 advisorsCache 中拿到。**

下面就对上面的流程，再深入解读一下。

## 2.2.1 判断是否是切面

上图的第 2 步，逻辑如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4kErgO6AKZLDrfiaQzlSzvv60iaVUjomEAkhebqSStkNLP0eeyazicae7w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.2.2 获取切面列表

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4Yz2kLQqbVTretLNcjbauA92vrrdKrgyxiaOy95bLQV6vqDzKW0NkW7A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4YckpSOOkHT17du9PBVjH9sJAZKVrAoJEZP8nx9dJVn6CpIn4I9GSuw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4xCXRZ2IZEJttO4MwAZcG6wZtjQ4cH0y5DTcbIL23IcwZfCklbrccbQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O44LZhYfppHo0gxlFKGibDm9fTzkRVXRhvgmDvECQKVCypNcKhIa2rbng/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

进入到 getAdvice()，生成切面信息。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4l65bEpP1VuOfIXGOLGWuQbKH2aMacdViaHxqYedcybE9dicPVSTkrq6A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.3 后置处理

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4XuzY8g3afBQaxomKfMHZQibsM6p7pice2g0QVIErdIRZZLZNafvuyfNw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

主要就是从缓存拿切面，和 louzai 的方法匹配，并创建 AOP 代理对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4UicMhNNVPNyowO2OEiclkxCB3vBicFMrVlFvUcd1W9gazg7iblEArpK4Zg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

进入 doCreateBean()，走下面逻辑。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ulBlkcNy9wFJeKmReRa5Ph7a1QDiaSMm2mDvPWHKZhTbOicaSLkSCd4w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O42fZuuCBVWyfm01ibGvFuHyvc94PdToq1II39kSCYYMUCw3c3yVG9P4w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4RLDUUt340p7zjEhGaGXbYuGWDbM52b6z1SwMAUy4kLqtPZBOcO6PcA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4hOmhWryibALt4G2ryDBPJL8EXzC6Q6Ro9egibibqhaE76F62UpicOQteQQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**这里是重点！敲黑板！！！**

1. 先获取 louzai 类的所有切面列表；
2. 创建一个 AOP 的代理对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4LVZ3qoDZcgicthAAN7ibXAqVAsiaygb5HXGicAmdJXuI17icorw9QEVGBIA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.3.1 获取切面

我们先进入第一步，看是如何获取 louzai 的切面列表。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4lPUHFvm7JH6R8icJPMr9mc4YP7RdLhL677EAstkpPjUTpb7YMbkhS8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4cSokxHgHXQ1TJicvibhZzJM0fz6Fgr45cunOOibrOSt3efTOn9s1vcRUw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



进入 buildAspectJAdvisors()，这个方法应该有印象，就是前面将切面信息放入缓存 advisorsCache 中，现在这里就是要获取缓存。



![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4iaP8LiaES89IKH48znUibTqzXmn37cxrKZ3LpXWq34lva1ic3SpGgDfCLQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

再回到 findEligibleAdvisors()，从缓存拿到所有的切面信息后，继续往后执行。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4MzCRxv6JkjGw6er8IgaKOCzMFhX6gPasibibUUQwmzDHNBk5gGlGMLfw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O496zgQ74I0D6zJg5b4iaokkqPSo9yUABebhArpISVicBXTPznZXrHOJ5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4CzzaAibMia7YaibR2BSuYicJWsefLlbLZJwQTuabQyN5ed6iawZJicclR1ZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4almx0bav8iaZXpqbrGG970ubbUKmhEd5CO7gTvUo0da13MdIgd3wB0A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4V9yOGCic3zpjp0VQKrq4FJicNN0fP5SlZF82QhRfY0p1KMZkzhANR6fg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.3.2 创建代理对象

有了 louzai 的切面列表，后面就可以开始去创建 AOP 代理对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ricNbjfsgkLYRdWKkRTnzXmIiaVhUl6lnbUFSVpLSicwaeFyqXOS7CNRQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4Vuo8yqBia7O2YG8PFwRTypZRTe8mictfttuhMJEe9gdQmkovvMkMLj8Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4FsYTYmz8OUZTlWDcLKZ7hz1vIDibeUpLk6icvMiaKlZib0YicibmQT5g0F6g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**这里是重点！敲黑板！！！**

这里有 2 种创建 AOP 代理对象的方式，我们是选用 Cglib 来创建。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4A82kvMpF5ycg5X9Q9DlkJ4gs2BOX7bARALxSrkI6ts5d0YxlCiclbOw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4HzJ9OwclEZTxcAyBcLtxe7VSJ0gCujB3ibOKlnbaXALkkecXKAQ2M3Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4PrJQVgHH85M1YW4xBuut8GicoqRibx4NFm69a2N2ZSq9LqlU9SuagxyQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们再回到创建代理对象的入口，看看创建的代理对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ht7TRNptISLQrdLxTSapvbQ02lU8ll23hGykbtqZE6QlWCawOiaNicHg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.4 切面执行

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ENxMzOmow3mQMmFMVMgZVlCr4yO0vhuWJvY51vgBkQVoWM5y2q0Oug/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

通过 “责任链 + 递归”，执行切面和方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4N8AN0m0rC3e0OV5TPlM98plR3pxa9r8tqT4DBFA9HRg4bibmN51owiaQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4Oq78DNF9NvVehAEWibMZA7yjqSXYg5nibTAApy4hzxPkJxHsF1CEeP8g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**前方高能！这块逻辑非常复杂！！！**

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O44uib7OMyGFLXeUhjpFApSFP8aolicmd34vIWl5M5jktkuywXCPJO3byQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

下面就是“执行切面”最核心的逻辑，简单说一下设计思路：

1. **设计思路**：采用递归 + 责任链的模式；
2. **递归**：反复执行 CglibMethodInvocation 的 proceed()；
3. **退出递归条件**：interceptorsAndDynamicMethodMatchers 数组中的对象，全部执行完毕；
4. **责任链**：示例中的责任链，是个长度为 3 的数组，每次取其中一个数组对象，然后去执行对象的 invoke（）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O44PkbH5GVhmWSlvlfCibIWwe0mkx2IQEnjaT8YSichr8o39nLEDaw2ZSg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

因为我们数组里面只有 3 个对象，所以只会递归 3 次，下面就看这 3 次是如何递归，责任链是如何执行的，设计得很巧妙！

### 2.4.1 第一次递归

数组的第一个对象是 ExposeInvocationInterceptor，执行 invoke（），**注意入参是 CglibMethodInvocation。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O40vqktjfV0eLHvBibAHAJRzqzrmqls6qqcRKIIAEibSoYvPLI9zjntOgw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

里面啥都没干，继续执行 CglibMethodInvocation 的 process()。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4S8ppleupSJvPxQOCHKW33Ckby5EE1UcqVVLJ4EZcH1VUd540Ewzjqg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4eVWmmk106G9fxJz643b5RIrRz8I6aMyXQibuaFtgdgdWecPia3icp3FZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.4.2 第二次递归

数组的第二个对象是 MethodBeforeAdviceInterceptor，执行 invoke（）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ibGMCtZEYQIH8ibLNbvg9ya5cGM6l5GNAI6ZpVxrmBpPMzTaBpowCfbw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4moUtcp5epEKwaK28VSoIxOcyBSCN39ZsM4HKBdU6g0W3ZSzWNZYg7Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4dKUXR3LAmFbLbHyE6xBtbzyE6iciab4nXelVjxwtt447QHtHy7ibQ61Hg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4eegplO4nroExuPBB4LetGxlLicg8FqgVUUS1iacf6Dj40ibV6PWQa7NGQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4bNynXLO4EdZwGJbnZC4fomeWOj0zHnL6cNhrDDIokMoBuS0z87D3og/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4lg3L1nITelicVmSWPl2FOwFAReib9yicO02Iel242sFWzIcwPo66TALcQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4DWKn7VQStcNT30jtrfrHqvNjRy8gE7BzJ0wQlQjssavybNIB07v2og/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4QKfK5hmBkbtyfCqVfO2iciaUnf62tPU3Ms0XMkRgtOOcPJic1bwImHfMg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4yV9T0v095V42AwtANOPeDPR1JkmLZj2hnPBWfgJJh8TiadHiadHwopeg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.4.3 第三次递归

数组的第二个对象是 AfterReturningAdviceInterceptor，执行 invoke（）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O41jz0BX5ib7ecicyqGG9hYT19PmiagSKjQI3Tticaiad5ae8dla7O7nXZR8Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4VUT8tQ8caK2JjNEpIALSghswOicAJLspI2pQt71ibPajmNhemvSO9wDg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4eUvHFzQdyUUDpP5YyPhy66PriaOT1pQVDI9vh7l5JD3c9W7PZ1GibGhg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

执行完上面逻辑，就会退出递归，我们看看 invokeJoinpoint() 的执行逻辑，其实就是执行主方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4U03Pfm8PpegqoRBwX7oOJ3sLg9eJ3dxZXLpibhSUBtm8mwJLKXEJBlg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4yR2icqOPAGFkgSBqLicMFseLkC3Q5B0WInpBwLuOyxo9ulQSW8PWhXFg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4ictic7WNSnLib1SYXusGDaa1Fic8MrVtbcKVpria7RxxxtNspWJTCoSrkbA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

再回到第三次递归的入口，继续执行后面的切面。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4XL3EIGRLTuGEnc9KM0UNKXTNyv1icP9NaSWo4RF8katPic2c4Ju3SBJw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

切面执行逻辑，前面已经演示过，直接看执行方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4gYauvQyvOVjjXnjcxE77lS8PiaHZ5jE6IlUk3XfraFnq1QN6Zrwgp6Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

后面就依次退出递归，整个流程结束。

### 2.4.4 设计思路

这块代码，我研究了大半天，因为这个不是纯粹的责任链模式。

纯粹的责任链模式，对象内部有一个自身的 next 对象，执行完当前对象的方法末尾，就会启动 next 对象的执行，直到最后一个 next 对象执行完毕，或者中途因为某些条件中断执行，责任链才会退出。

这里 CglibMethodInvocation 对象内部没有 next 对象，全程是通过 interceptorsAndDynamicMethodMatchers 长度为 3 的数组控制，依次去执行数组中的对象，直到最后一个对象执行完毕，责任链才会退出。

**这个也属于责任链，只是实现方式不一样，后面会详细剖析**，下面再讨论一下，这些类之间的关系。

我们的主对象是 CglibMethodInvocation，继承于 ReflectiveMethodInvocation，然后 process() 的核心逻辑，其实都在 ReflectiveMethodInvocation 中。

**ReflectiveMethodInvocation 中的 process() 控制整个责任链的执行。**

ReflectiveMethodInvocation 中的 process() 方法，里面有个长度为 3 的数组 interceptorsAndDynamicMethodMatchers，里面存储了 3 个对象，分别为 ExposeInvocationInterceptor、MethodBeforeAdviceInterceptor、AfterReturningAdviceInterceptor。

**注意！！！这 3 个对象，都是继承 MethodInterceptor 接口。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/fEsWkVrSk57LEFac78iab97vUUjticT7O4MrfMfMOZ1NsWCVegYAVHW5MH3Dib33kL6FXfrXuP3JOnqk2ibxhxnrRA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

然后每次执行 invoke() 时，里面都会去执行 CglibMethodInvocation 的 process()。

**是不是听得有些蒙圈？甭着急，我重新再帮你梳理一下。**

对象和方法的关系：

- **接口继承**：数组中的 3 个对象，都是继承 MethodInterceptor 接口，实现里面的 invoke() 方法；
- **类继承**：我们的主对象 CglibMethodInvocation，继承于 ReflectiveMethodInvocation，复用它的 process() 方法；
- **两者结合（策略模式）**：invoke() 的入参，就是 CglibMethodInvocation，执行 invoke() 时，内部会执行 CglibMethodInvocation.process()，这个其实就是个策略模式。

> 可能有同学会说，invoke() 的入参是 MethodInvocation，没错！但是 CglibMethodInvocation 也继承了 MethodInvocation，不信自己可以去看。

执行逻辑：

- **程序入口**：是 CglibMethodInvocation 的 process() 方法；
- **链式执行（衍生的责任链模式）**：process() 中有个包含 3 个对象的数组，依次去执行每个对象的 invoke() 方法。
- **递归（逻辑回退）**：invoke() 方法会执行切面逻辑，同时也会执行 CglibMethodInvocation 的 process() 方法，让逻辑再一次进入 process()。
- **递归退出**：当数字中的 3 个对象全部执行完毕，流程结束。

所以这里设计巧妙的地方，是因为纯粹责任链模式，里面的 next 对象，需要保证里面的对象类型完全相同。

但是数组里面的 3 个对象，里面没有 next 成员对象，所以不能直接用责任链模式，那怎么办呢？就单独搞了一个 CglibMethodInvocation.process()，通过去无限递归 process()，来实现这个责任链的逻辑。

**这就是我们为什么要看源码，学习里面优秀的设计思路！**

# 3. 总结

我们再小节一下，文章先介绍了什么是 AOP，以及 AOP 的原理和示例。

之后再剖析了 AOP 的源码，分为 3 块：

- 将所有的切面都保存在缓存中；
- 取出缓存中的切面列表，和 louzai 对象的所有方法匹配，拿到属于 louzai 的切面列表；
- 创建 AOP 代理对象；
- 通过“责任链 + 递归”，去执行切面逻辑。