# 三种跨域解决方案：HttpClient、注解、网关

- 注解：@CrossOrigin
- 网关整合
- Httpclient

------

##### 为什么会有跨域问题

因为浏览器的同源政策，就会产生跨域。比如说发送的异步请求是不同的两个源，就比如是不同的的两个端口或者不同的两个协议或者不同的域名。由于浏览器为了安全考虑，就会产生一个同源政策，不是同一个地方出来的是不允许进行交互的。

##### 常见的跨域解决方式

1. 在控制层加入允许跨域的注解 `@CrossOrigin`
2. 使用`httpclient`，不依赖浏览器
3. 使用网关 `Gateway`

## ***\*注解：@CrossOrigin\****

在控制层加入允许跨域的注解，即可完成一个项目中前后端口跨域的问题

## ***\*网关整合\****

`Spring Cloud Gateway`作为Spring Cloud生态系统中的网关，目标是替代`Netflix Zuul`，其 不仅提供统一的路由方式，并且还基于Filer链的方式提供了网关基本的功能，例如：安全、监 控/埋点、限流等。

**（1）路由**

路由是网关最基础的部分，路由信息有一个ID、一个目的URL、一组断言和一组 Filter组成。如果断言路由为真，则说明请求的URL和配置匹配

**（2）断言**

Java8中的断言函数。`Spring Cloud Gateway`中的断言函数输入类型是Spring5.0框 架中的`ServerWebExchange`。`Spring Cloud Gateway`中的断言函数允许开发者去定义匹配来自 于`http request`中的任何信息，比如请求头和参数等。

**（3）过滤器**

一个标准的`Spring webFilter`。`Spring cloud gateway`中的filter分为两种类型的 Filter，分别是`Gateway Filter`和`Global Filter`。过滤器Filter将会对请求和响应进行修改处理

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq4BOKNxic7d1b3fXcrqh2xDxzurBT9MTpyn29hXibDXppb6YibrpRUSdwAk1CD1exI5noqNJNYL6EExA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图片

`Spring cloud Gateway`发出请求。然后再由`Gateway Handler Mapping`中找到与请 求相匹配的路由，将其发送到`Gateway web handler`。Handler再通过指定的过滤器链将请求发 送到实际的服务执行业务逻辑，然后返回。

##### 项目中使用

新建模块`service_gateway`

```
<dependencies>
<!-- 公共模块依赖 -->
    <dependency>
        <groupId>com.lzq</groupId>
        <artifactId>service_utils</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <!-- 服务注册 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

配置文件

```
#服务端口
server.port=9090

# 服务名
spring.application.name=service-gateway
# nacos服务地址 默认8848
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8888
#使用服务发现路由
spring.cloud.gateway.discovery.locator.enabled=true
#设置路由id
spring.cloud.gateway.routes[0].id=service-hosp
#设置路由的uri  lb负载均衡
spring.cloud.gateway.routes[0].uri=lb://service-hosp
#设置路由断言,代理servicerId为auth-service的/auth/路径
spring.cloud.gateway.routes[0].predicates= Path=/*/hosp/**
 
#设置路由id
spring.cloud.gateway.routes[1].id=service-cmn
#设置路由的uri
spring.cloud.gateway.routes[1].uri=lb://service-cmn
#设置路由断言,代理servicerId为auth-service的/auth/路径
spring.cloud.gateway.routes[1].predicates= Path=/*/cmn/**
#设置路由id
spring.cloud.gateway.routes[2].id=service-hosp
#设置路由的uri
spring.cloud.gateway.routes[2].uri=lb://service-hosp
#设置路由断言,代理servicerId为auth-service的/auth/路径
spring.cloud.gateway.routes[2].predicates= Path=/*/userlogin/**
```

创建启动类

```
@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

修改前端`.evn`文件，改成访问网关端口号

做集群部署时，他会根据名称实现负载均衡

跨域理解：发送请求后，网关过滤器会进行请求拦截，将跨域放行，转发到服务器中

跨域配置类

```
@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");
 
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource(new PathPatternParser());
        source.registerCorsConfiguration("/**", config);
 
        return new CorsWebFilter(source);
    }
}
```

若之前采用注解跨域,需要将`@CrossOrigin`去掉

## ***\*Httpclient\****

常见的使用场景：多系统之间接口的交互、爬虫

http原生请求，获取百度首页代码

```
public class HttpTest {
    @Test
    public void test1() throws Exception {
     String url = "https://www.badu.com";
        URL url1 = new URL(url);
        //url连接
        URLConnection urlConnection = url1.openConnection();
        HttpURLConnection httpURLConnection = (HttpURLConnection)urlConnection;
        //获取httpURLConnection输入流
        InputStream is = httpURLConnection.getInputStream();
        //转换为字符串
        InputStreamReader reader = new InputStreamReader(is, StandardCharsets.UTF_8);
        BufferedReader br = new BufferedReader(reader);
        String line;
        //将字符串一行一行读取出来
        while ((line = br.readLine())!= null){
            System.out.println(line);
        }
    }
}
//设置请求类型
httpURLConnection.setRequestMethod("GET");
//请求包含 请求行、空格、请求头、请求体
//设置请求头编码
httpURLConnection.setRequestProperty("Accept-Charset","utf-8");
```

使用`HttpClient`发送请求、接收响应

1. 创建`HttpClient`对象。
2. 创建请求方法的实例，并指定请求URL。如果需要发送GET请求，创建`HttpGet`对象；如果需要发送POST请求，创建`HttpPost`对象。
3. 如果需要发送请求参数，可调用`HttpGet`、`HttpPost`共同的`setParams(HetpParams params)`方法来添加请求参数；对于`HttpPost`对象而言，也可调用`setEntity(HttpEntity entity)`方法来设置请求参数。
4. 调用`HttpClient`对象的`execute(HttpUriRequest request)`发送请求，该方法返回一个`HttpResponse`。
5. 调用`HttpResponse`的`getAllHeaders()`、`getHeaders(String name)`等方法可获取服务器的响应头；调用`HttpResponse`的`getEntity()`方法可获取`HttpEntity`对象，该对象包装了服务器的响应内容。程序可通过该对象获取服务器的响应内容。
6. 释放连接。无论执行方法是否成功，都必须释放连接

集成测试，添加依赖

```
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.13</version>
</dependency>
@Test
public void test2(){
    //可关闭的httpclient客户端，相当于打开一个浏览器
    CloseableHttpClient client = HttpClients.createDefault();
    String url = "https://www.baidu.com";
    //构造httpGet请求对象
    HttpGet httpGet = new HttpGet(url);
    //响应
    CloseableHttpResponse response = null;
    try {
        response = client.execute(httpGet);
        // 获取内容
        String result = EntityUtils.toString(response.getEntity(), "utf-8");
        System.out.println(result);
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        //关闭流
        if (client != null){
            try {
                client.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

项目中使用，系统调用平台接口保存信息，根据传入josn数据保存信息

系统中

```
@RequestMapping(value="/hospital/save",method=RequestMethod.POST)
public String saveHospital(String data, HttpServletRequest request) {
 try {
  apiService.saveHospital(data);
 } catch (YyghException e) {
  return this.failurePage(e.getMessage(),request);
 } catch (Exception e) {
  return this.failurePage("数据异常",request);
 }
 return this.successPage(null,request);
}
```

`saveHospital`方法

```
@Override
public boolean saveHospital(String data) {
    JSONObject jsonObject = JSONObject.parseObject(data);
    Map<String, Object> paramMap = new HashMap<>();
    paramMap.put("hoscode","10000");
    paramMap.put("hosname",jsonObject.getString("hosname"))
    //图片
    paramMap.put("logoData", jsonObject.getString("logoData"));
    //  http://localhost:8201/api/hosp/saveHospital
    //httpclient
    JSONObject respone =
            HttpRequestHelper.sendRequest(paramMap,this.getApiUrl()+"/api/hosp/saveHospital");
    System.out.println(respone.toJSONString());

    if(null != respone && 200 == respone.getIntValue("code")) {
        return true;
    } else {
        throw new YyghException(respone.getString("message"), 201);
    }
}
```

`HttpRequestHelper`工具类

```
/**
 * 封装同步请求
 * @param paramMap
 * @param url
 * @return
 */
