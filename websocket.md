##### （1）maven依赖

```
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

#### （2）WebSocket配置类

```
package org.jeecg.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

@Configuration
public class WebSocketConfig {
    /**
     * 	注入ServerEndpointExporter，
     * 	这个bean会自动注册使用了@ServerEndpoint注解声明的Websocket endpoint
     */
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
    
}
```

#### WebSocket操作类

> 通过该类WebSocket可以进行群推送以及单点推送

```
package org.jeecg.modules.message.websocket;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import javax.websocket.*;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;

import com.alibaba.fastjson.JSONObject;
import org.jeecg.common.base.BaseMap;
import org.jeecg.common.constant.WebsocketConst;
import org.jeecg.common.modules.redis.client.JeecgRedisClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

/**
 * @Author scott
 * @Date 2019/11/29 9:41
 * @Description: 此注解相当于设置访问URL
 */
@Component
@Slf4j
@ServerEndpoint("/websocket/{userId}")
public class WebSocket {
    
    /**线程安全Map*/
    private static ConcurrentHashMap<String, Session> sessionPool = new ConcurrentHashMap<>();

    /**
     * Redis触发监听名字
     */
    public static final String REDIS_TOPIC_NAME = "socketHandler";
    @Autowired
    private JeecgRedisClient jeecgRedisClient;


    //==========【websocket接受、推送消息等方法 —— 具体服务节点推送ws消息】========================================================================================
    @OnOpen
    public void onOpen(Session session, @PathParam(value = "userId") String userId) {
        try {
            sessionPool.put(userId, session);
            log.info("【系统 WebSocket】有新的连接，总数为:" + sessionPool.size());
        } catch (Exception e) {
        }
    }

    @OnClose
    public void onClose(@PathParam("userId") String userId) {
        try {
            sessionPool.remove(userId);
            log.info("【系统 WebSocket】连接断开，总数为:" + sessionPool.size());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * ws推送消息
     *
     * @param userId
     * @param message
     */
    public void pushMessage(String userId, String message) {
        for (Map.Entry<String, Session> item : sessionPool.entrySet()) {
            //userId key值= {用户id + "_"+ 登录token的md5串}
            //TODO vue2未改key新规则，暂时不影响逻辑
            if (item.getKey().contains(userId)) {
                Session session = item.getValue();
                try {
                    //update-begin-author:taoyan date:20211012 for: websocket报错 https://gitee.com/jeecg/jeecg-boot/issues/I4C0MU
                    synchronized (session){
                        log.info("【系统 WebSocket】推送单人消息:" + message);
                        session.getBasicRemote().sendText(message);
                    }
                    //update-end-author:taoyan date:20211012 for: websocket报错 https://gitee.com/jeecg/jeecg-boot/issues/I4C0MU
                } catch (Exception e) {
                    log.error(e.getMessage(),e);
                }
            }
        }
    }

    /**
     * ws遍历群发消息
     */
    public void pushMessage(String message) {
        try {
            for (Map.Entry<String, Session> item : sessionPool.entrySet()) {
                try {
                    item.getValue().getAsyncRemote().sendText(message);
                } catch (Exception e) {
                    log.error(e.getMessage(), e);
                }
            }
            log.info("【系统 WebSocket】群发消息:" + message);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
    }


    /**
     * ws接受客户端消息
     */
    @OnMessage
    public void onMessage(String message, @PathParam(value = "userId") String userId) {
        if(!"ping".equals(message) && !WebsocketConst.CMD_CHECK.equals(message)){
            log.info("【系统 WebSocket】收到客户端消息:" + message);
        }else{
            log.debug("【系统 WebSocket】收到客户端消息:" + message);
        }
        
//        //------------------------------------------------------------------------------
//        JSONObject obj = new JSONObject();
//        //业务类型
//        obj.put(WebsocketConst.MSG_CMD, WebsocketConst.CMD_CHECK);
//        //消息内容
//        obj.put(WebsocketConst.MSG_TXT, "心跳响应");
//        this.pushMessage(userId, obj.toJSONString());
//        //------------------------------------------------------------------------------
    }

    /**
     * 配置错误信息处理
     *
     * @param session
     * @param t
     */
    @OnError
    public void onError(Session session, Throwable t) {
        log.warn("【系统 WebSocket】消息出现错误");
        t.printStackTrace();
    }
    //==========【系统 WebSocket接受、推送消息等方法 —— 具体服务节点推送ws消息】========================================================================================
    

    //==========【采用redis发布订阅模式——推送消息】========================================================================================
    /**
     * 后台发送消息到redis
     *
     * @param message
     */
    public void sendMessage(String message) {
        //log.info("【系统 WebSocket】广播消息:" + message);
        BaseMap baseMap = new BaseMap();
        baseMap.put("userId", "");
        baseMap.put("message", message);
        jeecgRedisClient.sendMessage(REDIS_TOPIC_NAME, baseMap);
    }

    /**
     * 此为单点消息 redis
     *
     * @param userId
     * @param message
     */
    public void sendMessage(String userId, String message) {
        BaseMap baseMap = new BaseMap();
        baseMap.put("userId", userId);
        baseMap.put("message", message);
        jeecgRedisClient.sendMessage(REDIS_TOPIC_NAME, baseMap);
    }

    /**
     * 此为单点消息(多人) redis
     *
     * @param userIds
     * @param message
     */
    public void sendMessage(String[] userIds, String message) {
        for (String userId : userIds) {
            sendMessage(userId, message);
        }
    }
    //=======【采用redis发布订阅模式——推送消息】==========================================================================================
    
}
```

### 前端中VUE使用WebSocket

```
<script>
    import store from '@/store/'

    export default {
        data() {
            return {
            }
        },
        mounted() { 
              //初始化websocket
              this.initWebSocket()
        },
        destroyed: function () { // 离开页面生命周期函数
              this.websocketclose();
        },
        methods: {
            initWebSocket: function () {
                // WebSocket与普通的请求所用协议有所不同，ws等同于http，wss等同于https
                var userId = store.getters.userInfo.id;
                var url = window._CONFIG['domianURL'].replace("https://","ws://").replace("http://","ws://")+"/websocket/"+userId;
                this.websock = new WebSocket(url);
                this.websock.onopen = this.websocketonopen;
                this.websock.onerror = this.websocketonerror;
                this.websock.onmessage = this.websocketonmessage;
                this.websock.onclose = this.websocketclose;
              },
              websocketonopen: function () {
                console.log("WebSocket连接成功");
              },
              websocketonerror: function (e) {
                console.log("WebSocket连接发生错误");
              },
              websocketonmessage: function (e) {
                var data = eval("(" + e.data + ")"); 
                 //处理订阅信息
                if(data.cmd == "topic"){
                   //TODO 系统通知
             
                }else if(data.cmd == "user"){
                   //TODO 用户消息
        
                }
              },
              websocketclose: function (e) {
                console.log("connection closed (" + e.code + ")");
              }
        }
    }
</script>
```