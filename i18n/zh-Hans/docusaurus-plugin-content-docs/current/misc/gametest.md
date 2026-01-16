import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 游戏测试(Game Tests)

游戏测试(Game Tests)是一种运行游戏内单元测试的方法。该系统设计为可扩展且可并行运行，以高效执行大量不同的测试。测试对象交互和行为仅仅是此框架众多应用中的一部分。由于该系统既可以完全通过代码实现，也可以通过[数据包(datapacks)]实现，下文将展示两种方式。

## 创建游戏测试(Game Test)

一个标准的游戏测试(Game Test)遵循四个基本步骤：

1.  加载一个结构(Structure)或模板(Template)，其中包含用于测试交互或行为的场景。
2.  创建一个供测试运行的环境。
3.  注册一个函数来运行逻辑。如果达到成功状态，则测试成功。否则，测试失败，结果存储在场景相邻的讲台(Lectern)内。
4.  创建一个测试实例(Test Instance)将上述三个对象链接在一起。

## 测试数据(Test Data)

所有测试实例都持有一些 `TestData`，它定义了游戏测试(Game Test)应如何运行，从其初始配置到要使用的环境和结构模板(Structure Template)。由于 `TestData` 被序列化为 `MapCodec`，因此数据存储在文件的根层级，与其他所有实例特定参数一起。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个游戏测试 examplemod:example_test
// 位于 'data/examplemod/test_instance/example_test.json'
{
    // `TestData`

    // 运行测试的环境
    // 指向 'data/examplemod/test_environment/example_environment.json'
    "environment": "examplemod:example_environment",

    // 用于游戏测试的结构(Structure)
    // 指向 'data/examplemod/structure/example_structure.nbt'
    "structure": "examplemod:example_structure",

    // 游戏测试(Game Test)运行直到自动失败的最大游戏刻(Tick)数
    "max_ticks": 400,

    // 用于设置游戏测试(Game Test)所需一切的游戏刻(Tick)数
    // 此计数不计入测试可以占用的最大游戏刻(Tick)数
    // 如果未指定，默认为 0
    "setup_ticks": 50,

    // 测试是否必须成功才能将批次运行标记为成功
    // 如果未指定，默认为 true
    "required": true,

    // 指定测试中结构和所有后续辅助方法应如何旋转
    // 如果未指定，则不旋转
    // 可选值：'none', 'clockwise_90', '180', 'counterclockwise_90'
    "rotation": "clockwise_90",

    // 为 true 时，测试只能通过 `/test` 命令运行
    // 如果未指定，默认为 false
    "manual_only": true,

    // 指定测试可以重新运行的最大次数
    // 如果未指定，默认为 1
    "max_attempts": 3,

    // 指定测试必须发生的最小成功次数才能被标记为成功
    // 此值必须小于或等于允许的最大尝试次数
    // 如果未指定，默认为 1
    "required_successes": 1,

    // 返回结构边界是否应保持顶部为空
    // 目前仅用于基于方块的测试实例(Block-Based Test Instance)
    // 如果未指定，默认为 false
    "sky_access": false

    // ...
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_INSTANCE, bootstrap -> {
            // 使用此方法获取测试环境(Test Environment)
            HolderGetter<TestEnvironmentDefinition> environments = bootstrap.lookup(Registries.TEST_ENVIRONMENT);

            // 注册一个游戏测试(Game Test)
            // 任何与测试数据(Test Data)无关的字段都被隐藏
            bootstrap.register(..., new FunctionGameTestInstance(...,
                new TestData<>(
                    // 运行测试的环境
                    // 指向 'data/examplemod/test_environment/example_environment.json'
                    environments.getOrThrow(EXAMPLE_ENVIRONMENT),

                    // 用于游戏测试的结构(Structure)
                    // 指向 'data/examplemod/structure/example_structure.nbt'
                    ResourceLocation.fromNamespaceAndPath("examplemod", "example_structure"),

                    // 游戏测试(Game Test)运行直到自动失败的最大游戏刻(Tick)数
                    400,

                    // 用于设置游戏测试(Game Test)所需一切的游戏刻(Tick)数
                    // 此计数不计入测试可以占用的最大游戏刻(Tick)数
                    // 如果未指定，默认为 0
                    50,

                    // 测试是否必须成功才能将批次运行标记为成功
                    // 如果未指定，默认为 true
                    true,

                    // 指定测试中结构和所有后续辅助方法应如何旋转
                    // 如果未指定，则不旋转
                    // 可选值：'none', 'clockwise_90', '180', 'counterclockwise_90'
                    Rotation.CLOCKWISE_90,

                    // 为 true 时，测试只能通过 `/test` 命令运行
                    // 如果未指定，默认为 false
                    true,

                    // 指定测试可以重新运行的最大次数
                    // 如果未指定，默认为 1
                    3,

                    // 指定测试必须发生的最小成功次数才能被标记为成功
                    // 此值必须小于或等于允许的最大尝试次数
                    // 如果未指定，默认为 1
                    1,

                    // 返回结构边界是否应保持顶部为空
                    // 目前仅用于基于方块的测试实例(Block-Based Test Instance)
                    // 如果未指定，默认为 false
                    false
                )
            ));
        })
    );
}
```

</TabItem>
</Tabs>

## 结构模板(Structure Templates)

游戏测试(Game Tests)在由结构(Structure)或模板(Template)加载的场景内执行。所有模板都定义了场景的尺寸以及将要加载的初始数据（方块和实体）。模板必须以 `.nbt` 文件形式存储在 `data/<命名空间>/structure` 目录下。`TestData` 中的 `structure` 字段使用相对的 `ResourceLocation` 引用 NBT 文件（例如，`examplemod:example_structure` 指向 `data/examplemod/structure/example_structure.nbt`）。

## 测试环境(Test Environments)

所有游戏测试(Game Tests)都在某个 `TestEnvironmentDefinition` 中运行，它决定了应如何设置当前的 `ServerLevel`。然后，一旦测试完成，环境将被拆除，让下一个或下一批实例运行。所有环境都是批处理的，这意味着如果多个测试实例具有相同的环境，它们将同时运行。所有测试环境(Test Environment)都位于 `data/<命名空间>/test_environment/<路径>.json`。

原版提供了 `minecraft:default`，它不修改 `ServerLevel`。但是，还有其他支持的定义类型可用于构建环境。

### 游戏规则(Game Rules)

此环境类型设置用于测试的游戏规则(Game Rules)。在拆除(Teardown)期间，游戏规则(Game Rules)将重置为其默认值。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// examplemod:example_environment
// 位于 'data/examplemod/test_environment/example_environment.json'
{
    "type": "minecraft:game_rules",

    // 要设置的布尔值游戏规则(Game Rules)列表
    "bool_rules": [
        {
            // 规则名称
            "rule": "doFireTick",
            "value": false
        }
        // ...
    ],

    // 要设置的整数值游戏规则(Game Rules)列表
    "int_rules": [
        {
            "rule": "playersSleepingPercentage",
            "value": 50
        }
        // ...
    ]
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_ENVIRONMENT, bootstrap -> {

            // 注册环境
            bootstrap.register(
                EXAMPLE_ENVIRONMENT,
                new TestEnvironmentDefinition.SetGameRules(
                    // 要设置的布尔值游戏规则(Game Rules)列表
                    List.of(
                        new TestEnvironmentDefinition.SetGameRules.Entry(
                            // 游戏规则(Game Rule)
                            GameRules.RULE_DOFIRETICK,
                            GameRules.BooleanValue.create(false)
                        )
                        // ...
                    ),
                    // 要设置的整数值游戏规则(Game Rules)列表
                    List.of(
                        new TestEnvironmentDefinition.SetGameRules.Entry(
                            // 游戏规则(Game Rule)
                            GameRules.RULE_PLAYERS_SLEEPING_PERCENTAGE,
                            GameRules.IntegerValue.create(50)
                        )
                        // ...
                    )
                )
            );
        })
    );
}
```

