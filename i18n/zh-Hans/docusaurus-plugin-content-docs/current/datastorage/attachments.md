---
sidebar_position: 4
---
# 数据附加(Data Attachments)

数据附加(Data Attachment)系统允许模组在方块实体(block entities)、区块(chunks)、实体(entities)和维度(levels)上附加和存储额外的数据。

_要存储额外的维度数据，可以使用[SavedData][saveddata]。_

:::note
物品堆栈(Item Stack)的数据附加功能已被原版的[数据组件(data components)][datacomponents]取代。
:::

## 创建附加类型(Attachment Type)

要使用此系统，你需要注册一个`AttachmentType`。附加类型包含以下配置：

-   一个默认值供应器(default value supplier)，用于在首次访问数据时创建实例。
-   一个可选的序列化器(serializer)，用于在需要持久化附加数据时使用。
-   （如果配置了序列化器）`copyOnDeath`标志，用于在死亡时自动复制实体数据（见下文）。

:::tip
如果你不希望你的附件被持久化，请不要提供序列化器。
:::

提供附件序列化器有几种方法：直接实现`IAttachmentSerializer`，实现[`ValueIOSerializable`][valueio]并使用静态方法`AttachmentType#serializable`创建构建器，或者向构建器提供一个映射编解码器(Map Codec)。

无论哪种情况，附件**必须注册**到`NeoForgeRegistries.ATTACHMENT_TYPES`注册表。以下是一个例子：

``` java
// 为附件类型创建 DeferredRegister
private static final DeferredRegister<AttachmentType<?>> ATTACHMENT_TYPES = DeferredRegister.create(NeoForgeRegistries.ATTACHMENT_TYPES, MOD_ID);

// 通过 ValueIOSerializable 序列化
private static final Supplier<AttachmentType<ItemStacksResourceHandler>> HANDLER = ATTACHMENT_TYPES.register(
    "handler", () -> AttachmentType.serializable(() -> new ItemStacksResourceHandler(1)).build()
);
// 通过映射编解码器序列化
private static final Supplier<AttachmentType<Integer>> MANA = ATTACHMENT_TYPES.register(
    "mana", () -> AttachmentType.builder(() -> 0).serialize(Codec.INT.fieldOf("mana")).build()
);
// 无序列化
private static final Supplier<AttachmentType<SomeCache>> SOME_CACHE = ATTACHMENT_TYPES.register(
    "some_cache", () -> AttachmentType.builder(() -> new SomeCache()).build()
);

// 在你的模组构造函数中，别忘了将 DeferredRegister 注册到你的模组事件总线(mod bus)：
ATTACHMENT_TYPES.register(modBus);
```

## 使用附加类型

一旦附加类型注册完成，就可以在任何持有者对象上使用。如果数据不存在，调用`getData`会附加一个新的默认实例。

``` java
// 如果 ItemStacksResourceHandler 已存在则获取，否则附加一个新的：
ItemStacksResourceHandler handler = chunk.getData(HANDLER);
// 如果玩家当前魔力值可用则获取，否则附加 0：
int playerMana = player.getData(MANA);
// 以此类推...
```

如果不希望附加默认实例，可以添加`hasData`检查：

``` java
// 在执行任何操作前，检查区块是否有 HANDLER 附件。
if (chunk.hasData(HANDLER)) {
    ItemStacksResourceHandler handler = chunk.getData(HANDLER);
    // 使用 chunk.getData(HANDLER) 做一些事情。
}
```

数据也可以通过`setData`更新：

``` java
// 将魔力值增加 10。
player.setData(MANA, player.getData(MANA) + 10);
```

:::important
通常，方块实体和区块在修改后需要标记为脏数据（使用`setChanged`和`setUnsaved(true)`）。对于`setData`的调用，这会自动完成：

``` java
chunk.setData(MANA, chunk.getData(MANA) + 10); // 将自动调用 setUnsaved
```

但如果你修改了从`getData`获取的数据（包括一个新创建的默认实例），那么你必须显式地将方块实体和区块标记为脏数据：

``` java
var mana = chunk.getData(MUTABLE_MANA);
mana.set(10);
chunk.setUnsaved(true); // 必须手动完成，因为我们没有使用 setData
```
:::

## 与客户端共享数据

要将方块实体、区块、维度或实体附件同步到客户端，你可以在构建器中实现`sync`。当附件通过`AttachmentHolder#getData`默认创建、通过`AttachmentHolder#setData`更新或通过`AttachmentHolder#removeData`移除时，附件会被发送到客户端。如果需要在其他时间发送数据，则可以调用`AttachmentHolder#syncData`并传入`AttachmentType`来进行同步。

