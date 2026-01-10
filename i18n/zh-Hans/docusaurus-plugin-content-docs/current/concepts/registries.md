---
sidebar_position: 1
---
# 注册表(Registries)

注册(Registration)是将模组(mod)的对象（如[物品][item]、[方块][block]、实体等）告知游戏的过程。注册非常重要，因为没有注册，游戏将无法识别这些对象，从而导致难以解释的行为和崩溃。

简单来说，注册表(Registry)是一个映射(map)的包装器，它将注册名称（见下文）映射到已注册的对象，这些对象通常称为注册项(registry entries)。在同一注册表中，注册名称必须唯一，但相同的注册名称可能出现在多个注册表中。最常见的例子是方块（在`BLOCKS`注册表中）具有相同注册名称的物品形式（在`ITEMS`注册表中）。

每个已注册的对象都有一个唯一的名称，称为其注册名称(registry name)。该名称由[`ResourceLocation`][resloc]表示。例如，泥土方块的注册名称是`minecraft:dirt`，僵尸的注册名称是`minecraft:zombie`。模组对象当然不会使用`minecraft`命名空间；而是使用其模组ID。

## 原版(Vanilla)与模组(Modded)

为了理解NeoForge注册系统的一些设计决策，我们首先看看Minecraft是如何做的。我们将以方块注册表为例，因为大多数其他注册表的工作方式相同。

注册表通常注册[单例(singletons)][singleton]。这意味着所有注册项只存在一次。例如，你在游戏中看到的所有石头方块实际上是同一个石头方块的多次显示。如果你需要石头方块，可以通过引用已注册的方块实例来获取它。

Minecraft在`Blocks`类中注册所有方块。通过`register`方法，调用`Registry#register()`，其中方块注册表`BuiltInRegistries.BLOCK`作为第一个参数。所有方块注册完成后，Minecraft会根据方块列表执行各种检查，例如验证所有方块是否已加载模型的自我检查。

这一切工作的主要原因是`Blocks`类被Minecraft足够早地加载。模组不会被Minecraft自动加载，因此需要解决方法。

## 注册方法

NeoForge提供了两种注册对象的方法：`DeferredRegister`类和`RegisterEvent`事件。请注意，前者是后者的包装器，推荐使用以防止错误。

### `DeferredRegister`

我们首先创建`DeferredRegister`：

``` java
public static final DeferredRegister<Block> BLOCKS = DeferredRegister.create(
        // 我们要使用的注册表。
        // Minecraft的注册表可以在BuiltInRegistries中找到，NeoForge的注册表可以在NeoForgeRegistries中找到。
        // 模组也可能添加自己的注册表，请参考各个模组的文档或源代码以找到它们。
        BuiltInRegistries.BLOCKS,
        // 我们的模组ID。
        ExampleMod.MOD_ID
);
```

然后，我们可以使用以下方法之一将注册项添加为静态最终字段（有关`new Block()`中要添加的参数，请参见[方块文章][block]）：

``` java
public static final DeferredHolder<Block, Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
        // 我们的注册名称。
        "example_block",
        // 我们要注册的对象的供应器(supplier)。
        () -> new Block(...)
);

public static final DeferredHolder<Block, SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
        // 我们的注册名称。
        "example_block",
        // 一个函数，根据其注册名称（作为ResourceLocation）创建我们要注册的对象。
        registryName -> new SlabBlock(...)
);
```

类`DeferredHolder<R, T extends R>`持有我们的对象。类型参数`R`是我们注册到的注册表类型（在我们的例子中是`Block`）。类型参数`T`是我们的供应器类型。由于在第一个示例中我们直接注册了一个`Block`，因此我们提供`Block`作为第二个参数。如果我们注册一个`Block`子类的对象，例如`SlabBlock`（如第二个示例所示），我们将在这里提供`SlabBlock`。

`DeferredHolder<R, T extends R>`是`Supplier<T>`的子类。当我们需要获取已注册的对象时，可以调用`DeferredHolder#get()`。`DeferredHolder`扩展`Supplier`的事实也允许我们使用`Supplier`作为字段的类型。这样，上面的代码块变为：

``` java
public static final Supplier<Block> EXAMPLE_BLOCK_1 = BLOCKS.register(
        // 我们的注册名称。
        "example_block",
        // 我们要注册的对象的供应器。
        () -> new Block(...)
);

public static final Supplier<SlabBlock> EXAMPLE_BLOCK_2 = BLOCKS.register(
        // 我们的注册名称。
        "example_block",
        // 一个函数，根据其注册名称（作为ResourceLocation）创建我们要注册的对象。
        registryName -> new SlabBlock(...)
);
```