</TabItem>
</Tabs>

### 时间(Time of Day)

此环境类型将时间设置为某个非负整数，类似于如何使用 `/time set <数字>` 命令。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// examplemod:example_environment
// 位于 'data/examplemod/test_environment/example_environment.json'
{
    "type": "minecraft:time_of_day",

    // 设置世界中的时间
    // 常用值：
    // - 白天(Day)      -> 1000
    // - 正午(Noon)     -> 6000
    // - 夜晚(Night)    -> 13000
    // - 午夜(Midnight) -> 18000
    "time": 13000
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_ENVIRONMENT, bootstrap -> {

            // 注册环境
            bootstrap.register(
                EXAMPLE_ENVIRONMENT,
                new TestEnvironmentDefinition.TimeOfDay(
                    // 设置世界中的时间
                    // 常用值：
                    // - 白天(Day)      -> 1000
                    // - 正午(Noon)     -> 6000
                    // - 夜晚(Night)    -> 13000
                    // - 午夜(Midnight) -> 18000
                    13000
                )
            );
        })
    );
}
```

</TabItem>
</Tabs>

### 天气(Weather)

此环境类型设置天气，类似于如何使用 `/weather` 命令。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// examplemod:example_environment
// 位于 'data/examplemod/test_environment/example_environment.json'
{
    "type": "minecraft:weather",

    // 可以是以下三个值之一：
    // - clear   （无天气）
    // - rain    （下雨）
    // - thunder （下雨和打雷）
    "weather": "thunder"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_ENVIRONMENT, bootstrap -> {

            // 注册环境
            bootstrap.register(
                EXAMPLE_ENVIRONMENT,
                new TestEnvironmentDefinition.Weather(
                    // 可以是以下三个值之一：
                    // - clear   （无天气）
                    // - rain    （下雨）
                    // - thunder （下雨和打雷）
                    TestEnvironmentDefinition.Weather.Type.THUNDER
                )
            );
        })
    );
}
```

