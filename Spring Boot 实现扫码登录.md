# Spring Boot 实现扫码登录

### 一、首先咱们需要一张表

这表是干啥的呢？就是记录一下谁扫码了。谁登录了。

> User_Token表

字段如下：

- uuid : 用于确保唯一性
- userId ：谁登录的
- loginTime ：登录时间
- createTime ：创建时间 用于判断是否过期
- state：是否二维码失效 0有效 1失效

### 二、角色都有哪些

咱们还需要分析一下子。扫码登录这个业务逻辑都有哪些角色

- android端 or 微信Web端 ：扫码
- PC端 ：被扫。登录
- 服务端：掌控全局，提供接口。

### 三、接口都需要哪些？

有了角色。你用大腿也能想出来接口了对不对！！

所以咱们的接口有2个！

- 生成二维码接口：生成一个二维码。二维码中有UUID。
- 确认身份接口：确定身份以及判断是否二维码过期等

### 四、步骤

那句话怎么说的来着。要把大象装冰箱一共分几步？

- PC端打开。调用生成二维码接口 并与 服务端建立链接。链接使用uuid进行绑定
- 微信Web端进行扫码。获取二维码中的uuid。
- 微信Web端拿到uuid以后。显示是否登录页面。点击确定后 调用 确认身份接口。
- 确认身份接口通过以后。服务端给PC端发送信息。完成登录。此时链接断开。
- 如果您正在学习Spring Boot，推荐一个连载多年还在继续更新的免费教程：http://blog.didispace.com/spring-boot-learning-2x/

好了！分析完了这些。你们一定在想。。还有完没完啊。。不要在BB了。。赶紧贴代码吧。。

> 作者：观众老爷们。我这是在教给你们如何思考的方法呀?

那么开始贴代码吧！希望大家在看到的同时也可以自己进行思考。

### 五、疯狂贴代码

首先需要获取二维码的代码对不对！贴！

```
//获取登录二维码、放入Token
@RequestMapping(value = "/getLoginQr" ,method = RequestMethod.GET)
public void createCodeImg(HttpServletRequest request, HttpServletResponse response){
    response.setHeader("Pragma", "No-cache");
    response.setHeader("Cache-Control", "no-cache");

    response.setDateHeader("Expires", 0);
    response.setContentType("image/jpeg");

    try {
        //这里没啥操作 就是生成一个UUID插入 数据库的表里
        String uuid = userService.createQrImg();
        response.setHeader("uuid", uuid);
        // 这里是开源工具类 hutool里的QrCodeUtil 
        // 网址：http://hutool.mydoc.io/
        QrCodeUtil.generate(uuid, 300, 300, "jpg",response.getOutputStream());
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

有了获取二维码的接口。相对的前端需要调用。

> 知识点：动态加载图片流并取出header中的参数

这里使用了xmlhttp进行处理。

为什么？

因为后端返回的是一个流。

那么流中。就是放置了二维码中的uuid。这个uuid作为一次会话的标识符使用。

那么前端也需要拿到。跟后端进行webSocket链接。

这样有人扫码后。服务端才可以使用webSocket的方式通知前端。有人扫码成功了。你做你的业务吧。酱紫。

所以为了拿到请求中 header中放置的uuid 所以这样通过xmlhttp进行处理

```
<div class="qrCodeImg-box" id="qrImgDiv"></div>
```

js

```
$(document).ready(function(){
    initQrImg();
});
 
 
 function initQrImg(){
    $("#qrImgDiv").empty();

    var xmlhttp;
    xmlhttp=new XMLHttpRequest();
    xmlhttp.open("GET",getQrPath,true);
    xmlhttp.responseType = "blob";
    xmlhttp.onload = function(){
        console.log(this);
        uuid = this.getResponseHeader("uuid");

        if (this.status == 200) {
            var blob = this.response;
            var img = document.createElement("img");
            img.className = 'qrCodeBox-img';
            img.onload = function(e) {
                window.URL.revokeObjectURL(img.src);
            };
            img.src = window.URL.createObjectURL(blob);
            document.getElementById("qrImgDiv").appendChild(img);

            initWebSocket();
        }
    }
    xmlhttp.send();
}



