# 数据地图

数据地图包含可数据驱动、可重新加载的对象，这些对象可以附加到已注册的对象上。该系统允许更轻松地数据驱动游戏行为，因为它们提供了同步或冲突解决等功能，从而带来更好、更可配置的用户体验。你可以将[标签]视为注册表对象 ➜ 布尔值映射，而数据地图则是更灵活的注册表对象 ➜ 对象映射。与[标签]类似，数据地图会添加到其对应的数据地图中，而不是覆盖它。

数据地图可以附加到静态、内置的注册表和动态数据驱动的数据包注册表。数据地图支持通过使用 `/reload` 命令或任何其他重新加载服务器资源的方式重新加载。

NeoForge 为常见用例提供了各种[内置数据地图][builtin]，用于替换硬编码的模组原生字段。更多信息可以在链接的文章中找到。

## 文件位置

数据地图从位于 `<mapNamespace>/data_maps/<registryNamespace>/<registryPath>/<mapPath>.json` 的 JSON 文件加载，其中：

- `<mapNamespace>` 是数据地图 ID 的命名空间，
- `<mapPath>` 是数据地图 ID 的路径，
- `<registryNamespace>` 是注册表 ID 的命名空间（如果它是 `minecraft` 则省略），以及
- `<registryPath>` 是注册表 ID 的路径。

示例：

- 对于 `minecraft:item` 注册表的数据地图 `mymod:drop_healing`（如下例所示），路径将是 `mymod/data_maps/item/drop_healing.json`。
- 对于 `minecraft:block` 注册表的数据地图 `somemod:somemap`，路径将是 `somemod/data_maps/block/somemap.json`。
- 对于 `somemod:custom` 注册表的数据地图 `example:stuff`，路径将是 `example/data_maps/somemod/custom/stuff.json`。

## JSON 结构

数据地图文件本身可以包含以下字段：

- `replace`：一个布尔值，将在添加此文件的值之前清除数据地图。模组永远不应附带此字段，而只应由希望为自己的目的覆盖此地图的整合包开发者使用。
- `neoforge:conditions`：一个[加载条件][conditions]列表。
- `values`：一个注册表 ID 或标签 ID 到应通过你的模组添加到数据地图的值的映射。值本身的结构由数据地图的编解码器定义（见下文）。
- `remove`：要从数据地图中移除的注册表 ID 或标签 ID 列表。

### 添加值

例如，假设我们有一个数据地图对象，它有两个浮点数键 `amount` 和 `chance`，针对注册表 `minecraft:item`。相应的数据地图文件可能如下所示：

```json5
{
    "values": {
        // 将值附加到胡萝卜物品
        "minecraft:carrot": {
            "amount": 12,
            "chance": 1
        },
        // 将值附加到 logs 标签中的所有物品
        "#minecraft:logs": {
            "amount": 1,
            "chance": 0.1
        }
    }
}
```

数据地图可能支持[合并器][mergers]，这将在发生冲突（例如，两个模组为同一个物品添加数据地图值）时导致自定义合并行为。为了避免触发合并器，我们可以在元素级别指定 `replace` 字段，如下所示：

```json5
{
    "values": {
        // 覆盖胡萝卜物品的值
        "minecraft:carrot": {
            // highlight-next-line
            "replace": true,
            // 新值将位于 value 子对象下
            "value": {
                "amount": 12,
                "chance": 1
            }
        }
    }
}
```

### 移除现有值

可以通过指定要移除的物品 ID 或标签 ID 列表来移除元素：

```json5
{
    // 我们不希望马铃薯有值，即使另一个模组的数据地图添加了它
    "remove": [
        "minecraft:potato"
    ]
}
```

移除操作在添加操作之后运行，因此我们可以包含一个标签，然后再从中排除某些元素：

```json5
{
    "values": {
        "#minecraft:logs": { /* ... */ }
    },
    // 再次排除绯红菌柄
    "remove": [
        "minecraft:crimson_stem"
    ]
}
```

数据地图可能支持带有额外参数的自定义[移除器]。为了提供这些，可以将 `remove` 列表转换为 JSON 对象，其中包含待移除元素作为映射键，额外数据作为关联的值。例如，假设我们的移除器对象被序列化为字符串，那么我们的移除器映射可能如下所示：