</TabItem>
</Tabs>

### Minecraft 函数(Minecraft Functions)

此环境类型提供两个指向 `mcfunction` 的 `ResourceLocation`，分别用于设置(Setup)和拆除(Teardown)世界(Level)。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// examplemod:example_environment
// 位于 'data/examplemod/test_environment/example_environment.json'
{
    "type": "minecraft:function",

    // 要使用的设置(Setup) mcfunction
    // 如果未指定，则不运行任何内容
    // 指向 'data/examplemod/function/example/setup.mcfunction'
    "setup": "examplemod:example/setup",

    // 要使用的拆除(Teardown) mcfunction
    // 如果未指定，则不运行任何内容
    // 指向 'data/examplemod/function/example/teardown.mcfunction'
    "teardown": "examplemod:example/teardown"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_ENVIRONMENT, bootstrap -> {

            // 注册环境
            bootstrap.register(
                EXAMPLE_ENVIRONMENT,
                new TestEnvironmentDefinition.Functions(
                    // 要使用的设置(Setup) mcfunction
                    // 如果未指定，则不运行任何内容
                    // 指向 'data/examplemod/function/example/setup.mcfunction'
                    Optional.of(ResourceLocation.fromNamespaceAndPath("examplemod", "example/setup")),

                    // 要使用的拆除(Teardown) mcfunction
                    // 如果未指定，则不运行任何内容
                    // 指向 'data/examplemod/function/example/teardown.mcfunction'
                    Optional.of(ResourceLocation.fromNamespaceAndPath("examplemod", "example/teardown"))
                )
            );
        })
    );
}
```

</TabItem>
</Tabs>

### 复合环境(Composites)

可以使用复合环境类型合并多个环境。定义列表可以接受对现有定义的引用，也可以是内联定义。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// examplemod:example_environment
// 位于 'data/examplemod/test_environment/example_environment.json'
{
    "type": "minecraft:all_of",

    // 要使用的测试环境(Test Environment)列表
    // 可以指定注册表名称或环境本身
    "definitions": [
        // 指向 'data/minecraft/test_environment/default.json'
        "minecraft:default",
        {
            // 原始环境定义
            "type": "..."
        }
        // ...
    ]
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_ENVIRONMENT, bootstrap -> {
            // 获取现有环境
            HolderGetter<TestEnvironmentDefinition> environments = bootstrap.lookup(Registries.TEST_ENVIRONMENT);

            // 注册环境
            bootstrap.register(
                EXAMPLE_ENVIRONMENT,
                new TestEnvironmentDefinition.AllOf(
                    List.of(
                        // 指向 'data/minecraft/test_environment/default.json'
                        environments.getOrThrow(GameTestEnvironments.DEFAULT_KEY),
                        Holder.direct(
                            // 在此处创建一个新的 TestEnvironmentDefinition
                            ...
                        )
                        // ...
                    )
                )
            );
        })
    );
}
```

