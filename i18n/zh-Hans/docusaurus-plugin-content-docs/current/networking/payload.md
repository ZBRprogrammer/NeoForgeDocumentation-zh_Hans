---
sidebar_position: 1
---
# 注册有效载荷(Registering Payloads)

有效载荷(Payloads)是一种在客户端和服务器之间发送任意数据的方式。它们使用 `RegisterPayloadHandlersEvent` 事件中的 `PayloadRegistrar` 进行注册。

```java
@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void register(RegisterPayloadHandlersEvent event) {
    // 设置当前网络版本
    final PayloadRegistrar registrar = event.registrar("1");
}
```

假设我们要发送以下数据：

```java
public record MyData(String name, int age) {}
```

然后我们可以实现 `CustomPacketPayload` 接口来创建一个可用于发送和接收此数据的有效载荷(Payload)。

```java
public record MyData(String name, int age) implements CustomPacketPayload {
    
    public static final CustomPacketPayload.Type<MyData> TYPE = new CustomPacketPayload.Type<>(ResourceLocation.fromNamespaceAndPath("mymod", "my_data"));

    // 每对元素定义用于编码/解码的元素的流编解码器(Stream Codec)和用于编码的元素的获取器(Getter)
    // 'name' 将作为字符串进行编码和解码
    // 'age' 将作为整数进行编码和解码
    // 最后一个参数按提供的顺序接收先前的参数以构造有效载荷(Payload)对象
    public static final StreamCodec<ByteBuf, MyData> STREAM_CODEC = StreamCodec.composite(
        ByteBufCodecs.STRING_UTF8,
        MyData::name,
        ByteBufCodecs.VAR_INT,
        MyData::age,
        MyData::new
    );
    
    @Override
    public CustomPacketPayload.Type<? extends CustomPacketPayload> type() {
        return TYPE;
    }
}
```

正如您从上面的示例中看到的，`CustomPacketPayload` 接口要求我们实现 `type` 方法。`type` 方法负责返回此有效载荷(Payload)的唯一标识符。然后我们还需要一个读取器(Reader)来稍后与 `StreamCodec` 一起注册，以读取和写入有效载荷(Payload)数据。

最后，我们可以将此有效载荷(Payload)注册到注册器(Registrar)：

```java
// 在某个通用事件类中

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void register(RegisterPayloadHandlersEvent event) {
    final PayloadRegistrar registrar = event.registrar("1");
    registrar.playBidirectional(
        MyData.TYPE,
        MyData.STREAM_CODEC,
        ServerPayloadHandler::handleDataOnMain
    );
}

// 在某个仅客户端的事件类中

@SubscribeEvent // 仅在物理客户端上，在模组事件总线(mod event bus)上
public static void register(RegisterClientPayloadHandlersEvent event) {
    event.register(
        MyData.TYPE,
        ClientPayloadHandler::handleDataOnMain
    );
}
```

剖析上面的代码，我们可以注意到以下几点：
- 注册器有 `play*` 方法，可用于注册在游戏游玩阶段(Play Phase)发送的有效载荷(Payload)。
    - 此代码中未显示的方法是 `configuration*` 和 `common*`；但是，它们也可用于为配置阶段(Configuration Phase)注册有效载荷(Payload)。`common` 方法可用于同时为配置阶段(Configuration Phase)和游玩阶段(Play Phase)注册有效载荷(Payload)。
- 注册器使用 `*Bidirectional` 方法，可用于注册发送到逻辑服务器和逻辑客户端的有效载荷(Payload)。
    - 此代码中未显示的方法是 `*ToClient` 和 `*ToServer`；但是，它们也可用于分别仅向逻辑客户端或逻辑服务器注册有效载荷(Payload)。
- 有效载荷(Payload)的类型用作其唯一标识符。
- [流编解码器(Stream Codec)][streamcodec]用于从网络发送的缓冲区读取和写入有效载荷(Payload)数据。
- 有效载荷(Payload)处理器(Handler)是有效载荷(Payload)到达某个逻辑端时的回调函数。
    - 如果使用 `*ToServer` 方法，有效载荷(Payload)处理器将是该方法的最后一个参数。
    - 如果使用 `*ToClient` 方法，则需要通过 `RegisterClientPayloadHandlersEvent` 注册有效载荷(Payload)处理器，传入有效载荷(Payload)类型和处理器。
    - 如果使用 `*Bidirectional` 方法，则需要同时使用两者。

:::注意
客户端有效载荷(Payload)处理器有自己的事件 `RegisterClientPayloadHandlersEvent`，以防止代码跨越逻辑端和物理端[sides]。
:::

现在我们已经注册了有效载荷(Payload)，需要实现一个处理器(Handler)。对于此示例，我们将专门查看客户端处理器，但服务器端处理器非常相似。

```java
public class ClientPayloadHandler {
    
    public static void handleDataOnMain(final MyData data, final IPayloadContext context) {
        // 在主线程上对数据执行某些操作
        blah(data.age());
    }
}
```

这里有几点需要注意：

- 此处的处理方法接收有效载荷(Payload)和一个上下文对象。
- 默认情况下，有效载荷(Payload)的处理方法在主线程上调用。