```json5
{
    "remove": {
        // 移除器将从值反序列化（在本例中为 `somekey1`）
        // 并应用于附加到胡萝卜物品的值
        "minecraft:carrot": "somekey1"
    }
}
```

## 自定义数据地图

首先，我们定义数据地图条目的格式。**数据地图条目必须是不可变的**，因此记录（record）是理想选择。重申我们上面的例子，有两个浮点数值 `amount` 和 `chance`，我们的数据地图条目将如下所示：

```java
public record ExampleData(float amount, float chance) {}
```

与其他许多事物一样，数据地图使用[编解码器]进行序列化和反序列化。这意味着我们需要为稍后将使用的数据地图条目提供一个编解码器：

```java
public record ExampleData(float amount, float chance) {
    public static final Codec<ExampleData> CODEC = RecordCodecBuilder.create(instance -> instance.group(
            Codec.FLOAT.fieldOf("amount").forGetter(ExampleData::amount),
            Codec.floatRange(0, 1).fieldOf("chance").forGetter(ExampleData::chance)
    ).apply(instance, ExampleData::new));
}
```

接下来，我们创建数据地图本身：

```java
// 在这个例子中，我们为 minecraft:item 注册表注册数据地图，因此我们使用 Item 作为泛型。
// 如果你想为不同的注册表创建数据地图，请相应调整类型。
public static final DataMapType<Item, ExampleData> EXAMPLE_DATA = DataMapType.builder(
        // 数据地图的 ID。此数据地图的数据地图文件将位于
        // <yourmodid>:examplemod/data_maps/item/example_data.json。
        ResourceLocation.fromNamespaceAndPath("examplemod", "example_data"),
        // 要为其注册数据地图的注册表。
        Registries.ITEM,
        // 数据地图条目的编解码器。
        ExampleData.CODEC
).build();
```

最后，在[模组事件总线][modbus]上的 [`RegisterDataMapTypesEvent`][events] 期间注册数据地图：

```java
@SubscribeEvent // 在模组事件总线上
public static void registerDataMapTypes(RegisterDataMapTypesEvent event) {
    event.register(EXAMPLE_DATA);
}
```

### 同步

同步的数据地图会将其值同步到客户端。可以通过在构建器上调用 `#synced` 来标记数据地图为已同步，如下所示：

```java
public static final DataMapType<Item, ExampleData> EXAMPLE_DATA = DataMapType.builder(...)
        .synced(
                // 用于同步的编解码器。可能与普通编解码器相同，但也可能是
                // 字段较少的编解码器，省略客户端不需要的对象部分。
                ExampleData.CODEC,
                // 数据地图是否是强制性的。将数据地图标记为强制性将断开
                // 其端缺少数据地图的客户端连接；这包括原版客户端。
                false
        ).build();
```

### 使用

由于数据地图可用于任何注册表，因此必须通过 `Holder`（持有者）查询，而不是通过实际的注册表对象查询。而且，它仅适用于引用持有者，不适用于 `Direct`（直接）持有者。然而，大多数地方都会返回引用持有者，例如 `Registry#wrapAsHolder`、`Registry#getHolder` 或不同的 `builtInRegistryHolder` 方法，因此在大多数情况下这应该不是问题。

然后，你可以通过 `Holder#getData(DataMapType)` 查询数据地图值。如果对象没有附加数据地图值，该方法将返回 `null`。重用我们之前的 `ExampleData`，让我们使用它们在玩家捡起它们时治疗玩家：

```java
@SubscribeEvent // 在游戏事件总线上
public static void itemPickup(ItemEntityPickupEvent.Post event) {
    ItemStack stack = event.getOriginalStack();
    // 通过 ItemStack#getItemHolder 获取 Holder<Item>。
    Holder<Item> holder = stack.getItemHolder();
    // 从持有者获取数据。
    //highlight-next-line
    ExampleData data = holder.getData(EXAMPLE_DATA);
    if (data != null) {
        // 值存在，让我们对它们做些事情！
        Player player = event.getPlayer();
        if (player.getLevel().getRandom().nextFloat() > data.chance()) {
            player.heal(data.amount());
        }
    }
}
```

