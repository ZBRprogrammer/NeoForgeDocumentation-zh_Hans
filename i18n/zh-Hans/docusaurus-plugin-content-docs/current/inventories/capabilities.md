---
sidebar_position: 1
---
# 能力（Capabilities）

能力允许以动态和灵活的方式暴露功能，而无需直接实现许多接口。

一般来说，每个能力以接口的形式提供一个功能。

NeoForge 为方块、实体和物品堆添加了能力支持。以下部分将详细解释这一点。

## 为什么要使用能力？

能力旨在将**什么**方块、实体或物品堆可以做与**如何**做分离开来。如果你在犹豫能力是否是适合工作的工具，请问自己以下问题：

1. 我只关心**什么**方块、实体或物品堆可以做，但不关心**如何**做吗？
2. 行为（**什么**）仅适用于某些方块、实体或物品堆，而不是全部吗？
3. 行为的实现（**如何**）取决于特定的方块、实体或物品堆吗？

以下是一些良好的能力使用示例：

- *“我希望我的流体容器与其他模组的流体容器兼容，但我不知道每个流体容器的具体细节。”* - 是的，使用 `ResourceHandler<FluidResource>` 能力。
- *“我想计算某些实体中有多少物品，但我不知道实体可能如何存储它们。”* - 是的，使用 `ResourceHandler<ItemResource>` 能力。
- *“我想给某个物品堆充能，但我不知道物品堆可能如何存储能量。”* - 是的，使用 `EnergyHandler` 能力。
- *“我想给玩家当前目标方块上色，但我不知道方块将如何被转换。”* - 是的。NeoForge 没有提供给方块上色的能力，但你可以自己实现一个。

以下是一个不鼓励使用能力的示例：

- *“我想检查一个实体是否在我的机器范围内。”* - 不，请改用辅助方法。

## NeoForge 提供的能力

NeoForge 为以下三个接口提供了能力：`ResourceHandler<ItemResource>`、`ResourceHandler<FluidResource>` 和 `EnergyHandler`。

`ResourceHandler<ItemResource>` 公开了处理物品栏槽位的接口。类型为 `ResourceHandler<ItemResource>` 的能力有：

- `Capabilities.Item.BLOCK`：方块的自动化可访问物品栏（用于箱子、机器等）。
- `Capabilities.Item.ENTITY`：实体的物品栏内容（额外的玩家槽位、生物/生物的物品栏/袋子）。
- `Capabilities.Item.ENTITY_AUTOMATION`：实体的自动化可访问物品栏（船、矿车等）。
- `Capabilities.Item.ITEM`：物品堆的内容（便携背包等）。

`ResourceHandler<FluidResource>` 公开了处理流体物品栏的接口。类型为 `ResourceHandler<FluidResource>` 的能力有：

- `Capabilities.Fluid.BLOCK`：方块的自动化可访问流体物品栏。
- `Capabilities.Fluid.ENTITY`：实体的流体物品栏。
- `Capabilities.Fluid.ITEM`：物品堆的流体物品栏。

`EnergyHandler` 公开了处理能量容器的接口。它基于 TeamCoFH 的 RedstoneFlux API。类型为 `EnergyHandler` 的能力有：

- `Capabilities.Energy.BLOCK`：方块中包含的能量。
- `Capabilities.Energy.ENTITY`：实体中包含的能量。
- `Capabilities.Energy.ITEM`：物品堆中包含的能量。

## 创建能力

NeoForge 支持方块、实体和物品堆的能力。

能力允许通过一些分发逻辑查找某些 API 的实现。NeoForge 中实现了以下几种能力：

- `BlockCapability`：方块和方块实体的能力；行为取决于特定的 `Block`。
    - 能力通常为不同的面指定一个 `Direction` 上下文。
- `EntityCapability`：实体的能力：行为取决于特定的 `EntityType`。
    - 能力通常为不同的面指定一个 `Direction` 上下文。
