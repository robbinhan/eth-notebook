## Ethereum protocol
node间p2p通信的顶层结构体叫eth.ProtocolManager(在eth/handler.go的66行)

![](http://img.blog.csdn.net/20171108141219288?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGVhc3ByaW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- Fetcher类型成员累积所有其他个体发送来的有关新数据的宣布消息，并在自身对照后，安排相应的获取请求。
- Downloader类型成员负责所有向相邻个体主动发起的同步流程。



## Whisper protocol（shh）

现在用的是v5版(实验性的协议)

```
细语(Whisper)是ĐΞV目前正在开发的一个处于概念阶段的协议，Whisper是加密通信传输系统, 计划成为一个通用的点到点（P2P）通信协议。在以太坊的每个节点可以为自己生成一个基于公钥的地址。细语可让客户端将消息发送给某个特定的接收者，或者通过附加到信息里的描述性标签或“主题“，将消息广播给多个接收者。 节点会聪明的利用尽可能多的有效信息，在彼此之间的路由消息。信息包括已知的有趣的话题（节点可以自由的用广播过滤他们对之感兴趣的话题），以及消息的生存时间（短的消息往往有更高的优先级）。所有的信息都含有生存时间，这样即使接收者处于离线状态，消息可最终通过。
```


![](https://camo.githubusercontent.com/97d1e4c80d5784875b32932566e0b486c5220828/687474703a2f2f626974636f696e386274632e71696e6975646e2e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f31312f66696c65303030372e6a7067)