public static JSONObject sendRequest(Map<String, Object> paramMap, String url){
    String result = "";
    try {
        //封装post参数
        StringBuilder postdata = new StringBuilder();
        for (Map.Entry<String, Object> param : paramMap.entrySet()) {
            postdata.append(param.getKey()).append("=")
                    .append(param.getValue()).append("&");
        }
        log.info(String.format("--> 发送请求：post data %1s", postdata));
        byte[] reqData = postdata.toString().getBytes("utf-8");
        byte[] respdata = HttpUtil.doPost(url,reqData);
        result = new String(respdata);
        log.info(String.format("--> 应答结果：result data %1s", result));
    } catch (Exception ex) {
        ex.printStackTrace();
    }
    return JSONObject.parseObject(result);
}
```

`HttpUtil`工具类

```
public static byte[] send(String strUrl, String reqmethod, byte[] reqData) {
  try {
   URL url = new URL(strUrl);
   HttpURLConnection httpcon = (HttpURLConnection) url.openConnection();
   httpcon.setDoOutput(true);
   httpcon.setDoInput(true);
   httpcon.setUseCaches(false);
   httpcon.setInstanceFollowRedirects(true);
   httpcon.setConnectTimeout(CONN_TIMEOUT);
   httpcon.setReadTimeout(READ_TIMEOUT);
   httpcon.setRequestMethod(reqmethod);
   httpcon.connect();
   if (reqmethod.equalsIgnoreCase(POST)) {
    OutputStream os = httpcon.getOutputStream();
    os.write(reqData);
    os.flush();
    os.close();
   }
   BufferedReader in = new BufferedReader(new InputStreamReader(httpcon.getInputStream(),"utf-8"));
   String inputLine;
   StringBuilder bankXmlBuffer = new StringBuilder();
   while ((inputLine = in.readLine()) != null) {  
       bankXmlBuffer.append(inputLine);  
   }  
   in.close();  
   httpcon.disconnect();
   return bankXmlBuffer.toString().getBytes();
  } catch (Exception ex) {
   log.error(ex.toString(), ex);
   return null;
  }
 }
```

对应平台接口

```
@RestController
@RequestMapping("/api/hosp")
public class ApiController {
    @Autowired
    private HospitalService hospitalService;
    @ApiOperation(value = "上传医院")
    @PostMapping("saveHospital")
    public R saveHospital(HttpServletRequest request) {
        //通过request取到前端接口传过来的值
        Map<String, String[]> parameterMap = request.getParameterMap();
        //将数组值转换成一个值
        Map<String, Object> paramMap = HttpRequestHelper.switchMap(parameterMap);
        //将map集合转成josn字符串
        String mapStr = JSONObject.toJSONString(paramMap);
        //josn字符串转成对象
        Hospital hospital = JSONObject.parseObject(mapStr, Hospital.class);
        //加入MongoDB中
        hospitalService.saveHosp(hospital);
        return R.ok();
    }
}
```

即可完成不同系统中的相互调用

