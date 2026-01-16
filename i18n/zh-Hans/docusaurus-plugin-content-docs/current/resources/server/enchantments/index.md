import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 附魔

附魔是可以应用于工具和其他物品的特殊效果。从 1.21 开始，附魔作为[数据组件]存储在物品上，以 JSON 定义，并由所谓的附魔效果组件构成。在游戏中，特定物品上的附魔包含在 ``DataComponents.ENCHANTMENTS`` 组件中，即一个 ``ItemEnchantments`` 实例。

可以通过在命名空间的 ``enchantment`` 数据包子文件夹中创建 JSON 文件来添加新的附魔。例如，要创建一个名为 ``examplemod:example_enchant`` 的附魔，需要创建文件 ``data/examplemod/enchantment/example_enchantment.json``。

## 附魔 JSON 格式

```json5
{
    // 将用作附魔游戏中名称的文本组件。
    // 可以是翻译键或字面字符串。
    // 如果使用翻译键，请记得在语言文件中翻译它！
    "description": {
        "translate": "enchantment.examplemod.enchant_name"
    },
    
    // 此附魔可以应用于哪些物品。
    // 可以是物品 ID，如 "minecraft:trident"，
    // 或物品 ID 列表，如 ["examplemod:red_sword", "examplemod:blue_sword"]
    // 或物品标签，如 "#examplemod:enchantable/enchant_name"。
    // 注意：这不会导致附魔出现在这些物品的附魔台上。
    "supported_items": "#examplemod:enchantable/enchant_name",

    // （可选）此附魔在附魔台或作为附魔提供者的一部分时，出现在哪些物品上。
    // 要使附魔在附魔台上显示，必须将其添加到 ``minecraft:in_enchanting_table`` 标签。
    // ``minecraft:non_treasure`` 条目默认已在附魔台标签中。
    // 可以是物品、物品列表或物品标签。
    // 如果未指定，则与 ``supported_items`` 相同。
    "primary_items": [
        "examplemod:item_a",
        "examplemod:item_b"
    ],

    // （可选）哪些附魔与此附魔不兼容。
    // 可以是附魔 ID，如 "minecraft:sharpness"，
    // 或附魔 ID 列表，如 ["minecraft:sharpness", "minecraft:fire_aspect"]，
    // 或附魔标签，如 "#examplemod:exclusive_to_enchant_name"。
    // 不兼容的附魔不会通过原版机制添加到同一物品上。
    "exclusive_set": "#examplemod:exclusive_to_enchant_name",
    
    // 此附魔出现在附魔台上的可能性。
    // 范围在 [1, 1024]。
    "weight": 6,
    
    // 此附魔允许达到的最大等级。
    // 范围在 [1, 255]。
    "max_level": 3,
    
    // 此附魔的最大成本，以“附魔力量”衡量。
    // 这对应于但不等于玩家需要满足才能赋予此附魔的等级阈值。
    // 详见下文。
    // 实际成本将在此值和 min_cost 之间。
    "max_cost": {
        "base": 45,
        "per_level_above_first": 9
    },
    
    // 指定此附魔的最小成本；同上。
    "min_cost": {
        "base": 2,
        "per_level_above_first": 8
    },

    // 在铁砧中修复物品时，此附魔增加的等级成本。成本乘以附魔等级。
    // 如果物品具有 DataComponentTypes.STORED_ENCHANTMENTS 组件，则成本减半。在原版中，这仅适用于附魔书。
    // 范围在 [1, 无穷)。
    "anvil_cost": 2,
    
    // （可选）此附魔提供效果的槽位组列表。
    // 槽位组定义为 EquipmentSlotGroup 枚举的可能值之一。
    // 在原版中，这些是：``any``、``hand``、``mainhand``、``offhand``、``armor``、``feet``、``legs``、``chest``、``head`` 和 ``body``。
    "slots": [
        "mainhand"
    ],

    // 此附魔提供的效果，作为附魔效果组件的映射（见下文）。
    "effects": {
        "examplemod:custom_effect": [
            {
                "effect": {
                    "type": "minecraft:add",
                    "value": {
                        "type": "minecraft:linear",
                        "base": 1,
                        "per_level_above_first": 1
                    }
                }
            }
        ]
    }
}
```

