---
sidebar_position: 3
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 事件(Events)

NeoForge 的主要特性之一是其事件系统。游戏中发生的各种事情都会触发事件。例如，有玩家右键点击、玩家或其他实体跳跃、渲染方块、游戏加载等事件。模组开发者可以订阅这些事件的事件处理器，然后在这些事件处理器中执行他们想要的行为。

事件在各自的事件总线(Event Bus)上触发。最重要的总线是 `NeoForge.EVENT_BUS`，也称为**游戏**总线(Game Bus)。除此之外，在启动期间，每个加载的模组都会生成一个模组总线(Mod Bus)并传递给模组的构造函数。许多模组总线事件是并行触发的（与始终在同一线程上运行的主总线事件相反），这极大地提高了启动速度。更多信息请参见[下文][modbus]。

## 注册事件处理器

有多种方式可以注册事件处理器。所有这些方式的共同点是，每个事件处理器都是一个具有单个事件参数且无结果（即返回类型为`void`）的方法。

### `IEventBus#addListener`

注册方法处理器最简单的方式是注册它们的方法引用，如下所示：

```
@Mod("yourmodid")
public class YourMod {
    public YourMod(IEventBus modBus) {
        NeoForge.EVENT_BUS.addListener(YourMod::onLivingJump);
    }

    // Heals an entity by half a heart every time they jump.
    private static void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        // Only heal on the server side
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}
```

### `@SubscribeEvent`

或者，事件处理器可以通过注解驱动，方法是创建一个事件处理方法并用`@SubscribeEvent`注解它。然后，您可以将包含类的实例传递给事件总线，注册该实例所有带有`@SubscribeEvent`注解的事件处理器：

```
public class EventHandler {
    @SubscribeEvent
    public void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}

@Mod("yourmodid")
public class YourMod {
    public YourMod(IEventBus modBus) {
        NeoForge.EVENT_BUS.register(new EventHandler());
    }
}
```

您也可以静态地完成。只需将所有事件处理器设为静态，然后传递类本身而不是类实例：

```
public class EventHandler {
	@SubscribeEvent
    public static void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}

@Mod("yourmodid")
public class YourMod {
    public YourMod(IEventBus modBus) {
        NeoForge.EVENT_BUS.register(EventHandler.class);
    }
}
```

### `@EventBusSubscriber`

我们可以更进一步，也用`@EventBusSubscriber`注解事件处理器类。这个注解会被NeoForge自动发现，允许您从模组构造函数中移除所有与事件相关的代码。本质上，它等价于在模组构造函数末尾调用`NeoForge.EVENT_BUS.register(EventHandler.class)`和`modBus.register(EventHandler.class)`。这意味着所有处理器也必须是静态的。

虽然不是必须的，但强烈建议在注解中指定`modid`参数，以便于调试（尤其是在处理模组冲突时）。

```
@EventBusSubscriber(modid = "yourmodid")
public class EventHandler {
    @SubscribeEvent
    public static void onLivingJump(LivingEvent.LivingJumpEvent event) {
        LivingEntity entity = event.getEntity();
        if (!entity.level().isClientSide()) {
            entity.heal(1);
        }
    }
}
```

## 事件选项(Event Options)

### 字段和方法

字段和方法可能是事件中最明显的部分。大多数事件包含事件处理器要使用的上下文，例如触发事件的实体或事件发生所在的世界。

### 层次结构

为了利用继承的优势，一些事件不直接扩展`Event`，而是扩展它的一个子类，例如`BlockEvent`（包含与方块相关事件的方块上下文）或`EntityEvent`（类似地包含实体上下文）及其子类`LivingEvent`（用于`LivingEntity`特定的上下文）和`PlayerEvent`（用于`Player`特定的上下文）。这些提供上下文的父事件是`abstract`的，不能被监听。

:::danger
如果您监听一个`abstract`事件，您的游戏将崩溃，因为这绝不是您想要的。您应该始终监听其中一个子事件。
:::

``` mermaid
graph TD;
    Event-->BlockEvent;
    BlockEvent-->BlockDropsEvent;
    Event-->EntityEvent;
    EntityEvent-->LivingEvent;
    LivingEvent-->PlayerEvent;
    PlayerEvent-->CanPlayerSleepEvent;

    class Event,BlockEvent,EntityEvent,LivingEvent,PlayerEvent red;
    class BlockDropsEvent,CanPlayerSleepEvent blue;
```

### 可取消事件(Cancellable Events)

一些事件实现了`ICancellableEvent`接口。这些事件可以使用`#setCanceled(boolean canceled)`取消，并且可以使用`#isCanceled()`检查取消状态。如果一个事件被取消，此事件的其他事件处理器将不会运行，并且会启用某种与“取消”相关的行为。例如，取消`LivingChangeTargetEvent`将阻止实体的目标实体改变。

事件处理器可以选择显式接收已取消的事件。这是通过将`IEventBus#addListener`（或`@SubscribeEvent`，取决于您附加事件处理器的方式）中的`receiveCanceled`布尔参数设置为`true`来完成的。

### 三态和结果(TriStates and Results)