</TabItem>
</Tabs>

### 自定义定义类型(Custom Definition Types)

自定义 `TestEnvironmentDefinition` 类型提供三个方法：`setup` 用于修改 `ServerLevel`，`teardown` 用于重置修改的内容，`codec` 用于提供用于编码和解码类型的 `MapCodec`：

```java
public record ExampleEnvironmentType(int value1, boolean value2) implements TestEnvironmentDefinition {

    // 构造要注册的映射编解码器(Map Codec)
    public static final MapCodec<ExampleEnvironmentType> CODEC = RecordCodecBuilder.mapCodec(instance -> instance.group(
            Codec.INT.fieldOf("value1").forGetter(ExampleEnvironmentType::value1),
            Codec.BOOL.fieldOf("value2").forGetter(ExampleEnvironmentType::value2)
        ).apply(instance, ExampleEnvironmentType::new)
    );

    @Override
    public void setup(ServerLevel level) {
        // 在此处进行必要的设置
    }

    @Override
    public void teardown(ServerLevel level) {
        // 撤消在 setup 方法中所做的任何更改
        // 这应该恢复到默认值或先前的值
    }

    @Override
    public MapCodec<ExampleEnvironmentType> codec() {
        return EXAMPLE_ENVIRONMENT_CODEC.get();
    }
}
```

然后，`MapCodec` 可以被[注册(registered)]：

```java
public static final DeferredRegister<MapCodec<? extends TestEnvironmentDefinition>> TEST_ENVIRONMENT_DEFINITION_TYPES = DeferredRegister.create(
        BuiltInRegistries.TEST_ENVIRONMENT_DEFINITION_TYPE,
        "examplemod"
);

public static final Supplier<MapCodec<ExampleEnvironmentType>> EXAMPLE_ENVIRONMENT_CODEC = TEST_ENVIRONMENT_DEFINITION_TYPES.register(
    "example_environment_type",
    () -> RecordCodecBuilder.mapCodec(instance -> instance.group(
            Codec.INT.fieldOf("value1").forGetter(ExampleEnvironmentType::value1),
            Codec.BOOL.fieldOf("value2").forGetter(ExampleEnvironmentType::value2)
        ).apply(instance, ExampleEnvironmentType::new)
    )
);
```

最后，该类型就可以在您的环境定义中使用了：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// examplemod:example_environment
// 位于 'data/examplemod/test_environment/example_environment.json'
{
    "type": "examplemod:example_environment_type",

    "value1": 0,
    "value2": true
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_ENVIRONMENT, bootstrap -> {

            // 注册环境
            bootstrap.register(
                EXAMPLE_ENVIRONMENT,
                new ExampleEnvironmentType(
                    0, true
                )
            );
        })
    );
}
```

</TabItem>
</Tabs>

## 测试函数(The Test Function)

游戏测试(Game Tests)的基本概念围绕运行某个接收 `GameTestHelper` 参数且不返回任何内容的方法构建。调用 `GameTestHelper` 内部的方法决定测试成功还是失败。每个测试函数都被[注册(registered)]，允许在测试实例(Test Instance)中引用：

```java
public class ExampleFunctions {

