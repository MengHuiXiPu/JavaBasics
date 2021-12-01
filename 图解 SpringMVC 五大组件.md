# 图解 SpringMVC 五大组件

SpringMVC最重要的就是五大组件

- DispatcherServlet
- HandleMapping
- Controller
- ModeAndView
- ViewResolver

下面一一介绍这五大控件

**1.DispatcherServlet**

这个控件是SpringMVC 最核心的一个控件，顾名思义其实他就是一个Servlet，是Spring写好的一个Servlet

**2.HandleMapping**

控件标明了路径与Controller的对应关系，不同的路径访问不同的Controller

**3. Controller**

用来处理业务逻辑的Java类

**4. ModeAndView**

Mode用来绑定处理后所得的数据，View视图名

**5. ViewResolver**

视图解析器明确了视图名与视图对象的关系，是调用demo.jsp还是调用demo.html,以及明确视图的位置

**五大组件的关系**

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudqhv6fzgUCwFIqhVAUr7elYm5TjaZpico5BErmaGdCc8rYDDWgwkF8Bo5EyITwpeo2IsibcWgTE3Bg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)点击放大

**五大组件的位置关系**

DispatcherServlet属于servlet所以位于Tomcat等服务器容器中，而、HandleMapping ViewResolver 属于Spring所以位于SpringMVC配置文件中，Contrlloer以及ModeView位于src文件中处理具体逻辑业务

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudqhv6fzgUCwFIqhVAUr7elqob3yqG69X9JnBw6wzBAQvpPsQUFfXC12YRZWDGFgKw6T9ljO5ZArw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)点击放大

下面说说五大组件的使用以及配置

##### 配置DispatcherServlet

DispatcherServlet属于Servlet所以配置在web.xml文件中。init-param为该Servlet启动所需参数。DispatcherServlet会读取初始化contextConfigLocation参数里面的值从而获取spring的配置位置，然后自启动容器

```
<!-- 配置前端控制器，配置Servlet -->
<servlet>
     <servlet-name>springMvc</servlet-name>
     <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
     <init-param>
          <param-name>contextConfigLocation</param-name>
           <param-value>classpath:springmvc.xml</param-value>
      </init-param>
      <load-on-startup>1</load-on-startup>
</servlet>
<!--配置请求路径-->
<servlet-mapping>
    <servlet-name>springMvc</servlet-name>
    <url-pattern>*.form</url-pattern>
</servlet-mapping>
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudqhv6fzgUCwFIqhVAUr7elt9eibQ8KslESzmGg1ESXKcpJ8SpmLYyVfhjMqNfrmQicabD7PUpRWknw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)点击放大

##### 2. HandleMapping

mvc:annotation-driven 配置mvc注解扫描 可以用注解@RequestMapping(“url”)加在方法上简化配置prop标明路径和XXController的关系

```
<!--开启mvc注解扫描-->
<mvc:annotation-driven/>
<!--创建Controller  bean-->
<bean id="loginController" class="包名+类名"/>

<bean class="org.springframework.web.servlrt.handler.SimpleUrlHandlerMapping">
    <property name="mappings">
        <props>
            <prop key="/login.form">loginController</prop>
        </props>
    </property>
</bean>
```

##### 3.Controller

处理getData.form该路径的业务逻辑

```
@Controller
public class DataController {
    @RequestMapping("getData.form")
    public ModeAndView hello(String stationId) {
        System.out.println("hello");
        return new ModeAndView("hello")
    }
}
```

##### 4. ModeAndView

两种ModeAndView的构造方法，第一个视图名，第二个需要绑定的数据

```
ModeAndView(String viewName)
ModeAndView(String viewName ,Map data)
```

##### 5. ViewResolver

前缀+视图名+后缀=映射到页面

```
<!-- 配置视图解析器 -->
<bean class="org.springframework.web.servlet.view.InternalResour    ceViewResolver">
    <property name="prefix" value="/WEB-INF/"/>
    <property name="suffix" value=".html"></property>
</bean>
```

 