### 附魔成本和等级

``max_cost`` 和 ``min_cost`` 字段指定了创建此附魔所需的附魔力量边界。然而，实际使用这些值的过程有些复杂。

首先，附魔台会考虑周围方块的 ``IBlockExtension#getEnchantPowerBonus()`` 返回值。由此，它调用 ``EnchantmentHelper#getEnchantmentCost`` 来推导每个槽位的“基础等级”。这个等级在游戏中显示为菜单中附魔旁边的绿色数字。对于每个附魔，基础等级通过由物品的可附魔性（其从 ``DataComponents#ENCHANTABLE`` 数据组件通过 ``Enchantable#value`` 提取的返回值）派生的随机值进行两次修改，如下所示：

``(修改后的等级) = (基础等级) + random.nextInt(e / 4 + 1) + random.nextInt(e / 4 + 1)``，其中 ``e`` 是可附魔性分数。

然后，这个修改后的等级会随机上下调整 15%，最终用于选择附魔。这个等级必须落在附魔的成本边界内才能被选择。

实际上，这意味着附魔定义中的成本值可能高于 30，有时甚至远高于 30。例如，对于可附魔性为 10 的物品，附魔台可能产生高达 1.15 * (30 + 2 * (10 / 4) + 1) = 40 成本的附魔。

## 附魔效果组件

附魔效果组件是特殊注册的[数据组件]，决定附魔的功能。组件的类型定义其效果，而它包含的数据用于告知或修改该效果。例如，``minecraft:damage`` 组件根据其数据修改武器造成的伤害。

原版定义了各种[内置附魔效果组件]，用于实现所有原版附魔。

### 自定义附魔效果组件

自定义附魔效果组件的逻辑必须由其创建者完全实现。首先，你应该定义一个类或记录来保存实现给定效果所需的信息。例如，让我们创建一个示例记录类 ``Increment``：

```java
// 定义一个示例数据承载记录。
public record Increment(int value) {
    public static final Codec<Increment> CODEC = RecordCodecBuilder.create(instance ->
            instance.group(
                    Codec.INT.fieldOf("value").forGetter(Increment::value)
            ).apply(instance, Increment::new)
    );

    public int add(int x) {
        return value() + x;
    }
}
```

附魔效果组件类型必须[注册]到 ``BuiltInRegistries.ENCHANTMENT_EFFECT_COMPONENT_TYPE``，它接受一个 ``DataComponentType<?>``。例如，你可以注册一个可以存储 ``Increment`` 对象的附魔效果组件，如下所示：

```java
// 在某个注册类中
public static final DeferredRegister.DataComponents ENCHANTMENT_COMPONENT_TYPES =
    DeferredRegister.createDataComponents(BuiltInRegistries.ENCHANTMENT_EFFECT_COMPONENT_TYPE, "examplemod");

public static final Supplier<DataComponentType<Increment>> INCREMENT =
    ENCHANTMENT_COMPONENT_TYPES.registerComponentType(
        "increment",
        builder -> builder.persistent(Increment.CODEC)
    );
```

现在，我们可以实现一些使用此组件来改变整数值的游戏逻辑：

```java
// 在游戏逻辑中某个有 ``itemStack`` 可用的地方。
// ``INCREMENT`` 是上面定义的附魔组件类型持有者。
// ``value`` 是一个整数。
AtomicInteger atomicValue = new AtomicInteger(value);

EnchantmentHelper.runIterationOnItem(stack, (enchantmentHolder, enchantLevel) -> {
    // 从附魔持有者获取 Increment 实例（如果是不同的附魔，则为 null）
    Increment increment = enchantmentHolder.value().effects().get(INCREMENT.get());

    // 如果此附魔有一个 Increment 组件，则使用它。
    if(increment != null){
        atomicValue.set(increment.add(atomicValue.get()));
    }
});

int modifiedValue = atomicValue.get();
// 在你的游戏逻辑的其他地方使用修改后的值。
```