:::note
请注意，有些地方明确要求`Holder`或`DeferredHolder`，而不接受任何`Supplier`。如果你需要这两种中的任何一种，最好根据需要将`Supplier`的类型改回`Holder`或`DeferredHolder`。
:::

最后，由于整个系统是围绕注册事件(registry events)的包装器，我们需要告诉`DeferredRegister`在需要时将其自身附加到注册事件：

``` java
//这是我们的模组构造函数
public ExampleMod(IEventBus modBus) {
    //highlight-next-line
    ExampleBlocksClass.BLOCKS.register(modBus);
    //其他内容在这里
}
```

:::info
对于方块、物品和数据组件，有专门的`DeferredRegister`变体，提供辅助方法：分别是[`DeferredRegister.Blocks`][defregblocks]、[`DeferredRegister.Items`][defregitems]、[`DeferredRegister.DataComponents`][defregcomp]和[`DeferredRegister.Entities`][defregentity]。
:::

### `RegisterEvent`

`RegisterEvent`是注册对象的第二种方式。这个[事件(event)][event]为每个注册表触发，在模组构造函数之后（因为那是`DeferredRegister`注册其内部事件处理器的地方）和配置加载之前。`RegisterEvent`在模组事件总线(mod event bus)上触发。

``` java
@SubscribeEvent // 在模组事件总线上
public static void register(RegisterEvent event) {
    event.register(
            // 这是注册表的注册键(registry key)。
            // 对于原版注册表，从BuiltInRegistries获取；对于NeoForge注册表，从NeoForgeRegistries.Keys获取。
            BuiltInRegistries.BLOCKS,
            // 在这里注册你的对象。
            registry -> {
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_1"), new Block(...));
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_2"), new Block(...));
                registry.register(ResourceLocation.fromNamespaceAndPath(MODID, "example_block_3"), new Block(...));
            }
    );
}
```

## 查询注册表

有时，你会遇到想要通过给定ID获取已注册对象的情况。或者，你想获取特定已注册对象的ID。由于注册表基本上是ID（`ResourceLocation`）到不同对象的映射，即一个可逆映射，这两种操作都有效：

``` java
BuiltInRegistries.BLOCKS.getValue(ResourceLocation.fromNamespaceAndPath("minecraft", "dirt")); // 返回泥土方块
BuiltInRegistries.BLOCKS.getKey(Blocks.DIRT); // 返回资源位置"minecraft:dirt"

// 假设ExampleBlocksClass.EXAMPLE_BLOCK.get()是一个ID为"yourmodid:example_block"的Supplier<Block>
BuiltInRegistries.BLOCKS.getValue(ResourceLocation.fromNamespaceAndPath("yourmodid", "example_block")); // 返回示例方块
BuiltInRegistries.BLOCKS.getKey(ExampleBlocksClass.EXAMPLE_BLOCK.get()); // 返回资源位置"yourmodid:example_block"
```

如果你只想检查对象是否存在，这也是可能的，但仅适用于键：

``` java
BuiltInRegistries.BLOCKS.containsKey(ResourceLocation.fromNamespaceAndPath("minecraft", "dirt")); // true
BuiltInRegistries.BLOCKS.containsKey(ResourceLocation.fromNamespaceAndPath("create", "brass_ingot")); // 仅当Create模组安装时为true
```

如最后一个示例所示，这适用于任何模组ID，因此是检查其他模组中是否存在特定物品的完美方式。

最后，我们还可以遍历注册表中的所有条目，无论是键还是条目（条目使用Java的`Map.Entry`类型）：

``` java
for (ResourceLocation id : BuiltInRegistries.BLOCKS.keySet()) {
    // ...
}
for (Map.Entry<ResourceKey<Block>, Block> entry : BuiltInRegistries.BLOCKS.entrySet()) {
    // ...
}
```

:::note
查询操作总是使用原版`Registry`，而不是`DeferredRegister`。这是因为`DeferredRegister`仅仅是注册工具。
:::

:::danger
查询操作仅在注册完成后安全使用。**在注册仍在进行时，不要查询注册表！**
:::

## 自定义注册表

自定义注册表允许你指定其他模组可能想要插入的附加系统。例如，如果你的模组要添加法术，你可以将法术设置为一个注册表，从而允许其他模组向你的模组添加法术，而无需你做任何其他事情。它还允许你自动执行一些操作，例如同步条目。