- `ItemCapability`：物品堆的能力：行为取决于特定的 `Item`。
    - 能力通常为持有的物品资源指定一个 `ItemAccess` 上下文。

:::tip
为了与其他模组兼容，我们建议尽可能使用 `Capabilities` 类中由 NeoForge 提供的能力。否则，你可以按照本节所述创建自己的能力。
:::

创建能力是一个单一的函数调用，结果对象应存储在 `static final` 字段中。必须提供以下参数：

- 能力的名称。
    - 多次创建同名能力将始终返回同一个对象。
    - 不同名称的能力**完全独立**，可以用于不同的目的。
- 被查询的行为类型。这是类型参数 `T`。
- 查询中额外上下文的类型。这是类型参数 `C`。

例如，以下是如何声明一个面向侧面（side-aware）的方块 `ResourceHandler<ItemResource>` 能力：

``` java
public static final BlockCapability<ResourceHandler<ItemResource>, @Nullable Direction> ITEM_HANDLER_BLOCK =
    BlockCapability.create(
        // 提供一个名称以唯一标识该能力。
        ResourceLocation.fromNamespaceAndPath("mymod", "item_handler"),
        // 提供查询的类型。这里，我们想要查找 `ResourceHandler<ItemResource>` 实例。
        ResourceHandler.asClass(),
        // 提供上下文类型。我们将允许查询接收一个额外的 `Direction side` 参数。
        Direction.class);
```

`@Nullable Direction` 对于方块来说非常常见，因此有一个专用的辅助方法：

``` java
public static final BlockCapability<ResourceHandler<ItemResource>, @Nullable Direction> ITEM_HANDLER_BLOCK =
    BlockCapability.createSided(
        // 提供一个名称以唯一标识该能力。
        ResourceLocation.fromNamespaceAndPath("mymod", "item_handler"),
        // 提供查询的类型。这里，我们想要查找 `ResourceHandler<ItemResource>` 实例。
        ResourceHandler.asClass());
```

如果不需要上下文，则应使用 `Void`。对于无上下文的能力，也有一个专用的辅助方法：

``` java
public static final BlockCapability<ResourceHandler<ItemResource>, Void> ITEM_HANDLER_NO_CONTEXT =
    BlockCapability.createVoid(
        // 提供一个名称以唯一标识该能力。
        ResourceLocation.fromNamespaceAndPath("mymod", "item_handler_no_context"),
        // 提供查询的类型。这里，我们想要查找 `ResourceHandler<ItemResource>` 实例。
        ResourceHandler.asClass());
```

对于实体和物品堆，`EntityCapability` 和 `ItemCapability` 中分别存在类似的方法。

## 查询能力

一旦我们在静态字段中有了 `BlockCapability`、`EntityCapability` 或 `ItemCapability` 对象，我们就可以查询能力。

对于实体和物品堆，我们可以尝试使用 `getCapability` 查找能力的实现。如果结果为 `null`，则表示没有可用的实现。

例如：

``` java
var object = entity.getCapability(CAP, context);
if (object != null) {
    // 使用 object
}
```

``` java
var object = stack.getCapability(CAP, context);
if (object != null) {
    // 使用 object
}
```

方块能力的用法略有不同，因为没有方块实体的方块也可以具有能力。现在，查询在 `level` 上进行，并使用我们正在查找的 `pos` 作为额外参数：

``` java
var object = level.getCapability(CAP, pos, context);
if (object != null) {
    // 使用 object
}
```

如果方块实体和/或方块状态已知，可以传递它们以节省查询时间：

``` java
var object = level.getCapability(CAP, pos, blockState, blockEntity, context);
if (object != null) {
    // 使用 object
}
```

为了给出更具体的例子，以下是如何从 `Direction.NORTH` 侧面查询方块的 `ResourceHandler<ItemResource>` 能力：