首先，我们调用 ``EnchantmentHelper#runIterationOnItem`` 的某个重载。此函数接受一个 ``EnchantmentHelper.EnchantmentVisitor``，这是一个函数式接口，接受一个附魔及其等级，并对给定物品堆拥有的所有附魔调用（本质上是一个 ``BiConsumer<Holder<Enchantment>, Integer>``）。

为了实际执行调整，使用提供的 ``Increment#add`` 方法。由于这在 lambda 表达式内部，我们需要使用可以原子更新的类型，例如 ``AtomicInteger``，来修改这个值。这也允许多个 ``INCREMENT`` 组件在同一物品上运行并叠加效果，就像在原版中发生的那样。

### ``ConditionalEffect``

将类型包装在 ``ConditionalEffect<?>`` 中允许附魔效果组件基于给定的[战利品上下文]选择性地生效。

``ConditionalEffect`` 提供了 ``ConditionalEffect#matches(LootContext context)``，根据其内部的 ``Optional<LootItemConditon>`` 返回效果是否应运行，并处理其 ``LootItemCondition`` 的序列化和反序列化。

原版添加了一个额外的辅助方法来进一步简化检查这些条件的过程：``Enchantment#applyEffects()``。此方法接受一个 ``List<ConditionalEffect<T>>``，评估条件，并对每个条件满足的 ``ConditionalEffect`` 中包含的每个 ``T`` 运行一个 ``Consumer<T>``。由于许多原版附魔效果组件被定义为 ``List<ConditionalEffect<?>>``，它们可以直接插入辅助方法，如下所示：

```java
// ``enchant`` 是一个 Enchantment 实例。
// ``lootContext`` 是一个 LootContext 实例。
enchant.applyEffects(
    // 或你想要的任何其他 List<ConditionalEffect<T>>
    enchant.getEffects(EnchantmentEffectComponents.KNOCKBACK),
    // 测试条件的上下文
    lootContext,
    (effectData) -> // 以你想要的方式使用 effectData（在此示例中是一个 EnchantmentValueEffect）。
);
```

注册一个自定义的 ``ConditionalEffect`` 包装的附魔效果组件类型可以如下完成：

```java
public static final DeferredHolder<DataComponentType<?>, DataComponentType<ConditionalEffect<Increment>>> CONDITIONAL_INCREMENT =
    ENCHANTMENT_COMPONENT_TYPES.register("conditional_increment",
        () -> DataComponentType.ConditionalEffect<Increment>builder()
            // 所需的 ContextKeySet 取决于附魔应该做什么。
            // 这可能是 ENCHANTED_DAMAGE、ENCHANTED_ITEM、ENCHANTED_LOCATION、ENCHANTED_ENTITY 或 HIT_BLOCK 之一
            // 因为这些都将附魔等级带入上下文（以及其他指示的信息）。
            .persistent(ConditionalEffect.codec(Increment.CODEC, LootContextParamSets.ENCHANTED_DAMAGE))
            .build());
```

``ConditionalEffect.codec`` 的参数是泛型 ``ConditionalEffect<T>`` 的编解码器，后跟某个 ``ContextKeySet`` 条目。

## 附魔数据生成

附魔 JSON 文件可以使用[数据生成]系统自动创建，通过将 ``RegistrySetBuilder`` 传递给 ``DatapackBuiltInEntriesProvider``，通过 ``GatherDataEvent#createDatapackRegistryObjects``。JSON 将放置在 ``<project root>/src/generated/data/<modid>/enchantment/<path>.json``。