如果您需要执行一些资源密集型的计算，那么工作应在网络线程上完成，而不是阻塞主线程。这可以通过在注册服务器绑定(Serverbound)连接的有效载荷(Payload)之前，通过 `PayloadRegistrar.executesOn` 将 `PayloadRegistrar` 的 `HandlerThread` 设置为 `HandlerThread.NETWORK` 来实现；对于客户端绑定(Clientbound)连接，则通过将 `HandlerThread` 传递给 `RegisterClientPayloadHandlersEvent.register` 来实现。

```java
// 在某个通用事件类中

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void register(RegisterPayloadHandlersEvent event) {
    final PayloadRegistrar registrar = event.registrar("1")
        .executesOn(HandlerThread.NETWORK); // 所有后续有效载荷(Payload)将在网络线程上注册
    registrar.playBidirectional(
        MyData.TYPE,
        MyData.STREAM_CODEC,
        ServerPayloadHandler::handleDataOnNetwork
    );
}

// 在某个仅客户端的事件类中

@SubscribeEvent // 仅在物理客户端上，在模组事件总线(mod event bus)上
public static void register(RegisterClientPayloadHandlersEvent event) {
    event.register(
        MyData.TYPE,
        HandlerThread.NETWORK // 有效载荷(Payload)处理器将在网络线程上调用
        ClientPayloadHandler::handleDataOnNetwork
    );
}
```

:::注意
在 `executesOn` 调用之后注册的所有有效载荷(Payload)将保留相同的线程执行位置，直到再次调用 `executesOn`。

```java
PayloadRegistrar registrar = event.registrar("1");

registrar.playBidirectional(...); // 在主线程上
registrar.playBidirectional(...); // 在主线程上

// 配置方法会修改注册器的状态
// 通过创建一个新实例，因此需要更新更改
/// 通过存储结果来更新
registrar = registrar.executesOn(HandlerThread.NETWORK);

registrar.playBidirectional(...); // 在网络线程上
registrar.playBidirectional(...); // 在网络线程上

registrar = registrar.executesOn(HandlerThread.MAIN);

registrar.playBidirectional(...); // 在主线程上
registrar.playBidirectional(...); // 在主线程上
```
:::

这里有几点需要注意：

- 如果要在主游戏线程上运行代码，可以使用 `enqueueWork` 向主线程提交任务。
    - 该方法将返回一个在主线程上完成的 `CompletableFuture`。
    - 注意：返回的是一个 `CompletableFuture`，这意味着您可以将多个任务链接在一起，并在单个位置处理异常。
    - 如果您没有在 `CompletableFuture` 中处理异常，那么它将被吞没，**并且您不会收到通知**。

```java
public class ClientPayloadHandler {
    
    public static void handleDataOnNetwork(final MyData data, final IPayloadContext context) {
        // 在网络线程上对数据执行某些操作
        blah(data.name());
        
        // 在主线程上对数据执行某些操作
        context.enqueueWork(() -> {
            blah(data.age());
        })
        .exceptionally(e -> {
            // 处理异常
            context.disconnect(Component.translatable("my_mod.networking.failed", e.getMessage()));
            return null;
        });
    }
}
```

使用您自己的有效载荷(Payload)，您可以使用它们通过[配置任务(Configuration Tasks)][configuration]来配置客户端和服务器。

## 发送有效载荷(Sending Payloads)

`CustomPacketPayload` 通过使用原版的包(Packet)系统在网络上发送，通过 `ServerboundCustomPayloadPacket`（当发送到服务器时）或 `ClientboundCustomPayloadPacket`（当发送到客户端时）包装有效载荷(Payload)。发送到客户端的有效载荷(Payload)最多只能包含 1 MiB 的数据，而发送到服务器的有效载荷(Payload)只能包含小于 32 KiB 的数据。

所有有效载荷(Payload)都通过 `Connection.send` 发送，带有一定程度的抽象；但是，如果您想根据给定条件向多人发送数据包，调用这些方法通常不太方便。因此，`PacketDistributor` 包含许多向客户端发送有效载荷(Payload)的便捷实现，而 `ClientPacketDistributor` 包含一个向服务器发送数据包的方法 (`sendToServer`)。

```java
// 在客户端上

// 向服务器发送有效载荷(Payload)
ClientPacketDistributor.sendToServer(new MyData(...));

// 在服务器上

// 发送给一个玩家 (ServerPlayer serverPlayer)
PacketDistributor.sendToPlayer(serverPlayer, new MyData(...));

/// 发送给跟踪此区块的所有玩家 (ServerLevel serverLevel, ChunkPos chunkPos)
PacketDistributor.sendToPlayersTrackingChunk(serverLevel, chunkPos, new MyData(...));

/// 发送给所有连接的玩家
PacketDistributor.sendToAllPlayers(new MyData(...));
```

有关更多实现，请参阅 `PacketDistributor` 和 `ClientPacketDistributor` 类。

[configuration]: configuration-tasks.md
[sides]: ../concepts/sides.md
[streamcodec]: streamcodecs.md