让我们首先创建[注册键(resource key)][resourcekey]和注册表本身：

``` java
// 我们以法术为例，这里不关心法术的具体细节（因为它不重要）。
// 当然，所有提到法术的地方都可以且应该替换为你的注册表实际内容。
public static final ResourceKey<Registry<Spell>> SPELL_REGISTRY_KEY = ResourceKey.createRegistryKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "spells"));
public static final Registry<YourRegistryContents> SPELL_REGISTRY = new RegistryBuilder<>(SPELL_REGISTRY_KEY)
        // 如果你想要启用整数ID同步，用于网络。
        // 这些应仅用于网络上下文，例如在数据包或纯粹与网络相关的NBT数据中。
        .sync(true)
        // 默认键。类似于方块的minecraft:air。这是可选的。
        .defaultKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "empty"))
        // 有效限制最大数量。通常不推荐，但在网络等设置中可能有意义。
        .maxId(256)
        // 构建注册表。
        .create();
```

然后，通过在`NewRegistryEvent`中将注册表注册到根注册表，告诉游戏该注册表存在：

``` java
@SubscribeEvent // 在模组事件总线上
public static void registerRegistries(NewRegistryEvent event) {
    event.register(SPELL_REGISTRY);
}
```

现在，你可以像任何其他注册表一样注册新的注册内容，通过`DeferredRegister`和`RegisterEvent`：

``` java
public static final DeferredRegister<Spell> SPELLS = DeferredRegister.create(SPELL_REGISTRY, "yourmodid");
public static final Supplier<Spell> EXAMPLE_SPELL = SPELLS.register("example_spell", () -> new Spell(...));

// 或者：
@SubscribeEvent // 在模组事件总线上
public static void register(RegisterEvent event) {
    event.register(SPELL_REGISTRY_KEY, registry -> {
        registry.register(ResourceLocation.fromNamespaceAndPath("yourmodid", "example_spell"), () -> new Spell(...));
    });
}
```

## 数据包注册表

数据包注册表（也称为动态注册表，或根据其主要用例称为世界生成注册表）是一种特殊类型的注册表，它在世界加载时从[数据包(datapack)][datapack] JSON加载数据，而不是在游戏启动时加载。默认的数据包注册表最显著地包括大多数世界生成注册表，以及其他一些。

数据包注册表允许其内容在JSON文件中指定。这意味着不需要代码（除了[数据生成(datagen)][datagen]，如果你不想自己编写JSON文件）。每个数据包注册表都有一个与之关联的[`Codec`][codec]，用于序列化，每个注册表的ID决定其数据包路径：

- Minecraft的数据包注册表使用格式`data/你的模组ID/注册表路径`（例如`data/你的模组ID/worldgen/biome`，其中`worldgen/biome`是注册表路径）。
- 所有其他数据包注册表（NeoForge或模组）使用格式`data/你的模组ID/注册表命名空间/注册表路径`（例如`data/你的模组ID/neoforge/biome_modifier`，其中`neoforge`是注册表命名空间，`biome_modifier`是注册表路径）。

数据包注册表可以从`RegistryAccess`获取。这个`RegistryAccess`可以通过在服务器上调用`ServerLevel#registryAccess()`来获取，或在客户端上调用`Minecraft.getInstance().getConnection()#registryAccess()`（后者仅在实际上连接到世界时有效，否则连接将为null）。这些调用的结果然后可以像任何其他注册表一样使用，以获取特定元素或遍历内容。

### 自定义数据包注册表

自定义数据包注册表不需要构建`Registry`。相反，它们只需要一个注册键和至少一个[`Codec`][codec]来（反）序列化其内容。重申之前的法术示例，将我们的法术注册表注册为数据包注册表如下所示：

``` java
public static final ResourceKey<Registry<Spell>> SPELL_REGISTRY_KEY = ResourceKey.createRegistryKey(ResourceLocation.fromNamespaceAndPath("yourmodid", "spells"));

@SubscribeEvent // 在模组事件总线上
public static void registerDatapackRegistries(DataPackRegistryEvent.NewRegistry event) {
    event.dataPackRegistry(
            // 注册键。
            SPELL_REGISTRY_KEY,
            // 注册表内容的编解码器(codec)。
            Spell.CODEC,
            // 注册表内容的网络编解码器。通常与普通编解码器相同。
            // 可能是普通编解码器的简化变体，省略了客户端不需要的数据。
            // 可能为null。如果为null，注册表条目将完全不会同步到客户端。
            // 可以省略，这在功能上等同于传递null（一个带有两个参数的方法重载被调用，它向普通的三参数方法传递null）。
            Spell.CODEC,
            // 一个消费者，通过RegistryBuilder配置构建的注册表。
            // 可以省略，这在功能上等同于传递builder -> {}。
            builder -> builder.maxId(256)
    );
}
```