var path = "://localhost:8085";
var getQrPath =  "http" + path + "/user/getLoginQr";
var wsPath =     "ws" + path + "/websocket/";



function initWebSocket(){

   if(typeof(WebSocket) == "undefined") {
       console.log("您的浏览器不支持WebSocket");
   }else{
       console.log("您的浏览器支持WebSocket");
       //实现化WebSocket对象，指定要连接的服务器地址与端口  建立连接
       //等同于socket = new WebSocket("ws://localhost:8083/checkcentersys/websocket/20");
       var wsPathStr = wsPath+uuid;
       socket = new WebSocket(wsPathStr);
       //打开事件
       socket.onopen = function() {
           console.log("Socket 已打开");
           //socket.send("这是来自客户端的消息" + location.href + new Date());
       };
       //获得消息事件
       socket.onmessage = function(msg) {
           console.log(msg.data);
           var data = JSON.parse(msg.data);
           if(data.code == 200){
               alert("登录成功！");
               //这里存放自己业务需要的数据。怎么放自己看
               window.sessionStorage.uuid = uuid;
               window.sessionStorage.userId = data.userId;
               window.sessionStorage.projId = data.projId;

               window.location.href = "pages/upload.html"
           }else{
               //如果过期了，关闭连接、重置连接、刷新二维码
               socket.close();
               initQrImg();
           }
           //发现消息进入    开始处理前端触发逻辑
       };
       //关闭事件
       socket.onclose = function() {
           console.log("Socket已关闭");
       };
       //发生了错误事件
       socket.onerror = function() {
           alert("Socket发生了错误");
           //此时可以尝试刷新页面
       }
   }

}
```

好了。上面已经提到了前端如何配置webSocket。

- 如果您正在学习Spring Boot，推荐一个连载多年还在继续更新的免费教程：http://blog.didispace.com/spring-boot-learning-2x/

### Spring Boot中操作WebSocket

1、增加pom.xml

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

2、增加一个Bean

```
/**
 * WebSocket的支持
 * @return
 */
@Bean
public ServerEndpointExporter serverEndpointExporter() {
    return new ServerEndpointExporter();
}
```

3、定义WebSocketServer

```
package com.stylefeng.guns.rest.modular.inve.websocket;
 
/**
 * Created by jiangjiacheng on 2019/6/4.
 */
import java.io.IOException;
import java.util.concurrent.CopyOnWriteArraySet;
 
import javax.websocket.OnClose;
import javax.websocket.OnError;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import org.springframework.stereotype.Component;
import cn.hutool.log.Log;
import cn.hutool.log.LogFactory;
 
@ServerEndpoint("/websocket/{sid}")
@Component
public class WebSocketServer {
 
    static Log log=LogFactory.get(WebSocketServer.class);
 
    //静态变量，用来记录当前在线连接数。应该把它设计成线程安全的。
    private static int onlineCount = 0;
 
    //concurrent包的线程安全Set，用来存放每个客户端对应的MyWebSocket对象。
    private static CopyOnWriteArraySet<WebSocketServer> webSocketSet = new CopyOnWriteArraySet<WebSocketServer>();
 
    //与某个客户端的连接会话，需要通过它来给客户端发送数据
    private Session session;
 
    //接收sid
    private String sid="";
 
    /**
     * 连接建立成功调用的方法*/
    @OnOpen
    public void onOpen(Session session,@PathParam("sid") String sid) {
        this.session = session;
        webSocketSet.add(this);     //加入set中
        addOnlineCount();           //在线数加1
        log.info("有新窗口开始监听:"+sid+",当前在线人数为" + getOnlineCount());
        this.sid=sid;
        /*try {
            sendMessage("连接成功");
        } catch (IOException e) {
            log.error("websocket IO异常");
        }*/
    }
 
    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose() {
        webSocketSet.remove(this);  //从set中删除
        subOnlineCount();           //在线数减1
        log.info("有一连接关闭！当前在线人数为" + getOnlineCount());
    }
 
