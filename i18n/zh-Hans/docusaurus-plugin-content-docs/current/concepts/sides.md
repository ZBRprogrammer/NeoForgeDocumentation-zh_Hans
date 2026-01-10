---
sidebar_position: 2
---
# 端(Sides)

像许多其他程序一样，Minecraft遵循客户端-服务器(client-server)概念，其中客户端负责显示数据，而服务器负责更新数据。当使用这些术语时，我们有一个相当直观的理解……对吗？

事实证明，并非如此。很多混淆源于Minecraft根据上下文有两种不同的端概念：物理端(physical side)和逻辑端(logical side)。

## 逻辑端(Logical Side)与物理端(Physical Side)

### 物理端(Physical Side)

当你打开Minecraft启动器，选择一个Minecraft安装并点击播放时，你启动了一个**物理客户端(physical client)**。这里使用“物理”一词的意思是“这是一个客户端程序”。这尤其意味着客户端侧功能，例如所有渲染内容，在这里可用并可以根据需要使用。相比之下，**物理服务器(physical server)**，也称为专用服务器(dedicated server)，是当你启动Minecraft服务器JAR时打开的。虽然Minecraft服务器带有一个基本的GUI，但它缺少所有仅客户端的功能。最值得注意的是，这意味着服务器JAR中缺少各种客户端类。在物理服务器上调用这些类将导致缺少类错误，即崩溃，因此我们需要防范这一点。

### 逻辑端(Logical Side)

逻辑端主要关注Minecraft的内部程序结构。**逻辑服务器(logical server)**是游戏逻辑运行的地方。诸如时间和天气变化、实体刻(ticks)、实体生成(spawning)等都在服务器上运行。各种数据，如库存内容，也是服务器的责任。另一方面，**逻辑客户端(logical client)**负责显示所有要显示的内容。Minecraft将所有客户端代码保存在一个独立的`net.minecraft.client`包中，并在一个称为渲染线程(Render Thread)的单独线程中运行它，而其他一切都被视为公共（即客户端和服务器）代码。

### 有什么区别？

物理端和逻辑端之间的区别最好通过两种场景来体现：

- 玩家加入**多人游戏(multiplayer)**世界。这相当直接：玩家的物理（和逻辑）客户端连接到某个地方的物理（和逻辑）服务器——玩家不关心在哪里；只要他们能连接，这就是客户端所知道的全部，也是客户端需要知道的全部。
- 玩家加入**单人游戏(singleplayer)**世界。这就是事情变得有趣的地方。玩家的物理客户端启动一个逻辑服务器，然后以逻辑客户端的角色连接到同一台机器上的逻辑服务器。如果你熟悉网络，可以将其视为连接到`localhost`（仅概念上；不涉及实际的套接字或类似内容）。

这两种场景也显示了主要问题：如果一个逻辑服务器可以与你的代码一起工作，仅这并不能保证物理服务器也能正常工作。这就是为什么你应该始终使用专用服务器进行测试以检查意外行为。由于客户端和服务器分离不正确而导致的`NoClassDefFoundError`和`ClassNotFoundException`是模组开发中最常见的错误之一。另一个常见错误是使用静态字段并从两个逻辑端访问它们；这尤其棘手，因为通常没有迹象表明有问题。

:::tip
如果你需要将数据从一端传输到另一端，必须[发送数据包(packet)][networking]。
:::

在NeoForge代码库中，物理端由一个称为`Dist`的枚举表示，而逻辑端由一个称为`LogicalSide`的枚举表示。

:::info
历史上，服务器JAR拥有客户端没有的类。在现代版本中已不是这种情况；物理服务器是物理客户端的一个子集，如果你愿意的话。
:::

## 执行端特定操作

### `Level#isClientSide()`

这个布尔检查将是你检查端的最常用方式。在`Level`对象上查询此字段确定了该级别所属的**逻辑**端：如果此字段为`true`，则该级别在逻辑客户端上运行。如果字段为`false`，则该级别在逻辑服务器上运行。因此，物理服务器将始终在此字段中包含`false`，但我们不能假设`false`意味着物理服务器，因为此字段对于物理客户端内的逻辑服务器（即单人游戏世界）也可能为`false`。

每当你需要确定是否应运行游戏逻辑和其他机制时，请使用此检查。例如，如果你希望玩家每次点击你的方块时都受到伤害，或让你的机器将泥土加工成钻石，你应该在确保`#isClientSide()`为`false`后才这样做。将游戏逻辑应用于逻辑客户端可能导致最佳情况下的不同步（幽灵实体、不同步的统计信息等），最坏情况下导致崩溃。

:::tip
此检查应作为你的默认首选。只要你有`Level`可用，就使用此检查。
:::

### `FMLEnvironment#getDist()`

`FMLEnvironment#getDist()`是`Level#isClientSide()`检查的**物理**对应物。如果此字段是`Dist.CLIENT`，你就在物理客户端上。如果字段是`Dist.DEDICATED_SERVER`，你就在物理服务器上。

#### `@Mod`

在处理仅客户端类时，检查物理环境很重要。推荐的方法是通过指定一个单独的[`@Mod`注解][mod]，将`dist`参数设置为模组类应加载的物理端：

``` java
@Mod("examplemod")
public class ExampleMod {
    public ExampleMod(IEventBus modBus) {
        // 执行应在两侧执行的逻辑
    }
}

@Mod(value = "examplemod", dist = Dist.CLIENT) 
public class ExampleModClient {
    public ExampleModClient(IEventBus modBus) {
        // 执行应仅在物理客户端上执行的逻辑
    }
}

@Mod(value = "examplemod", dist = Dist.DEDICATED_SERVER) 
public class ExampleModDedicatedServer {
    public ExampleModDedicatedServer(IEventBus modBus) {
        // 执行应仅在物理服务器上执行的逻辑
    }
}
```

:::tip
模组通常期望在任一端工作。这尤其意味着，如果你开发一个仅客户端模组，你应该验证模组实际上在物理客户端上运行，并在不运行时无操作(no-op)。
:::

[networking]: ../networking/index.md
[mod]: ../gettingstarted/modfiles.md#javafml-and-mod