``` java
ResourceHandler<ItemResource> handler = level.getCapability(Capabilities.Item.BLOCK, pos, Direction.NORTH);
if (handler != null) {
    // 使用该处理器进行一些与物品相关的操作。
}
```

## 方块能力缓存

当查找能力时，系统将在后台执行以下步骤：

1. 获取方块实体和方块状态（如果未提供）。
2. 获取已注册的能力提供者（更多信息见下文）。
3. 遍历提供者并询问它们是否能提供能力。
4. 其中一个提供者将返回一个能力实例，可能会分配一个新对象。

该实现相当高效，但对于频繁执行的查询（例如每个游戏刻一次），这些步骤可能会占用大量的服务器时间。`BlockCapabilityCache` 系统为在给定位置频繁查询的能力提供了显著的速度提升。

:::tip
通常，`BlockCapabilityCache` 将被创建一次，然后存储在执行频繁能力查询的对象的字段中。具体何时何地存储缓存由你决定。
:::

要创建缓存，请使用要查询的能力、世界、位置和查询上下文调用 `BlockCapabilityCache.create`。

``` java
// 声明字段：
private BlockCapabilityCache<ResourceHandler<ItemResource>, @Nullable Direction> capCache;

// 稍后，例如在方块实体的 `onLoad` 中：
this.capCache = BlockCapabilityCache.create(
    Capabilities.Item.BLOCK, // 要缓存的能力
    level, // 世界
    pos, // 目标位置
    Direction.NORTH // 上下文
);
```

然后使用 `getCapability()` 查询缓存：

``` java
ResourceHandler<ItemResource> handler = this.capCache.getCapability();
if (handler != null) {
    // 使用该处理器进行一些与物品相关的操作。
}
```

**缓存由垃圾收集器自动清除，无需注销。**

也可以接收能力对象更改时的通知！这包括能力更改（`oldHandler != newHandler`）、变为不可用（`null`）或再次变为可用（不再为 `null`）。

然后，需要使用两个额外参数创建缓存：

- 有效性检查，用于确定缓存是否仍然有效。
    - 在作为方块实体字段的最简单用法中，`() -> !this.isRemoved()` 即可。
- 失效监听器，在能力更改时调用。
    - 这是你可以对能力更改、移除或出现做出反应的地方。

``` java
// 在方块实体的 `onLoad` 中：
// 带有可选的失效监听器：
this.capCache = BlockCapabilityCache.create(
    Capabilities.Item.BLOCK, // 要缓存的能力
    level, // 世界
    pos, // 目标位置
    Direction.NORTH, // 上下文
    () -> !this.isRemoved(), // 有效性检查（因为缓存可能比它所属的对象寿命更长）
    () -> onCapInvalidate() // 失效监听器
);
```

## 方块能力失效

:::info
失效仅适用于方块能力。实体和物品堆能力无法被缓存，也不需要失效。
:::

为了确保缓存能够正确更新其存储的能力，**每当能力更改、出现或消失时，模组开发者必须调用 `level.invalidateCapabilities(pos)`**。

``` java
// 每当能力更改、出现或消失时：
level.invalidateCapabilities(pos);
```

NeoForge 已经处理了常见情况，如区块加载/卸载和方块实体创建/移除，但其他情况需要模组开发者显式处理。例如，模组开发者必须在以下情况下使能力失效：

- 如果先前返回的能力不再有效。
- 如果提供能力的方块（没有方块实体）被放置或状态更改，则通过重写 `onPlace`。
- 如果提供能力的方块（没有方块实体）被移除，则通过重写 `onRemove`。

有关纯方块的示例，请参阅 `ComposterBlock.java` 文件。

有关更多信息，请参阅 [`IBlockCapabilityProvider`][block-cap-provider] 的 javadoc。

## 注册能力

能力*提供者*是最终提供能力的东西。能力提供者是一个函数，它可以返回一个能力实例，如果它不能提供能力，则返回 `null`。提供者特定于：