    // 这是我们的示例函数
    public static void exampleTest(GameTestHelper helper) {
        // 执行操作
    }
}

// 注册我们的函数以供使用
public static final DeferredRegister<Consumer<GameTestHelper>> TEST_FUNCTION = DeferredRegister.create(
        BuiltInRegistries.TEST_FUNCTION,
        "examplemod"
);

public static final DeferredHolder<Consumer<GameTestHelper>, Consumer<GameTestHelper>> EXAMPLE_FUNCTION = TEST_FUNCTION.register(
    "example_function",
    () -> ExampleFunctions::exampleTest
);
```

### 相对定位(Relative Positioning)

所有测试函数都使用结构方块(Structure Block)的当前位置，将结构模板(Structure Template)场景内的相对坐标转换为绝对坐标。为了便于在相对坐标和绝对坐标之间转换，可以分别使用 `GameTestHelper.absolutePos` 和 `GameTestHelper.relativePos`。

可以通过[测试命令(test command)][test]加载结构来获取结构模板的相对位置，将玩家置于所需位置，最后运行 `/test pos` 命令。这将获取玩家相对于玩家 200 格范围内最近结构的坐标。该命令会将相对位置作为可复制的文本组件导出到聊天中，用作最终的局部变量。

:::提示
`/test pos` 生成的局部变量可以通过在命令末尾附加引用来指定其引用名称：

```bash
/test pos <变量名> # 导出 'final BlockPos <变量名> = new BlockPos(...);'
```
:::

### 成功完成(Successful Completion)

测试函数负责一件事：在有效完成时标记测试成功。如果在达到超时（由 `TestData.maxTicks` 定义）之前未达到成功状态，则测试自动失败。

`GameTestHelper` 中有许多抽象方法可用于定义成功状态；然而，有四个方法极为重要，需要了解。

| 方法                  | 描述                                                                                                                              |
| :-------------------- | :-------------------------------------------------------------------------------------------------------------------------------- |
| `succeed`             | 测试被标记为成功。                                                                                                                |
| `succeedIf`           | 立即测试提供的 `Runnable`，如果不抛出 `GameTestAssertException` 则成功。如果测试没有在立即刻(Tick)成功，则标记为失败。             |
| `succeedWhen`         | 每个游戏刻(Tick)测试提供的 `Runnable` 直到超时，如果其中一个游戏刻(Tick)的检查没有抛出 `GameTestAssertException` 则成功。         |
| `succeedOnTickWhen`   | 在指定的游戏刻(Tick)测试提供的 `Runnable`，如果不抛出 `GameTestAssertException` 则成功。如果 `Runnable` 在任何其他游戏刻(Tick)成功，则标记为失败。 |

:::警告
游戏测试(Game Tests)每个游戏刻(Tick)执行，直到测试被标记为成功。因此，安排在给定游戏刻(Tick)成功的方法必须注意在此前的任何游戏刻(Tick)都失败。
:::

### 调度操作(Scheduling Actions)

并非所有操作都会在测试开始时发生。操作可以安排在特定时间或间隔执行：

| 方法             | 描述                           |
| :--------------- | :----------------------------- |
| `runAtTickTime`  | 在指定的游戏刻(Tick)执行操作。 |
| `runAfterDelay`  | 在当前游戏刻(Tick)之后 `x` 个游戏刻(Tick)执行操作。 |
| `onEachTick`     | 每个游戏刻(Tick)执行操作。     |

### 断言(Assertions)

在游戏测试(Game Test)期间的任何时刻，都可以进行断言以检查给定条件是否为真。`GameTestHelper` 中有许多断言方法；然而，简而言之，每当不满足适当状态时就会抛出 `GameTestAssertException`。

## 注册测试实例(Registering The Test Instance)

有了 `TestData`、`TestEnvironmentDefinition` 和测试函数，我们现在可以通过 `GameTestInstance` 将所有内容链接在一起。每个测试实例代表一个要运行的单个游戏测试(Game Test)。所有测试实例都位于 `data/<命名空间>/test_instance/<路径>.json`。

### 基于函数的测试(Function-Based Tests)

`FunctionGameTestInstance` 将 `TestData` 链接到某个已注册的测试函数。测试实例将在被调用时运行该测试函数。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个游戏测试 examplemod:example_test
// 位于 'data/examplemod/test_instance/example_test.json'
{
    // `TestData`

    "environment": "examplemod:example_environment",
    "structure": "examplemod:example_structure",
    "max_ticks": 400,
    "setup_ticks": 50,
    "required": true,
    "rotation": "clockwise_90",
    "manual_only": true,
    "max_attempts": 3,
    "required_successes": 1,
    "sky_access": false,

    // `FunctionGameTestInstance`
    "type": "minecraft:function",

    // 指向测试函数注册表中的 'Consumer<GameTestHelper>'
    "function": "examplemod:example_function"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 测试实例键
public static final ResourceKey<GameTestInstance> EXAMPLE_TEST_INSTANCE = ResourceKey.create(
    Registries.TEST_INSTANCE,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_test")
);

// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_INSTANCE, bootstrap -> {
            // 使用此方法获取测试环境(Test Environment)
            HolderGetter<TestEnvironmentDefinition> environments = bootstrap.lookup(Registries.TEST_ENVIRONMENT);

            // 注册一个游戏测试(Game Test)
            // 任何与测试数据(Test Data)无关的字段都被隐藏
            bootstrap.register(EXAMPLE_TEST_INSTANCE,
                new FunctionGameTestInstance(
                    // 指向测试函数注册表中的 'Consumer<GameTestHelper>'
                    EXAMPLE_FUNCTION.getKey()
                    new TestData<>(
                        environments.getOrThrow(EXAMPLE_ENVIRONMENT),
                        ResourceLocation.fromNamespaceAndPath("examplemod", "example_structure"),
                        400,
                        50,
                        true,
                        Rotation.CLOCKWISE_90,
                        true,
                        3,
                        1,
                        false
                    )
            ));
        })
    );
}
```

