# 全局战利品修改器

全局战利品修改器（Global Loot Modifiers），简称 GLM，是一种数据驱动的方式来修改掉落物，无需覆盖数十或数百个原版战利品表，也无需处理需要与另一个模组的战利品表交互而不知道加载了哪些模组的效果。

GLM 的工作原理是首先掷出相关的[战利品表][loottable]，然后将 GLM 应用于掷表的结果。GLM 也是叠加的，而不是后加载者胜出，以允许多个模组修改同一个战利品表，这与[标签][tags]类似。

要注册一个 GLM，你需要四样东西：

- 一个位于 `data/neoforge/loot_modifiers/global_loot_modifiers.json`（**不在你的模组命名空间内**）的 `global_loot_modifiers.json` 文件。该文件告诉 NeoForge 要应用哪些修改器，以及以什么顺序应用。
- 一个代表你的战利品修改器的 JSON 文件。该文件包含你修改的所有数据，允许数据包调整你的效果。它位于 `data/<命名空间>/loot_modifiers/<路径>.json`。
- 一个实现 `IGlobalLootModifier` 或扩展 `LootModifier`（它又实现了 `IGlobalLootModifier`）的类。这个类包含了使修改器工作的代码。
- 一个用于编码和解码你的战利品修改器类的映射[编解码器][codec]。通常，这在战利品修改器类中作为 `public static final` 字段实现。

## `global_loot_modifiers.json`

`global_loot_modifiers.json` 文件告诉 NeoForge 对战利品表应用哪些修改器。该文件可能包含两个键：

- `entries` 是应该加载的修改器列表。指定的 [`ResourceLocation`][resloc] 指向它们在 `data/<命名空间>/loot_modifiers/<路径>.json` 中的关联条目。此列表是有序的，意味着修改器将按照指定的顺序应用，这在出现模组兼容性问题时有时很重要。
- `replace` 表示修改器是应该替换旧的（`true`）还是简单地添加到现有列表中（`false`）。这与[标签][tags]中的 `replace` 键类似，但与标签不同的是，此处该键是必需的。通常，模组开发者应始终在此处使用 `false`；使用 `true` 的能力是针对模组包或数据包开发者的。

示例用法：

```json5
{
    "replace": false, // 必须存在
    "entries": [
        // 代表 data/examplemod/loot_modifiers/example_glm_1.json 中的一个战利品修改器
        "examplemod:example_glm_1",
        "examplemod:example_glm_2"
        // ...
    ]
}
```

## 战利品修改器 JSON

此文件包含与你的修改器相关的所有值，例如应用几率、要添加的物品等。建议尽可能避免硬编码值，以便数据包制作者可以调整平衡性（如果他们希望的话）。一个战利品修改器必须至少包含两个字段，根据情况可能包含更多：

- `type` 字段包含战利品修改器的注册名。
- `conditions` 字段是此修改器激活的战利品表条件列表。
- 根据使用的编解码器，可能需要或可选其他属性。

:::tip
GLM 的一个常见用例是向一个特定的战利品表添加额外的战利品。为了实现这一点，可以使用 [`neoforge:loot_table_id` 条件][loottableid]。
:::

一个示例用法可能如下所示：

```json5
{
    // 这是战利品修改器的注册名
    "type": "examplemod:my_loot_modifier",
    "conditions": [
        // 战利品表条件在此
    ],
    // 编解码器指定的额外属性
    "field1": "somestring",
    "field2": 10,
    "field3": "minecraft:dirt"
}
```

## `IGlobalLootModifier` 和 `LootModifier`

为了实际将战利品修改器应用于战利品表，必须指定一个 `IGlobalLootModifier` 实现。在大多数情况下，你会希望使用 `LootModifier` 子类，它为你处理诸如条件之类的事情。首先，我们在战利品修改器类中扩展 `LootModifier`：

```java
// 我们不能使用记录，因为记录不能扩展其他类。
public class MyLootModifier extends LootModifier {
    // 编解码器的工作方式见下文。
    public static final MapCodec<MyLootModifier> CODEC = ...;
    // 我们的额外属性。
    private final String field1;
    private final int field2;
    private final Item field3;
    
    // 第一个构造函数参数是条件列表。其余的是我们的额外属性。
    public MyLootModifier(LootItemCondition[] conditions, String field1, int field2, Item field3) {
        super(conditions);
        this.field1 = field1;
        this.field2 = field2;
        this.field3 = field3;
    }
    
    // 在此返回我们的编解码器。
    @Override
    public MapCodec<? extends IGlobalLootModifier> codec() {
        return CODEC;
    }
    
    // 这就是魔法发生的地方。如果需要，在此使用你的额外属性。
    // 参数是现有的战利品和战利品上下文。
    @Override
    protected ObjectArrayList<ItemStack> doApply(ObjectArrayList<ItemStack> generatedLoot, LootContext context) {
        // 在此将你的物品添加到 generatedLoot。
        return generatedLoot;
    }
}
```