一些事件具有三个潜在的返回状态，由`TriState`表示，或者直接在事件类上使用`Result`枚举。返回状态通常可以取消事件处理的操作（`TriState#FALSE`）、强制操作运行（`TriState#TRUE`）或执行原版默认行为（`TriState#DEFAULT`）。

具有三个潜在返回状态的事件具有一些`set*`方法来设置所需的结果。

```
// In some event handler class

@SubscribeEvent // on the game event bus
public static void renderNameTag(RenderNameTagEvent.CanRender event) {
    // Uses TriState to set the return state
    event.setCanRender(TriState.FALSE);
}

@SubscribeEvent // on the game event bus
public static void mobDespawn(MobDespawnEvent event) {
    // Uses a Result enum to set the return state
    event.setResult(MobDespawnEvent.Result.DENY);
}
```

### 优先级(Priority)

事件处理器可以选择性地分配优先级。`EventPriority`枚举包含五个值：`HIGHEST`、`HIGH`、`NORMAL`（默认）、`LOW`和`LOWEST`。事件处理器按优先级从高到低执行。如果它们具有相同的优先级，它们在主总线上按注册顺序触发（大致与模组加载顺序相关），在模组总线上按精确的模组加载顺序触发（见下文）。

可以通过在`IEventBus#addListener`或`@SubscribeEvent`中设置`priority`参数来定义优先级，具体取决于您附加事件处理器的方式。请注意，对于并行触发的事件，优先级将被忽略。

### 单端事件(Sided Events)

一些事件只在一个[端][side]触发。常见的例子包括各种渲染事件，它们只在客户端触发。由于仅客户端的事件通常需要访问Minecraft代码库的其他仅客户端部分，因此需要相应地注册它们。

使用`IEventBus#addListener`的事件处理器应通过`FMLEnvironment#getDist`或主模组构造函数中的`Dist`参数检查当前物理端，并在单独的仅客户端类中添加监听器，如关于[端][side]的文章所述。

使用`@EventBusSubscriber`的事件处理器可以在注解的`value`参数中指定端，例如`@EventBusSubscriber(value = Dist.CLIENT, modid = "yourmodid")`。

## 事件总线(Event Buses)

虽然大多数事件发布在`NeoForge.EVENT_BUS`上，但有些事件发布在模组事件总线上。这些通常称为模组总线事件。模组总线事件可以通过它们的超接口`IModBusEvent`与常规事件区分开来。

模组事件总线作为参数传递给您在模组构造函数中，然后您可以向它订阅模组总线事件。如果您使用`@EventBusSubscriber`，事件将自动订阅到正确的总线。

### 模组生命周期

大多数模组总线事件是所谓的生命周期事件。生命周期事件在每个模组的启动过程中运行一次。其中许多事件通过子类化`ParallelDispatchEvent`并行触发；如果您想在主线程上运行这些事件之一的代码，请使用`#enqueueWork(Runnable runnable)`将它们加入队列。

生命周期通常遵循以下顺序：

- 模组构造函数被调用。在此处或下一步注册您的事件处理器。
- 调用所有`@EventBusSubscriber`。
- 触发`FMLConstructModEvent`。
- 触发注册表事件，包括[`NewRegistryEvent`][newregistry]、[`DataPackRegistryEvent.NewRegistry`][newdatapackregistry]以及每个注册表的[`RegisterEvent`][registerevent]。
- 触发`FMLCommonSetupEvent`。这里进行各种杂项设置。
- 触发[端][side]设置：在物理客户端上触发`FMLClientSetupEvent`，在物理服务器上触发`FMLDedicatedServerSetupEvent`。
- 处理`InterModComms`（见下文）。
- 触发`FMLLoadCompleteEvent`。

#### `InterModComms`

`InterModComms`是一个允许模组开发者向其他模组发送消息以实现兼容性功能的系统。该类保存了给模组的消息，所有方法调用都是线程安全的。该系统主要由两个事件驱动：`InterModEnqueueEvent`和`InterModProcessEvent`。

在`InterModEnqueueEvent`期间，您可以使用`InterModComms#sendTo`向其他模组发送消息。这些方法接受要发送消息的模组ID、与消息数据关联的键（用于区分不同消息）以及一个持有消息数据的`Supplier`。也可以选择指定发送者。

然后，在`InterModProcessEvent`期间，您可以使用`InterModComms#getMessages`获取所有接收到的消息流，作为`IMCMessage`对象。这些对象保存数据的发送者、数据的预期接收者、数据键以及实际数据的提供者。

### 其他模组总线事件

除了生命周期事件外，还有一些杂项事件在模组事件总线上触发，主要是出于遗留原因。这些通常是您可以注册、设置或初始化各种东西的事件。与生命周期事件相反，这些事件大多数不是并行运行的。一些例子：

- `RegisterColorHandlersEvent.Block`、`.ItemTintSources`、`.ColorResolvers`
- `ModelEvent.BakingCompleted`
- `TextureAtlasStitchedEvent`

:::warning
这些事件大多数计划在未来版本中移动到游戏事件总线。
:::

[modbus]: #event-buses
[newdatapackregistry]: registries.md#custom-datapack-registries
[newregistry]: registries.md#custom-registries
[registerevent]: registries.md#registerevent
[side]: sides.md