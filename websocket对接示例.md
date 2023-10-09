### 系统通告业务对接示例

系统通告管理中，管理员可发两种类型的消息，全体用户和指定用户发送

发送的消息是一个json串

#### （1）调用WebSocket 服务

```
@Resource
private WebSocket webSocket;
```

#### （2）方法中调用

> cmd为业务类型，例如topic表示系统消息，user表示用户消息，可以自定义cmd类型，客户端根据返回的cmd类型处理不同的业务响应

**全体发送**

```
//创建业务消息信息
JSONObject obj = new JSONObject();
obj.put("cmd", "topic");//业务类型
obj.put("msgId", sysAnnouncement.getId());//消息id
obj.put("msgTxt", sysAnnouncement.getTitile());//消息内容
//全体发送
webSocket.sendAllMessage(obj.toJSONString());
								
```

**单个用户发送**

```
//创建业务消息信息
JSONObject obj = new JSONObject();
obj.put("cmd", "user");//业务类型
obj.put("msgId", sysAnnouncement.getId());//消息id
obj.put("msgTxt", sysAnnouncement.getTitile());//消息内容
//单个用户发送 (userId为用户id)
webSocket.sendOneMessage(userId, obj.toJSONString());
								
```

**多个用户发送**

```
//创建业务消息信息
JSONObject obj = new JSONObject();
obj.put("cmd", "user");//业务类型
obj.put("msgId", sysAnnouncement.getId());//消息id
obj.put("msgTxt", sysAnnouncement.getTitile());//消息内容
//多个用户发送 (userIds为多个用户id，逗号‘,’分隔)
webSocket.sendMoreMessage(userIds, obj.toJSONString());
								
```

#### （3）vue 客户端根据返回的cmd类型处理不同的业务响应

```
websocketonmessage: function (e) {
        console.log("-----接收消息-------",e.data);
        var data = eval("(" + e.data + ")"); //解析json对象
        if(data.cmd == "topic"){
          //TODO 系统通知
            
        }else if(data.cmd == "user"){
          //TODO 用户消息
        
        }

},
```