### 数据包注册表的数据生成

由于手动编写所有JSON文件既繁琐又容易出错，NeoForge提供了一个[数据提供器(data provider)][datagenindex]来为你生成JSON文件。这适用于内置和你自己的数据包注册表。

首先，我们创建一个`RegistrySetBuilder`并向其中添加条目（一个`RegistrySetBuilder`可以包含多个注册表的条目）：

``` java
new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        // 通过引导上下文(bootstrap context)注册配置特征（见下文）
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        // 通过引导上下文注册放置特征（见下文）
    });
```

`bootstrap` lambda参数是我们实际用来注册对象的。它的类型是`BootstrapContext`。要注册一个对象，我们调用它的`#register`方法，如下所示：

``` java
// 我们对象的资源键。
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_CONFIGURED_FEATURE = ResourceKey.create(
    Registries.CONFIGURED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_configured_feature")
);

new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        bootstrap.register(
            // 我们配置特征的资源键。
            EXAMPLE_CONFIGURED_FEATURE,
            // 实际的配置特征。
            new ConfiguredFeature<>(Feature.ORE, new OreConfiguration(...))
        );
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        // ...
    });
```

`BootstrapContext`还可以用于在需要时从另一个注册表查找条目：

``` java
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_CONFIGURED_FEATURE = ResourceKey.create(
    Registries.CONFIGURED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_configured_feature")
);
public static final ResourceKey<PlacedFeature> EXAMPLE_PLACED_FEATURE = ResourceKey.create(
    Registries.PLACED_FEATURE,
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_placed_feature")
);

new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, bootstrap -> {
        bootstrap.register(EXAMPLE_CONFIGURED_FEATURE, ...);
    })
    .add(Registries.PLACED_FEATURE, bootstrap -> {
        HolderGetter<ConfiguredFeature<?, ?>> otherRegistry = bootstrap.lookup(Registries.CONFIGURED_FEATURE);
        bootstrap.register(EXAMPLE_PLACED_FEATURE, new PlacedFeature(
            otherRegistry.getOrThrow(EXAMPLE_CONFIGURED_FEATURE), // 获取配置特征
            List.of() // 放置发生时的无操作——替换为你的放置参数
        ));
    });
```

最后，我们在实际的数据提供器中使用我们的`RegistrySetBuilder`，并将该数据提供器注册到事件：

``` java
@SubscribeEvent // 在模组事件总线上
public static void onGatherData(GatherDataEvent.Client event) {
    // 将生成的注册表对象添加到当前查找提供器，以供其他数据生成使用。
    this.createDatapackRegistryObjects(
        // 我们用于生成数据的RegistrySetBuilder。
        new RegistrySetBuilder().add(...),
        //（可选）一个双消费者(biconsumer)，接受与资源键关联的对象的加载条件。
        conditions -> {
            conditions.accept(resourceKey, condition);
        },
        //（可选）我们为其生成条目的模组ID集合。
        // 默认提供当前模组容器的模组ID。
        Set.of("yourmodid")
    );

    // 你可以通过调用`#create*`方法之一或通过`#getLookupProvider`获取实际查找来使用带有生成条目的查找提供器。
    // ...
}
```

[block]: ../blocks/index.md
[blockentity]: ../blockentities/index.md
[codec]: ../datastorage/codecs.md
[datagen]: #data-generation-for-datapack-registries
[datagenindex]: ../resources/index.md#data-generation
[datapack]: ../resources/index.md#data
[defregblocks]: ../blocks/index.md#deferredregisterblocks-helpers
[defregcomp]: ../items/datacomponents.md#creating-custom-data-components
[defregentity]: ../entities/index.md#entitytype
[defregitems]: ../items/index.md#deferredregisteritems
[event]: events.md
[item]: ../items/index.md
[resloc]: ../misc/resourcelocation.md
[resourcekey]: ../misc/resourcelocation.md#resourcekeys
[singleton]: https://en.wikipedia.org/wiki/Singleton_pattern