有关 ``RegistrySetBuilder`` 和 ``DatapackBuiltinEntriesProvider`` 如何工作的更多信息，请参阅关于[数据包注册表的数据生成]的文章。

<Tabs>
<TabItem value="datagen" label="数据生成">

```java

// 这个 RegistrySetBuilder 应该传递给你 ``GatherDataEvent`` 监听器中的 DatapackBuiltinEntriesProvider。
RegistrySetBuilder BUILDER = new RegistrySetBuilder();
BUILDER.add(
    Registries.ENCHANTMENT,
    bootstrap -> bootstrap.register(
        // 定义我们的附魔的 ResourceKey。
        ResourceKey.create(
            Registries.ENCHANTMENT,
            ResourceLocation.fromNamespaceAndPath("examplemod", "example_enchantment")
        ),
        new Enchantment(
            // 指定附魔名称的文本组件。
            Component.literal("Example Enchantment"),  
            
            // 为我们附魔的附魔定义。
            new Enchantment.EnchantmentDefinition(
                // 附魔将兼容的物品的 HolderSet。
                HolderSet.direct(...), 

                // 附魔视为“主要”的物品的可选 HolderSet。
                Optional.empty(), 

                // 附魔的权重。
                30, 

                // 此附魔可以达到的最大等级。
                3, 

                // 附魔的最小成本。第一个参数是基础成本，第二个是每级成本。
                Enchantment.dynamicCost(3, 1), 

                // 附魔的最大成本。同上。
                Enchantment.dynamicCost(4, 2), 

                // 附魔的铁砧成本。
                2, 

                // 此附魔有效果的 EquipmentSlotGroup 列表。
                List.of(EquipmentSlotGroup.ANY) 
            ),
            // 不兼容的其他附魔的 HolderSet。
            HolderSet.empty(), 

            // 与此附魔关联的附魔效果组件及其值的数据组件映射。
            DataComponentMap.builder() 
                .set(MY_ENCHANTMENT_EFFECT_COMPONENT_TYPE, new ExampleData())
                .build()
        )
    )
);

```

</TabItem>

<TabItem value="json" label="JSON" default>

```json5
// 有关每个条目的更多详细信息，请查看上面的附魔 JSON 格式部分。
{
    // 附魔的铁砧成本。
    "anvil_cost": 2,

    // 指定附魔名称的文本组件。
    "description": "Example Enchantment",

    // 与此附魔关联的效果组件及其值的映射。
    "effects": {
        // <效果组件>
    },

    // 附魔的最大成本。
    "max_cost": {
        "base": 4,
        "per_level_above_first": 2
    },

    // 此附魔可以达到的最大等级。
    "max_level": 3,

    // 附魔的最小成本。
    "min_cost": {
        "base": 3,
        "per_level_above_first": 1
    },

    // 此附魔有效果的 EquipmentSlotGroup 别名列表。
    "slots": [
        "any"
    ],

    // 可以使用铁砧应用此附魔的物品集合。
    "supported_items": /* <支持的物品列表> */,

    // 此附魔的权重。
    "weight": 30
}
```

</TabItem>
</Tabs>

[数据组件]: ../../../items/datacomponents.md
[编解码器]: ../../../datastorage/codecs.md
[附魔定义 Minecraft Wiki 页面]: https://minecraft.wiki/w/Enchantment_definition
[注册]: ../../../concepts/registries.md
[谓词]: https://minecraft.wiki/w/Predicate
[数据生成]: ../../../resources/index.md#data-generation
[数据包注册表的数据生成]: https://docs.neoforged.net/docs/concepts/registries/#data-generation-for-datapack-registries
[相关的 Minecraft Wiki 页面]: https://minecraft.wiki/w/Enchantment_definition#Entity_effects
[内置附魔效果组件]: builtin.md
[战利品上下文]: ../loottables/index.md#loot-context