# Spring 的 @Autowire 和 @Resource 注解使用方式推荐

Idea写代码的时候，看到之前的项目中显示有warning的提示，去看了下，是如下代码?

```
@Autowire
private JdbcTemplate jdbcTemplate;
```

提示的警告信息

## “Field injection is not recommended Inspection info: Spring Team recommends: "Always use constructor based dependency injection in your beans. Always use assertions for mandatory dependencies".

这段是Spring工作组的建议，大致翻译一下：

属性字段注入的方式不推荐，检查到的问题是：Spring团队建议:"始终在bean中使用基于构造函数的依赖项注入，始终对强制性依赖项使用断言"

如图

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLbk2CHfLwx8jJkeLvAeIDXYVIwH1JKUbeNhMCRSmzaOwQjbYL62hBibgYjQicn3Xl4icynl5vGDLgl6Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



注入方式





虽然当前有关Spring Framework（5.0.3）的文档仅定义了两种主要的注入类型，但实际上有三种:

**基于构造函数的依赖注入**

```
public class UserServiceImpl implents UserService{
    private UserDao userDao;

    @Autowire
    public UserServiceImpl(UserDao userDao){
        this.userDao = userDao;
    }
}
```

**基于Setter的依赖注入**

```
public class UserServiceImpl implents UserService{
     private UserDao userDao;

     @Autowire
     public serUserDao(UserDao userDao){
         this.userDao = userDao;
     }
 }
```

**基于字段的依赖注入**

```
public class UserServiceImpl implents UserService{
     @Autowire
     private UserDao userDao;
 }
```

基于字段的依赖注入方式会在Idea当中吃到黄牌警告，但是这种使用方式使用的也最广泛，因为简洁方便.您甚至可以在一些Spring指南中看到这种注入方法，尽管在文档中不建议这样做.(有点执法犯法的感觉)。

**如图**

![图片](https://mmbiz.qpic.cn/mmbiz/1J6IbIcPCLbk2CHfLwx8jJkeLvAeIDXYFibG2CTpoRrjz4EkE1m5df6kntmMWI5wsSf9JXaRTr555iccjKTShTBQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



基于字段的依赖注入缺点





## **| 对于有final修饰的变量不好使** 

Spring的IOC对待属性的注入使用的是set形式，但是final类型的变量在调用class的构造函数的这个过程当中就得初始化完成，这个是基于字段的依赖注入 做不到的地方．只能使用**基于构造函数的依赖注入**的方式

**| 掩盖单一职责的设计思想**

我们都知道在OOP的设计当中有一个**单一职责思想**，如果你采用的是**基于构造函数的依赖注入**的方式来使用Spring的IOC的时候，当你注入的太多的时候，这个构造方法的参数就会很庞大，类似于下面。

当你看到这个类的构造方法那么多参数的时候，你自然而然的会想一下:**这个类是不是违反了单一职责思想****?** .但是使用**基于字段的依赖注入**不会让你察觉，你会很沉浸在@Autowire当中。

```
public class VerifyServiceImpl implents VerifyService{

  private AccountService accountService;
  private UserService userService;
  private IDService idService;
  private RoleService roleService;
  private PermissionService permissionService;
  private EnterpriseService enterpriseService;
  private EmployeeService employService;
  private TaskService taskService;
  private RedisService redisService;
  private MQService mqService;

  public SystemLogDto(AccountService accountService, 
                      UserService userService, 
                      IDService idService, 
                      RoleService roleService, 
                      PermissionService permissionService, 
                      EnterpriseService enterpriseService, 
                      EmployeeService employService, 
                      TaskService taskService, 
                      RedisService redisService, 
                      MQService mqService) {
      this.accountService = accountService;
      this.userService = userService;
      this.idService = idService;
      this.roleService = roleService;
      this.permissionService = permissionService;
      this.enterpriseService = enterpriseService;
      this.employService = employService;
      this.taskService = taskService;
      this.redisService = redisService;
      this.mqService = mqService;
  }
}
```

**| 与Spring的IOC机制紧密耦合**

当你使用**基于字段的依赖注入**方式的时候，确实可以省略构造方法和setter这些个模板类型的方法，但是，你把控制权全给Spring的IOC了，别的类想重新设置下你的某个注入属性，没法处理(当然反射可以做到)。

本身Spring的目的就是解藕和依赖反转，结果通过再次与类注入器（在本例中为Spring）耦合，失去了通过自动装配类字段而实现的对类的解耦，从而使类在Spring容器之外无效。

**| 隐藏依赖性**

当你使用Spring的IOC的时候，被注入的类应当使用一些public类型(构造方法，和setter类型方法)的方法来向外界表达:我需要什么依赖.但是**基于字段的依赖注入**的方式，基本都是private形式的，private把属性都给封印到class当中了。

**| 无法对注入的属性进行安检**

**基于字段的依赖注入**方式，你在程序启动的时候无法拿到这个类，只有在真正的业务使用的时候才会拿到，一般情况下，这个注入的都是非null的，万一要是null怎么办，在业务处理的时候错误才爆出来，时间有点晚了，如果在启动的时候就暴露出来，那么bug就可以很快得到修复(当然你可以加注解校验)。

如果你想在属性注入的时候，想根据这个注入的对象操作点东西，你无法办到．我碰到过的例子：一些配置信息啊，有些人总是会配错误，等到了自己测试业务阶段才知道配错了，例如线程初始个数不小心配置成了3000，机器真的是狂叫啊!这个时候就需要再某些Value注入的时候做一个检测机制。