:::info
从一个修改器返回的掉落物列表将按照它们注册的顺序馈送到其他修改器中。因此，可以而且应该预期修改过的战利品会被另一个战利品修改器修改。
:::

## 战利品修改器编解码器

为了告诉游戏我们的战利品修改器的存在，我们必须为其定义并[注册][register]一个[编解码器][codec]。重申我们之前带有三个字段的示例，看起来会像这样：

```java
public static final MapCodec<MyLootModifier> CODEC = RecordCodecBuilder.mapCodec(inst -> 
        // LootModifier#codecStart 添加 conditions 字段。
        LootModifier.codecStart(inst).and(inst.group(
                Codec.STRING.fieldOf("field1").forGetter(e -> e.field1),
                Codec.INT.fieldOf("field2").forGetter(e -> e.field2),
                BuiltInRegistries.ITEM.byNameCodec().fieldOf("field3").forGetter(e -> e.field3)
        )).apply(inst, MyLootModifier::new)
);
```

然后，我们将编解码器[注册][register]到注册表：

```java
public static final DeferredRegister<MapCodec<? extends IGlobalLootModifier>> GLOBAL_LOOT_MODIFIER_SERIALIZERS =
        DeferredRegister.create(NeoForgeRegistries.Keys.GLOBAL_LOOT_MODIFIER_SERIALIZERS, ExampleMod.MOD_ID);

public static final Supplier<MapCodec<MyLootModifier>> MY_LOOT_MODIFIER =
        GLOBAL_LOOT_MODIFIER_SERIALIZERS.register("my_loot_modifier", () -> MyLootModifier.CODEC);
```

## 内置战利品修改器

NeoForge 为你提供了一个开箱即用的战利品修改器：

### `neoforge:add_table`

此战利品修改器掷出第二个战利品表，并将结果添加到应用此修改器的战利品表中。

```json5
{
    "type": "neoforge:add_table",
    "conditions": [], // 所需的战利品条件
    "table": "minecraft:chests/abandoned_mineshaft" // 要掷出的第二个表
}
```

## 数据生成

GLM 可以[数据生成][datagen]。这是通过继承 `GlobalLootModifierProvider` 来完成的：

```java
public class MyGlobalLootModifierProvider extends GlobalLootModifierProvider {
    // 从 `GatherDataEvent` 中获取参数。
    public MyGlobalLootModifierProvider(PackOutput output, CompletableFuture<HolderLookup.Provider> registries) {
        super(output, registries, ExampleMod.MOD_ID);
    }
    
    @Override
    protected void start() {
        // 调用 #add 来添加新的 GLM。这也会在 global_loot_modifiers.json 中添加一个对应的条目。
        this.add(
                // 修改器的名称。这将是文件名。
                "my_loot_modifier_instance",
                // 要添加的战利品修改器。为了举例，我们添加一个天气战利品条件。
                new MyLootModifier(new LootItemCondition[] {
                        WeatherCheck.weather().setRaining(true).build()
                }, "somestring", 10, Items.DIRT),
                // 数据加载条件列表。请注意，这些与修改器本身指定的战利品条件无关。
                // 为了举例，我们添加一个模组已加载条件。
                // #add 的一个重载可用，它接受一个条件变参而不是列表。
                List.of(new ModLoadedCondition("create"))
        );
    }
}
```

像所有数据提供器一样，你必须将提供器注册到 `GatherDataEvent`：

```java
@SubscribeEvent // 在模组事件总线上
public static void onGatherData(GatherDataEvent.Client event) {
    // 如果添加数据包对象，请先调用 event.createDatapackRegistryObjects(...)

    event.createProvider(MyGlobalLootModifierProvider::new);
}
```

[codec]: ../../../datastorage/codecs.md
[datagen]: ../../index.md#data-generation
[loottable]: index.md
[loottableid]: lootconditions#neoforgeloot_table_id
[register]: ../../../concepts/registries.md#methods-for-registering
[resloc]: ../../../misc/resourcelocation.md
[tags]: ../tags.md