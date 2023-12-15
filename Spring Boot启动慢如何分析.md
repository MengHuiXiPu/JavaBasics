# Spring Boot启动慢如何分析

# 分析方法

## 自定义监听器

SpringApplicationRunListener是Spring Boot中的一个接口，它的作用是在SpringApplication运行的各个阶段提供回调接口，以便我们可以在这些阶段执行自定义的逻辑。SpringApplicationRunListener接口定义了以下几个方法：

1. **starting：** 在SpringApplication运行开始时调用。
2. **environmentPrepared：** 在Environment准备好，但在ApplicationContext创建之前调用。
3. **contextPrepared：** 在ApplicationContext准备好，但在其加载源之前调用。
4. **contextLoaded：** 在ApplicationContext加载源后调用。
5. **started：** 在ApplicationContext刷新并启动后，但在SpringApplication.run方法完成之前调用。
6. **running：** 在SpringApplication.run方法完成后调用。
7. **failed：** 在SpringApplication运行失败时调用。

通过实现SpringApplicationRunListener接口，我们可以在SpringApplication运行的各个阶段执行自定义的逻辑，例如初始化资源、清理资源、记录日志等。下面的代码例子展示继承SpringApplicationRunListener接口并打印相应日志

```
@Slf4j
public class MySpringApplicationRunListener implements SpringApplicationRunListener {
    // 这个构造函数不能少，否则反射生成实例会报错
    public MySpringApplicationRunListener(SpringApplication sa, String[] args) {
    }
    @Override
    public void starting() {
        log.info("starting {}", LocalDateTime.now());
    }
    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        log.info("environmentPrepared {}", LocalDateTime.now());
    }
    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        log.info("contextPrepared {}", LocalDateTime.now());
    }
    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        log.info("contextLoaded {}", LocalDateTime.now());
    }
    @Override
    public void started(ConfigurableApplicationContext context) {
        log.info("started {}", LocalDateTime.now());
    }
    @Override
    public void running(ConfigurableApplicationContext context) {
        log.info("running {}", LocalDateTime.now());
    }
    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        log.info("failed {}", LocalDateTime.now());
    }
}
```

对于每个Bean初始化耗时的监控，我们可以通过继承BeanPostProcessor接口方式，BeanPostProcessor 是Spring框架的一个接口，它定义了两个回调方法：postProcessBeforeInitialization和postProcessAfterInitialization。这两个方法分别在Bean初始化前后被调用。你可以在这两个方法中记录时间，然后计算出Bean初始化的耗时。

```
@Component
public class BeanInitiateTimeCost implements BeanPostProcessor {
    private Map<String, Long> costMap = Maps.newConcurrentMap();
  
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        costMap.put(beanName, System.currentTimeMillis());
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (costMap.containsKey(beanName)) {
            Long start = costMap.get(beanName);
            long cost  = System.currentTimeMillis() - start;
            if (cost > 0) {
                costMap.put(beanName, cost);
                System.out.println("bean: " + beanName + "\ttime: " + cost);
            }
        }
        return bean;
    }
}
```

## 通过Actuator实现

**前提条件：** Spring Boot启动跟踪需要Spring boot的版本在2.4及以上。相关步骤

1. 添加相关依赖

引入spring-boot-starter-actuator

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2.暴露startup路径

将下面配置加入到*application.properties*， 从而暴露startup链接

> management.endpoints.web.exposure.include=startup

1. 使用BufferingApplicationStartup的方式启动程序

在使用Spring Actuator分析Spring Boot启动慢的问题时，你需要在程序启动时启用BufferingApplicationStartup。这是因为Spring Boot 2.4.0及以上版本的Actuator的/actuator/startup端点使用BufferingApplicationStartup来收集启动过程的信息。

```
public class Application {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(Application.class);
        application.setApplicationStartup(new BufferingApplicationStartup(2048));
        application.run(args);
    }
}
```

在这个例子中，我们创建了一个BufferingApplicationStartup实例，并设置了缓冲区大小为2048。然后我们将这个BufferingApplicationStartup实例设置为SpringApplication的ApplicationStartup。

这样，当你启动应用程序并访问/actuator/startup端点时，你就可以看到启动过程的详细信息，包括每个bean的启动时间。你可以根据这些信息找出启动时间较长的bean，并进行优化。

## 返回结果分析

`curl 'http://localhost:8080/actuator/startup'` 返回JSON格式的数据，如下

```
{
    "springBootVersion": "2.6.3",
    "timeline": {
        "startTime": 1698287960.140000000,
        "events": [
            {
                "endTime": 1698287960.176000000,
                "duration": 0.018000000,
                "startupStep": {
                    "name": "spring.boot.application.starting",
                    "id": 0,
                    "tags": [
                        {
                            "key": "mainApplicationClass",
                            "value": "com.yuehuijiankang.blood.pressure.BloodPressureServiceApplication"
                        }
                    ],
                    "parentId": null
                },
                "startTime": 1698287960.158000000
            },
   ........         
```

**startupStep**中可以查看具体的执行步骤和耗时情况。借助jq工具可以方便分析JSON数据。比如，如果我们想查看Bean初始化最慢的前10个bean。我们可以执行下面命令：

```
curl 'http://localhost:8080/actuator/startup' -X POST | jq '[.timeline.events | sort_by(.duration) | reverse[] | select(.startupStep.name | match("spring.beans.instantiate")) | {beanName: .startupStep.tags[0].value, duration: .duration}] | .[:10]'
```

# 

返回结果如下：![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/o5KQuEJxTEE7Q4aiagApQiafuoOOwnGfF8gumvtzJzF2lx3f828usib6JWXk6ap8jYnkkjOzjVv4KpNGia18SuvsbA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



# 总结

如果你发现Spring Boot项目启动慢，可以通过以下两种方法进行分析：

1. 自定义监听器：通过实现SpringApplicationRunListener接口，可以在SpringApplication运行的各个阶段执行自定义的逻辑，例如记录每个阶段的时间。此外，可以通过实现BeanPostProcessor接口，记录Bean初始化前后的时间，从而计算出Bean初始化的耗时。
2. 使用Actuator：Spring Boot 2.4及以上版本的Actuator提供了/actuator/startup端点，可以收集启动过程的信息。首先，需要在pom.xml文件中添加spring-boot-starter-actuator依赖，并在application.properties文件中暴露startup端点。然后，需要在程序启动时启用BufferingApplicationStartup。最后，通过访问/actuator/startup端点，可以看到启动过程的详细信息，包括每个bean的启动时间。

通过以上两种方法，可以找出启动时间较长的阶段或Bean，并进行相应的优化。

 