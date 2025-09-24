# 让逆向再次伟大
# MSGA - Make Reverse Great Again！
# 会断点就会逆向！
# 项目基于JsRpc和msedge 注入！
```sh
DOCKER-COMPOSE.YML
---
services:
  msedge:
    image: smallfawn/mrga:latest
    container_name: mrga
    security_opt:
      - seccomp:unconfined #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - EDGE_CLI=https://www.linuxserver.io/ #optional
    volumes:
      - /:/config
    ports:
      - 3000:3000
      - 3001:3001
      - 12080:12080
    shm_size: "1gb"
    restart: unless-stopped
```

**api 简介**

- `/list` :查看当前连接的ws服务  (get)
- `/ws`  :浏览器注入ws连接的接口 (ws | wss)
- `/wst`  :ws测试使用-发啥回啥 (ws | wss)
- `/go` :获取数据的接口  (get | post)
- `/execjs` :传递jscode给浏览器执行 (get | post)
- `/page/cookie` :直接获取当前页面的cookie (get)
- `/page/html` :获取当前页面的html (get)

说明：接口用?group分组 如 "ws://127.0.0.1:12080/ws?group={}"
以及可选参数 clientId
clientId说明：以group分组后，如果有注册相同group的 可以传入这个id来区分客户端，如果不传 服务程序会自动生成一个。当访问调用接口时，服务程序随机发送请求到相同group的客户端里。

//注入例子 group可以随便起名(必填)
http://127.0.0.1:12080/go?group={}&action={}&param={} //这是调用的接口
group填写上面注入时候的，action是注册的方法名,param是可选的参数 param可以传string类型或者object类型(会尝试用JSON.parse)

### 注入JS，构建通信环境（(查看仓库目录下env.js)）

打开JsEnv 复制粘贴到网站控制台(注意：可以在浏览器开启的时候就先注入环境，不要在调试断点时候注入)

![image](https://github.com/jxhczhl/JsRpc/assets/41224971/799fd2ce-28f6-4719-9ff8-e60da57068d7")



### 连接通信

```js
// 注入环境后连接通信
var demo = new Hlclient("ws://127.0.0.1:12080/ws?group=zzz");
// 可选  
//var demo = new Hlclient("ws://127.0.0.1:12080/ws?group=zzz&clientId=hliang/"+new Date().getTime())
```

#### I 远程调用0：

##### 接口传js代码让浏览器执行

浏览器已经连接上通信后 调用execjs接口就行

```python
import requests

js_code = """
(function(){
    console.log("test")
    return "执行成功"
})()
"""

url = "http://localhost:12080/execjs"
data = {
    "group": "zzz",
    "code": js_code
}
res = requests.post(url, data=data)
print(res.text)
```

![image](https://user-images.githubusercontent.com/41224971/165704850-0a22dd7e-68ea-44fe-bda9-608c10795558.png)

#### Ⅱ 远程调用1： 浏览器预先注册js方法 传递函数名调用

##### 远程调用1：无参获取值

```js

// 注册一个方法 第一个参数hello为方法名，
// 第二个参数为函数，resolve里面的值是想要的值(发送到服务器的)
demo.regAction("hello", function (resolve) {
    //这样每次调用就会返回“好困啊+随机整数”
    var Js_sjz = "好困啊"+parseInt(Math.random()*1000);
    resolve(Js_sjz);
})


```

访问接口，获得js端的返回值  
http://127.0.0.1:12080/go?group=zzz&action=hello

![image](https://github.com/jxhczhl/JsRpc/assets/41224971/5f0da051-18f3-49ac-98f8-96f408440475)


##### 远程调用2：带参获取值

```js
//写一个传入字符串，返回base64值的接口(调用内置函数btoa)
demo.regAction("hello2", function (resolve,param) {
    //这样添加了一个param参数，http接口带上它，这里就能获得
    var base666 = btoa(param)
    resolve(base666);
})
```

访问接口，获得js端的返回值
http://127.0.0.1:12080/go?group=zzz&action=hello2&param=123456  

![image](https://github.com/jxhczhl/JsRpc/assets/41224971/91b993ae-7831-4b65-8553-f90e19cc7ebe)


##### 远程调用3：带多个参获 并且使用post方式 取值

```js
//假设有一个函数 需要传递两个参数
function hlg(User,Status){
    return User+"说："+Status;
}

demo.regAction("hello3", function (resolve,param) {
    //这里还是param参数 param里面的key 是先这里写，但到时候传接口就必须对应的上
    res=hlg(param["user"],param["status"])
    resolve(res);
})
```

访问接口，获得js端的返回值

```python
url = "http://127.0.0.1:12080/go"
data = {
    "group": "zzz",
    "action": "hello3",
    "param": json.dumps({"user":"黑脸怪","status":"好困啊"})
}
print(data["param"]) #dumps后就是长这样的字符串{"user": "\u9ed1\u8138\u602a", "status": "\u597d\u56f0\u554a"}
res=requests.post(url, data=data) #这里换get也是可以的
print(res.text)
```

![image](https://github.com/jxhczhl/JsRpc/assets/41224971/5af9bf90-cdfd-4d89-a3c0-a11a54ca7969)


##### 远程调用4：获取页面基础信息

```python
resp = requests.get("http://127.0.0.1:12080/page/html?group=zzz")     # 直接获取当前页面的html
resp = requests.get("http://127.0.0.1:12080/page/cookie?group=zzz")   # 直接获取当前页面的cookie
```


list接口可查看当前注入的客户端信息  
<img width="321" alt="image" src="https://github.com/jxhczhl/JsRpc/assets/41224971/5b2ac7af-f6f0-4569-ac64-553ea41be387">

---