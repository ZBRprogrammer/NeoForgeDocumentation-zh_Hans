---
sidebar_position: 3
---
# 使用配置任务(Using Configuration Tasks)

客户端和服务器之间的网络协议有一个特定的阶段，在玩家实际加入游戏之前，服务器可以配置客户端。这个阶段称为配置阶段(Configuration Phase)，例如，原版服务器使用此阶段向客户端发送资源包信息。

模组也可以使用此阶段在玩家加入游戏之前配置客户端。

## 注册配置任务(Registering a configuration task)

使用配置阶段的第一步是注册一个配置任务。这可以通过在 `RegisterConfigurationTasksEvent` 事件中注册一个新的配置任务来完成。

```java
@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void register(final RegisterConfigurationTasksEvent event) {
    event.register(new MyConfigurationTask());
}
```

`RegisterConfigurationTasksEvent` 事件在模组总线(mod bus)上触发，并暴露服务器用于配置相关客户端的当前监听器。模组开发者可以使用暴露的监听器来判断客户端是否正在运行该模组，如果是，则注册一个配置任务。

## 实现配置任务(Implementing a configuration task)

配置任务是一个简单的接口：`ICustomConfigurationTask`。此接口有两个方法：`void run(Consumer<CustomPacketPayload> sender);` 和 `ConfigurationTask.Type type();`，后者返回配置任务的类型。类型用于标识配置任务。下面显示了一个配置任务的示例：

```java
public record MyConfigurationTask implements ICustomConfigurationTask {
    public static final ConfigurationTask.Type TYPE = new ConfigurationTask.Type(ResourceLocation.fromNamespaceAndPath("mymod", "my_task"));
    
    @Override
    public void run(final Consumer<CustomPacketPayload> sender) {
        final MyData payload = new MyData();
        sender.accept(payload);
    }

    @Override
    public ConfigurationTask.Type type() {
        return TYPE;
    }
}
```

## 确认配置任务(Acknowledging a configuration task)

您的配置在服务器上执行，服务器需要知道何时可以执行下一个配置任务。这是通过确认所述配置任务的执行来完成的。

有两种主要方法可以实现这一点：

### 捕获监听器(Capturing the listener)

当客户端不需要确认配置任务时，可以捕获监听器，并在服务器端直接确认配置任务。

```java
public record MyConfigurationTask(ServerConfigurationPacketListener listener) implements ICustomConfigurationTask {
    public static final ConfigurationTask.Type TYPE = new ConfigurationTask.Type(ResourceLocation.fromNamespaceAndPath("mymod", "my_task"));
    
    @Override
    public void run(final Consumer<CustomPacketPayload> sender) {
        final MyData payload = new MyData();
        sender.accept(payload);
        this.listener().finishCurrentTask(this.type());
    }

    @Override
    public ConfigurationTask.Type type() {
        return TYPE;
    }
}
```

要使用这样的配置任务，需要在 `RegisterConfigurationTasksEvent` 事件中捕获监听器。

```java
@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void register(final RegisterConfigurationTasksEvent event) {
    event.register(new MyConfigurationTask(event.getListener()));
}
```

然后，当前配置任务完成后将立即执行下一个配置任务，客户端无需确认配置任务。此外，服务器不会等待客户端正确处理发送的有效载荷(Payload)。

### 确认配置任务(Acknowledging the configuration task)

当客户端需要确认配置任务时，您需要向客户端发送自己的有效载荷(Payload)：

```java
public record AckPayload() implements CustomPacketPayload {
    public static final CustomPacketPayload.Type<AckPayload> TYPE = new CustomPacketPayload.Type<>(ResourceLocation.fromNamespaceAndPath("mymod", "ack"));
    
    // 无数据的单元编解码器(Unit codec)
    public static final StreamCodec<ByteBuf, AckPayload> STREAM_CODEC = StreamCodec.unit(new AckPayload());

    @Override
    public CustomPacketPayload.Type<? extends CustomPacketPayload> type() {
        return TYPE;
    }
}
```

当服务器端配置任务的有效载荷(Payload)被正确处理时，您可以向服务器发送此有效载荷(Payload)以确认配置任务。

```java
public void onMyData(MyData data, IPayloadContext context) {
    context.enqueueWork(() -> {
        blah(data.name());
    })
    .exceptionally(e -> {
        // 处理异常
        context.disconnect(Component.translatable("my_mod.configuration.failed", e.getMessage()));
        return null;
    })
    .thenAccept(v -> {
        context.reply(new AckPayload());
    });     
}
```

其中 `onMyData` 是服务器端配置任务发送的有效载荷(Payload)的处理程序。

当服务器收到此有效载荷(Payload)时，它将确认配置任务，并执行下一个配置任务：

```java
public void onAck(AckPayload payload, IPayloadContext context) {
    context.finishCurrentTask(MyConfigurationTask.TYPE);
}
```

其中 `onAck` 是客户端发送的有效载荷(Payload)的处理程序。

## 停滞登录过程(Stalling the login process)

如果配置未得到确认，服务器将永远等待，客户端将永远无法加入游戏。因此，除非配置任务失败（此时可以断开客户端连接），否则始终确认配置任务非常重要。