- 它们为之提供的给定能力，以及
- 它们为之提供的方块实例、方块实体类型、实体类型或物品实例。

它们需要在 `RegisterCapabilitiesEvent` 中注册。

方块提供者使用 `registerBlock` 注册。例如：

``` java
@SubscribeEvent // 在模组事件总线上
public static void registerCapabilities(RegisterCapabilitiesEvent event) {
    event.registerBlock(
        Capabilities.Item.BLOCK, // 要为之注册的能力
        (level, pos, state, be, side) -> <返回 ResourceHandler<ItemResource>>,
        // 要为之注册的方块
        MY_ITEM_HANDLER_BLOCK,
        MY_OTHER_ITEM_HANDLER_BLOCK
    );
}
```

通常，注册将特定于某些方块实体类型，因此也提供了辅助方法 `registerBlockEntity`：

``` java
event.registerBlockEntity(
    Capabilities.Item.BLOCK, // 要为之注册的能力
    MY_BLOCK_ENTITY_TYPE, // 要为之注册的方块实体类型
    (myBlockEntity, side) -> myBlockEntity.myResourceHandlerForTheGivenSide
);
```

:::danger
如果方块或方块实体提供者先前返回的能力不再有效，*你必须通过调用 `level.invalidateCapabilities(pos)` 使缓存失效*。有关更多信息，请参阅上面的[失效部分][invalidation]。
:::

实体注册类似，使用 `registerEntity`：

``` java
event.registerEntity(
    Capabilities.Item.ENTITY, // 要为之注册的能力
    MY_ENTITY_TYPE, // 要为之注册的实体类型
    (myEntity, v) -> myEntity.myResourceHandlerForTheGivenContext
);
```

物品注册也类似。注意，提供者接收物品堆：

``` java
event.registerItem(
    Capabilities.Item.ITEM, // 要为之注册的能力
    (stack, itemAccess) -> <返回物品堆的 ResourceHandler<ItemResource>>,
    // 要为之注册的物品
    MY_ITEM,
    MY_OTHER_ITEM
);
```

## 为所有对象注册能力

如果由于某种原因你需要为所有方块、实体或物品注册提供者，你将需要遍历相应的注册表并为每个对象注册提供者。

例如，NeoForge 使用此系统为所有 `BucketItem`（不包括子类）注册流体资源处理器能力：

``` java
// 供参考，你可以在 `CapabilityHooks` 类中找到此代码。
for (Item item : BuiltInRegistries.ITEM) {
    if (item.getClass() == BucketItem.class) {
        event.registerItem(Capabilities.Fluid.ITEM, (stack, itemAccess) -> new BucketResourceHandler(itemAccess), item);
    }
}
```

提供者按照它们注册的顺序被询问能力。如果你希望在 NeoForge 已经为你的对象之一注册的提供者之前运行，请以更高的优先级注册你的 `RegisterCapabilitiesEvent` 处理器。

例如：

``` java
// 使用 HIGH 优先级在 NeoForge 之前注册！
@SubscribeEvent(priority = EventPriority.HIGH) // 在模组事件总线上
public static void registerCapabilities(RegisterCapabilitiesEvent event) {
    event.registerItem(
        Capabilities.Fluid.ITEM,
        (stack, itemAccess) -> new BucketResourceHandler(itemAccess),
        // 要为之注册的物品
        MY_CUSTOM_BUCKET
    );
}
```

有关 NeoForge 本身注册的提供者列表，请参阅 [`CapabilityHooks`][capability-hooks]。

[block-cap-provider]: https://github.com/neoforged/NeoForge/blob/1.21.x/src/main/java/net/neoforged/neoforge/capabilities/IBlockCapabilityProvider.java
[capability-hooks]: https://github.com/neoforged/NeoForge/blob/1.21.x/src/main/java/net/neoforged/neoforge/capabilities/CapabilityHooks.java
[invalidation]: #block-capability-invalidation