</TabItem>
</Tabs>

### 基于方块的测试(Block-Based Tests)

`BlockBasedTestInstance` 是一种特殊的测试实例，它依赖于由 `Blocks.TEST_BLOCK` 发送和接收的红石信号。为此测试正常工作，结构模板必须至少包含两个测试方块(Test Block)：一个且仅一个设置为 `TestBlockMode.START`，另一个设置为 `TestBlockMode.ACCEPT`。测试开始时，启动测试方块(Starting Test Block)被触发，发送一个持续一刻的15级信号脉冲。预期此信号最终会触发其他处于 `LOG`、`FAIL` 或 `ACCEPT` 状态的测试方块。`LOG` 测试方块在激活时也会发送一个15级信号脉冲。`ACCEPT` 和 `FAIL` 测试方块分别导致测试实例成功或失败。在给定的游戏刻(Tick)上，`ACCEPT` 总是优先于 `FAIL`。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个游戏测试 examplemod:example_test
// 位于 'data/examplemod/test_instance/example_test.json'
{
    // `TestData`

    "environment": "examplemod:example_environment",
    "structure": "examplemod:example_structure",
    "max_ticks": 400,
    "setup_ticks": 50,
    "required": true,
    "rotation": "clockwise_90",
    "manual_only": true,
    "max_attempts": 3,
    "required_successes": 1,
    "sky_access": false,

    // `BlockBasedTestInstance`
    "type": "minecraft:block_based"
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 测试实例键
public static final ResourceKey<GameTestInstance> EXAMPLE_TEST_INSTANCE = ResourceKey.create(
    Registries.TEST_INSTANCE,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_test")
);

// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_INSTANCE, bootstrap -> {
            // 使用此方法获取测试环境(Test Environment)
            HolderGetter<TestEnvironmentDefinition> environments = bootstrap.lookup(Registries.TEST_ENVIRONMENT);

            // 注册一个游戏测试(Game Test)
            // 任何与测试数据(Test Data)无关的字段都被隐藏
            bootstrap.register(EXAMPLE_TEST_INSTANCE,
                new BlockBasedTestInstance(
                    new TestData<>(
                        environments.getOrThrow(EXAMPLE_ENVIRONMENT),
                        ResourceLocation.fromNamespaceAndPath("examplemod", "example_structure"),
                        400,
                        50,
                        true,
                        Rotation.CLOCKWISE_90,
                        true,
                        3,
                        1,
                        false
                    )
            ));
        })
    );
}
```

</TabItem>
</Tabs>

### 自定义测试实例(Custom Test Instances)

如果出于某种原因需要实现自己的基于测试的逻辑，可以扩展 `GameTestInstance`。必须实现两个方法：`run`，代表测试函数；以及 `typeDescription`，提供测试实例的描述。如果测试实例要用于数据生成(Datagen)，它必须有一个 `MapCodec` 才能被[注册(registered)]。

```java
public class ExampleTestInstance extends GameTestInstance {