    /**
     * 收到客户端消息后调用的方法
     *
     * @param message 客户端发送过来的消息*/
    @OnMessage
    public void onMessage(String message, Session session) {
        log.info("收到来自窗口"+sid+"的信息:"+message);
        //群发消息
        for (WebSocketServer item : webSocketSet) {
            try {
                item.sendMessage(message);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
 
    /**
     *
     * @param session
     * @param error
     */
    @OnError
    public void onError(Session session, Throwable error) {
        log.error("发生错误");
        error.printStackTrace();
    }
    /**
     * 实现服务器主动推送
     */
    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }
 
 
    /**
     * 群发自定义消息
     * */
    public static void sendInfo(String message,@PathParam("sid") String sid) throws IOException {
        log.info("推送消息到窗口"+sid+"，推送内容:"+message);
        for (WebSocketServer item : webSocketSet) {
            try {
                //这里可以设定只推送给这个sid的，为null则全部推送
                if(sid == null) {
                    item.sendMessage(message);
                }else if(item.sid.equals(sid)){
                    item.sendMessage(message);
                }
            } catch (IOException e) {
                continue;
            }
        }
    }
 
    public static synchronized int getOnlineCount() {
        return onlineCount;
    }
 
    public static synchronized void addOnlineCount() {
        WebSocketServer.onlineCount++;
    }
 
    public static synchronized void subOnlineCount() {
        WebSocketServer.onlineCount--;
    }
}
```

这样就增加了webSocket的支持啦。

1、首先PC端调用接口展示出来了二维码。

2、请求二维码中的http请求。就有uuid在 header中。直接取到uuid 作为webSocket的标识sid进行连接。

3、然后手机端使用相机拿到二维码中的uuid。使用uuid + userid 请求 扫码成功接口。

贴扫码成功接口

Controller代码：

```
/**
 * 确认身份接口：确定身份以及判断是否二维码过期等
 * @param token
 * @param userId
 * @return
 */
@RequestMapping(value = "/bindUserIdAndToken" ,method = RequestMethod.GET)
@ResponseBody
public Object bindUserIdAndToken(@RequestParam("token") String token ,
                                 @RequestParam("userId") Integer userId,
                                 @RequestParam(required = false,value = "projId") Integer projId){

    try {
        return new SuccessTip(userService.bindUserIdAndToken(userId,token,projId));
    } catch (Exception e) {
        e.printStackTrace();
        return new ErrorTip(500,e.getMessage());
    }

}
```

Service代码

```
@Override
public String bindUserIdAndToken(Integer userId, String token,Integer projId) throws Exception {

    QrLoginToken qrLoginToken = new QrLoginToken();
    qrLoginToken.setToken(token);
    qrLoginToken = qrLoginTokenMapper.selectOne(qrLoginToken);

    if(null == qrLoginToken){
        throw  new Exception("错误的请求！");
    }

    Date createDate = new Date(qrLoginToken.getCreateTime().getTime() + (1000 * 60 * Constant.LOGIN_VALIDATION_TIME));
    Date nowDate = new Date();
    if(nowDate.getTime() > createDate.getTime()){//当前时间大于校验时间

        JSONObject jsonObject = new JSONObject();
        jsonObject.put("code",500);
        jsonObject.put("msg","二维码失效！");
        WebSocketServer.sendInfo(jsonObject.toJSONString(),token);

        throw  new Exception("二维码失效！");
    }

    qrLoginToken.setLoginTime(new Date());
    qrLoginToken.setUserId(userId);

    int i = qrLoginTokenMapper.updateById(qrLoginToken);

    JSONObject jsonObject = new JSONObject();
    jsonObject.put("code",200);
    jsonObject.put("msg","ok");
    jsonObject.put("userId",userId);
    if(ToolUtil.isNotEmpty(projId)){
        jsonObject.put("projId",projId);
    }
    WebSocketServer.sendInfo(jsonObject.toJSONString(),token);

    if(i > 0 ){
        return null;
    }else{
        throw  new Exception("服务器异常！");
    }
}
```

逻辑大概就是判断一下 token对不对。

如果对的话。时间是否过期。如果没有过期进行业务逻辑操作

```
//这句话比较关键
WebSocketServer.sendInfo(jsonObject.toJSONString(),token);
```

就是通知前端已经登录成功了。并且给他业务所需要的内容。

然后前端代码接收到了。就进行业务逻辑操作就可以