`AttachmentType.Builder#sync`有三个重载方法；不过，它们各自都创建一个`AttachmentSyncHandler<T>`，其中`T`是数据附件的类型。该处理器有三个方法：两个用于从网络`read`（读取）和`write`（写入），一个用于确定给定玩家是否可以看到持有者广播的数据（`sendToPlayer`）。如果数据附件被移除，同步处理器将被忽略。

``` java
public class ExampleSyncHandler implements AttachmentSyncHandler<ExampleData> {

    @Override
    public void write(RegistryFriendlyByteBuf buf, ExampleData attachment, boolean initialSync) {
        // 将附件数据写入缓冲区
        // 如果 `initialSync` 为 true，你应该写入整个附件，因为客户端没有任何先前的数据
        // 如果 `initialSync` 为 false，你可以选择只写入你想要更新的数据
        
        // 示例：
        if (initialSync) {
            // 写入整个附件
            ExampleData.STREAM_CODEC.encode(buf, attachment);
        } else {
            // 写入更新数据
        }
    }

    @Override
    @Nullable
    public ExampleData read(IAttachmentHolder holder, RegistryFriendlyByteBuf buf, @Nullable ExampleData previousValue) {
        // 从缓冲区读取数据并返回新的数据附件
        // 如果客户端先前没有数据，则 `previousValue` 为 `null`
        // 如果应该移除数据附件，则结果应返回 `null`

        // 示例：
        if (previousValue == null) {
            // 读取整个附件
            return ExampleData.STREAM_CODEC.decode(buf);
        } else {
            // 读取更新数据并合并到先前值
            return previousValue;
        }
    }

    @Override
    public boolean sendToPlayer(IAttachmentHolder holder, ServerPlayer to) {
        // 返回持有者数据是否同步到给定玩家的客户端
        // 检查的玩家根据附件持有者的不同而不同：
        // - 方块实体：所有跟踪该方块实体所在区块的玩家
        // - 区块：所有跟踪该区块的玩家
        // - 实体：所有跟踪当前实体的玩家，如果当前玩家是附件持有者，则包括该玩家
        // - 维度：当前维度/等级中的所有玩家

        // 示例：
        // 仅当他们是附件持有者时才发送附件
        return holder == to;
    }
}
```

另外两个委托给`AttachmentSyncHandler`的重载方法接收一个[`StreamCodec`][streamcodec]用于`read`和`write`，以及一个可选的谓词(predicate)用于`sendToPlayer`。

``` java
// 假设 ExampleData 有某个流编解码器 STREAM_CODEC

// 同步处理器
public static final Supplier<AttachmentType<ExampleData>> WITH_SYNC_HANDLER = ATTACHMENT_TYPES.register(
    "with_sync_handler", () -> AttachmentType.builder(() -> new ExampleData())
        .sync(new ExampleSyncHandler())
        .build()
);


// 流编解码器
public static final Supplier<AttachmentType<ExampleData>> WITH_STREAM_CODEC = ATTACHMENT_TYPES.register(
    "with_stream_codec", () -> AttachmentType.builder(() -> new ExampleData())
        .sync(ExampleData.STREAM_CODEC)
        .build()
);

// 带谓词的流编解码器
public static final Supplier<AttachmentType<ExampleData>> WITH_PREDICATE = ATTACHMENT_TYPES.register(
    "with_predicate", () -> AttachmentType.builder(() -> new ExampleData())
        .sync((holder, to) -> holder == to, ExampleData.STREAM_CODEC)
        .build()
);
```

:::note
使用`StreamCodec`重载意味着每次都会同步整个数据附件，而忽略客户端上已有的任何数据。
:::

## 玩家死亡时复制数据

默认情况下，[实体(entity)]数据附件不会在玩家死亡时被复制。要在玩家死亡时自动复制附件，请在附件构建器中设置`copyOnDeath`。

更复杂的处理可以通过`PlayerEvent.Clone`事件实现，通过从原始实体读取数据并将其分配给新实体。在此事件中，`#isWasDeath`方法可用于区分死亡后重生和从末地返回。这很重要，因为从末地返回时数据已经存在，因此必须注意不要在这种情况下复制值。

例如：

``` java
@SubscribeEvent // 在游戏事件总线上
public static void onClone(PlayerEvent.Clone event) {
    if (event.isWasDeath() && event.getOriginal().hasData(MY_DATA)) {
        event.getEntity().getData(MY_DATA).fieldToCopy = event.getOriginal().getData(MY_DATA).fieldToCopy;
    }
}
```

[datacomponents]: ../items/datacomponents.md
[entity]: ../entities/index.md
[saveddata]: saveddata.md
[streamcodec]: ../networking/streamcodecs.md
[valueio]: valueio.md#valueioserializable