    public ExampleTestInstance(int value1, boolean value2, TestData<Holder<TestEnvironmentDefinition>> info) {
        super(info);
    }

    @Override
    public void run(GameTestHelper helper) {
        // 运行您想要的任何游戏测试命令
        helper.assertBlockPresent(...);

        // 确保您有某种成功的方式
        helper.succeedIf(() -> ...);
    }

    @Override
    public MapCodec<ExampleTestInstance> codec() {
        return EXAMPLE_INSTANCE_CODEC.get();
    }

    @Override
    protected MutableComponent typeDescription() {
        // 提供有关此测试应该是什么的描述
        // 应使用可翻译组件(Translatable Component)
        return Component.literal("示例测试实例(Example Test Instance)");
    }
}

// 注册我们的测试实例以供使用
public static final DeferredRegister<MapCodec<? extends GameTestInstance>> TEST_INSTANCE = DeferredRegister.create(
        BuiltInRegistries.TEST_INSTANCE_TYPE,
        "examplemod"
);

public static final Supplier<MapCodec<? extends GameTestInstance>> EXAMPLE_INSTANCE_CODEC = TEST_INSTANCE.register(
    "example_test_instance",
    () -> RecordCodecBuilder.mapCodec(instance -> instance.group(
            Codec.INT.fieldOf("value1").forGetter(test -> test.value1),
            Codec.BOOL.fieldOf("value2").forGetter(test -> test.value2),
            TestData.CODEC.forGetter(ExampleTestInstance::info)
        ).apply(instance, ExampleTestInstance::new)
    )
);
```

然后，该测试实例就可以在数据包(Datapack)中使用了：

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个游戏测试 examplemod:example_test
// 位于 'data/examplemod/test_instance/example_test.json'
{
    // `TestData`

    "environment": "examplemod:example_environment",
    "structure": "examplemod:example_structure",
    "max_ticks": 400,
    "setup_ticks": 50,
    "required": true,
    "rotation": "clockwise_90",
    "manual_only": true,
    "max_attempts": 3,
    "required_successes": 1,
    "sky_access": false,

    // `ExampleTestInstance`
    "type": "examplemod:example_test_instance",

    "value1": 0,
    "value2": true
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 测试实例键
public static final ResourceKey<GameTestInstance> EXAMPLE_TEST_INSTANCE = ResourceKey.create(
    Registries.TEST_INSTANCE,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_test")
);

// 假设我们有一些测试环境(Test Environment)
public static final ResourceKey<TestEnvironmentDefinition> EXAMPLE_ENVIRONMENT = ResourceKey.create(
    Registries.TEST_ENVIRONMENT,
    ResourceLocation.fromNamespaceAndPath("examplemod", "example_environment")
);

@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(
        new RegistrySetBuilder().add(Registries.TEST_INSTANCE, bootstrap -> {
            // 使用此方法获取测试环境(Test Environment)
            HolderGetter<TestEnvironmentDefinition> environments = bootstrap.lookup(Registries.TEST_ENVIRONMENT);

            // 注册一个游戏测试(Game Test)
            // 任何与测试数据(Test Data)无关的字段都被隐藏
            bootstrap.register(EXAMPLE_TEST_INSTANCE,
                new ExampleTestInstance(
                    0,
                    true,
                    new TestData<>(
                        environments.getOrThrow(EXAMPLE_ENVIRONMENT),
                        ResourceLocation.fromNamespaceAndPath("examplemod", "example_structure"),
                        400,
                        50,
                        true,
                        Rotation.CLOCKWISE_90,
                        true,
                        3,
                        1,
                        false
                    )
            ));
        })
    );
}
```