这个过程当然也适用于 NeoForge 提供的所有数据地图。

## 高级数据地图

高级数据地图是使用 `AdvancedDataMapType` 而不是标准 `DataMapType`（`AdvancedDataMapType` 是其子类）的数据地图。它们具有一些额外功能，即指定自定义合并器和自定义移除器的能力。对于值类型是集合或类集合（例如 `List` 或 `Map`）的数据地图，强烈推荐实现此功能。

虽然 `DataMapType` 有两个泛型 `R`（注册表类型）和 `T`（数据地图值类型），但 `AdvancedDataMapType` 还有一个：`VR extends DataMapValueRemover<R, T>`。这个泛型允许使用适当的类型安全性进行数据生成移除器。

`AdvancedDataMapType` 是使用 `AdvancedDataMapType#builder()` 而不是 `DataMapType#builder()` 创建的，返回一个 `AdvancedDataMapType.Builder`。此构建器有两个额外的方法 `#remover` 和 `#merger`，分别用于指定移除器和合并器（见下文）。所有其他功能，包括同步，保持不变。

### 合并器

合并器可用于处理多个尝试为同一对象添加值的数据包之间的冲突。默认合并器（`DataMapValueMerger#defaultMerger`）将用新值覆盖现有值（例如来自优先级较低的数据包的值），因此如果这不是期望的行为，则需要自定义合并器。

合并器将被赋予两个冲突的值，以及值所附加到的对象（作为 `Either<TagKey<R>, ResourceKey<R>>`，因为值可以附加到标签中的所有对象或单个对象）和对象所属的注册表，并应返回实际应附加的值。通常，合并器应尽可能只进行合并而不执行覆盖（即，只有当正常合并方式不起作用时才覆盖）。如果数据包想要绕过合并器，它应该在对象上指定 `replace` 字段（参见[添加值][add]）。

让我们设想一个场景，我们有一个为物品添加整数的数据地图。然后我们可以通过将两个值相加来解决冲突，如下所示：

```java
public class IntMerger implements DataMapValueMerger<Item, Integer> {
    @Override
    public Integer merge(Registry<Item> registry,
            Either<TagKey<Item>, ResourceKey<Item>> first, Integer firstValue,
            Either<TagKey<Item>, ResourceKey<Item>> second, Integer secondValue) {
        return firstValue + secondValue;
    }
}
```

这样，如果一个整合包为 `minecraft:carrot` 指定值 12，另一个整合包为 `minecraft:carrot` 指定值 15，那么 `minecraft:carrot` 的最终值将是 27。如果这些对象中的任何一个指定了 `"replace": true`，则将使用该对象的值。如果两者都指定了 `"replace": true`，则使用优先级更高的数据包的值。

最后，不要忘记在构建器中实际指定合并器，如下所示：

```java
// 数据地图的类型必须与合并器的类型匹配。
AdvancedDataMapType<Item, Integer> ADVANCED_MAP = AdvancedDataMapType.builder(...)
        .merger(new IntMerger())
        .build();
```

:::tip
NeoForge 在 `DataMapValueMerger` 中为列表、集合和映射提供了默认合并器。
:::

### 移除器

与更复杂数据的合并器类似，移除器可用于正确处理元素的 `remove` 子句。默认移除器（`DataMapValueRemover.Default.INSTANCE`）将简单地移除与指定对象相关的任何和所有信息，因此我们希望使用自定义移除器来仅移除对象数据的某些部分。

传递给构建器的编解码器（请继续阅读）将用于解码移除器实例。然后移除器将被传递当前附加到对象的值及其来源，并应返回一个 `Optional`，其中包含要替换旧值的值。或者，一个空的 `Optional` 将导致值被实际移除。

考虑以下移除器的示例，该移除器将从基于 `Map<String, String>` 的数据地图中移除具有特定键的值：

