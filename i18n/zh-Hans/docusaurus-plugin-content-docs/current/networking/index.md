# 网络(Networking)

服务器和客户端之间的通信是成功实现模组的支柱。

网络通信有两个主要目标：

1.  确保客户端视图与服务器视图“同步”
    - 坐标 (X, Y, Z) 处的花刚刚生长
2.  为客户端提供一种告诉服务器玩家已发生某些变化的方式
    - 玩家按下了某个键

实现这些目标的最常见方式是在客户端和服务器之间传递消息。这些消息通常是结构化的，包含特定排列的数据，以便于发送和接收。

NeoForge 提供了一种主要基于 [Netty] 构建的通信技术。通过监听 `RegisterPayloadHandlersEvent` 事件，然后向注册器(Registrar)注册特定类型的[有效载荷(Payloads)]、其读取器(Reader)及其处理函数，即可使用此技术。

[netty]: https://netty.io "Netty 网站"
[payloads]: payload.md "注册自定义有效载荷(Registering custom Payloads)"