</TabItem>
</Tabs>

### 跳过数据包(Skipping the Datapack)

如果您不想使用数据包(Datapack)来构建游戏测试(Game Tests)，可以改为监听[模组事件总线(mod event bus)][event]上的 `RegisterGameTestsEvent`，并分别通过 `registerEnvironment` 和 `registerTest` 注册您的环境和测试实例。

```java
@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void registerTests(RegisterGameTestsEvent event) {
    Holder<TestEnvironmentDefinition> environment = event.registerEnvironment(
        // 测试环境的名称
        EXAMPLE_ENVIRONMENT.location(),
        // 测试环境定义的可变参数列表
        new ExampleEnvironmentType(
            0, true
        )
    );

    event.registerTest(
        // 测试实例的名称
        EXAMPLE_TEST_INSTANCE.location(),
        new ExampleTestInstance(
            0,
            true,
            new TestData<>(
                environments.getOrThrow(EXAMPLE_ENVIRONMENT),
                ResourceLocation.fromNamespaceAndPath("examplemod", "example_structure"),
                400,
                50,
                true,
                Rotation.CLOCKWISE_90,
                true,
                3,
                1,
                false
            )
        )
    );
}
```

## 运行游戏测试(Running Game Tests)

可以使用 `/test` 命令运行游戏测试(Game Tests)。`test` 命令高度可配置；然而，只有少数对于运行测试很重要：

| 子命令(Subcommand) | 描述                                           |
| :----------------: | :--------------------------------------------- |
| `run`              | 运行指定的测试：`run <测试名称>`。           |
| `runall`           | 运行所有可用测试。                             |
| `runclosest`       | 运行玩家 15 格范围内最近的测试。               |
| `runthese`         | 运行玩家 200 格范围内的测试。                  |
| `runfailed`        | 运行上一次运行中所有失败的测试。               |

:::注意
子命令跟在测试命令之后：`/test <子命令>`。
:::

## 构建脚本配置(Buildscript Configurations)

游戏测试(Game Tests)在构建脚本（`build.gradle` 文件）内提供额外的配置设置，以在不同设置中运行和集成。

### 游戏测试服务器运行配置(Game Test Server Run Configuration)

游戏测试服务器(Game Test Server)是一种特殊的配置，用于运行构建服务器。该构建服务器返回一个退出代码，表示所需但失败的游戏测试(Game Tests)的数量。所有失败的测试，无论是必需的还是可选的，都会被记录。可以使用 `gradlew runGameTestServer` 运行此服务器。

### 在其他运行配置中启用游戏测试(Enabling Game Tests in Other Run Configurations)

默认情况下，只有 `client` 和 `gameTestServer` 运行配置启用了游戏测试(Game Tests)。如果另一个运行配置应该运行游戏测试(Game Tests)，则必须将 `neoforge.enableGameTest` 属性设置为 `true`。

```gradle
// 在运行配置内部
property 'neoforge.enableGameTest', 'true'
```

[datapacks]: ../resources/index.md#data
[registered]: ../concepts/registries.md#methods-for-registering
[test]: #running-game-tests
[event]: ../concepts/events.md#registering-an-event-handler