```java
public record MapRemover(String key) implements DataMapValueRemover<Item, Map<String, String>> {
    public static final Codec<MapRemover> CODEC = Codec.STRING.xmap(MapRemover::new, MapRemover::key);
    
    @Override
    public Optional<Map<String, String>> remove(Map<String, String> value, Registry<Item> registry, Either<TagKey<Item>, ResourceKey<Item>> source, Item object) {
        final Map<String, String> newMap = new HashMap<>(value);
        newMap.remove(key);
        return Optional.of(newMap);
    }
}
```

考虑到这个移除器，考虑以下数据文件：

```json5
{
    "values": {
        "minecraft:carrot": {
            "somekey1": "value1",
            "somekey2": "value2"
        }
    }
}
```

现在，考虑这个放置在比第一个文件优先级更高的第二个数据文件：

```json5
{
    "remove": {
        // 由于移除器被解码为字符串，我们可以在这里使用字符串作为值。
        // 如果它被解码为一个对象，我们就需要使用一个对象。
        "minecraft:carrot": "somekey1"
    }
}
```

这样，在两个文件都应用后，最终结果将是（内存中的表示）这个：

```json5
{
    "values": {
        "minecraft:carrot": {
            "somekey2": "value2"
        }
    }
}
```

与合并器一样，不要忘记将它们添加到构建器中。请注意，我们在这里只使用编解码器：

```java
// 我们假设 AdvancedData 包含某种 Map<String, String> 属性。
AdvancedDataMapType<Item, AdvancedData> ADVANCED_MAP = AdvancedDataMapType.builder(...)
        .remover(MapRemover.CODEC)
        .build();
```

## 数据生成

可以通过扩展 `DataMapProvider` 并覆盖 `#gather` 来创建你的条目，从而[数据生成][datagen]数据地图。重用我们之前的 `ExampleData`（具有浮点数值 `amount` 和 `chance`），我们的数据生成文件可能如下所示：

```java
public class MyDataMapProvider extends DataMapProvider {
    public MyDataMapProvider(PackOutput packOutput, CompletableFuture<HolderLookup.Provider> lookupProvider) {
        super(packOutput, lookupProvider);
    }
    
    @Override
    protected void gather() {
        // 我们为 EXAMPLE_DATA 数据地图创建一个构建器，并使用 #add 添加我们的条目。
        this.builder(EXAMPLE_DATA)
                // 我们开启替换。永远不要这样发布模组！这纯粹是为了教育目的。
                .replace(true)
                // 我们为所有台阶添加值 "amount": 10, "chance": 1。布尔参数控制
                // "replace" 字段，这在模组中应始终为 false。
                .add(ItemTags.SLABS, new ExampleData(10, 1), false)
                // 我们为苹果添加值 "amount": 5, "chance": 0.2。
                .add(Items.APPLE.builtInRegistryHolder(), new ExampleData(5, 0.2f), false) // 也可以使用 Registry#wrapAsHolder 获取注册表对象的持有者
                // 我们再次移除木质台阶。
                .remove(ItemTags.WOODEN_SLABS)
                // 我们为植物魔法添加一个模组加载条件，为什么不呢。
                .conditions(new ModLoadedCondition("botania"));
    }
}
```

然后将产生以下 JSON 文件：

```json5
{
    "replace": true,
    "values": {
        "#minecraft:slabs": {
            "amount": 10,
            "chance": 1.0
        },
        "minecraft:apple": {
            "amount": 5,
            "chance": 0.2
        }
    },
    "remove": [
        "#minecraft:wooden_slabs"
    ],
    "neoforge:conditions": [
        {
            "type": "neoforge:mod_loaded",
            "modid": "botania"
        }
    ]
}
```

与所有数据提供器一样，不要忘记将提供器添加到事件中：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    // 如果添加数据包对象，请先调用 event.createDatapackRegistryObjects(...)

    event.createProvider(MyDataMapProvider::new);
}
```

[builtin]: builtin.md
[codecs]: ../../../datastorage/codecs.md
[conditions]: ../conditions.md
[datagen]: ../../index.md#data-generation
[events]: ../../../concepts/events.md
[add]: #adding-values
[mergers]: #mergers
[modbus]: ../../../concepts/events.md#event-buses
[removers]: #removers
[tags]: ../tags.md