import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 内置附魔效果组件

原版 Minecraft 提供了多种不同类型的附魔效果组件，用于在[附魔]定义中使用。本文将解释每一种组件，包括它们的用法和代码内定义。

## 数值效果组件

_另请参阅 Minecraft Wiki 上的[数值效果组件]_

数值效果组件用于在游戏中改变某个数值的附魔，由类 ``EnchantmentValueEffect`` 实现。如果一个数值被多个数值效果组件改变（例如，由多个附魔），它们的所有效果都会应用。

数值效果组件可以设置为对给定的值使用以下任何操作：
- ``minecraft:set``：覆盖基于等级的给定值。
- ``minecraft:add``：将指定的基于等级的值添加到旧值上。
- ``minecraft:multiply``：将指定的基于等级的因子乘以旧值。
- ``minecraft:remove_binomial``：使用二项分布轮询给定的（基于等级的）几率。如果成功，则从值中减去 1。注意，许多值实际上是标志，在 1 时完全开启，在 0 时完全关闭。
- ``minecraft:all_of``：接受其他数值效果的列表，并按声明的顺序应用它们。

锋利附魔使用数值效果组件 ``minecraft:damage`` 来实现其效果，如下所示：

<Tabs>
<TabItem value="sharpness.json" label="JSON">

```json5
"effects": {
    // 此效果组件的类型是 "minecraft:damage"。
    // 这意味着效果将修改武器伤害。
    // 更多效果组件类型的列表见下文。
    "minecraft:damage": [
        {
            // 应应用的数值效果。
            // 在这种情况下，由于只有一个，所以这个数值效果只命名为 "effect"。
            "effect": {
                // 要使用的数值效果类型。在这种情况下，是 "minecraft:add"，因此该值（如下给出）将被添加到武器伤害值上。
                "type": "minecraft:add",

                // 值块。在这种情况下，该值是一个基于等级的值（LevelBasedValue），起始为 1，每附魔等级增加 0.5。
                "value": {
                    "type": "minecraft:linear",
                    "base": 1.0,
                    "per_level_above_first": 0.5
                }
            }
        }
    ]
}
```

</TabItem>
<TabItem value="sharpness.datagen" label="数据生成">

```java
// 在数据生成期间传递给附魔的 'effects'
// 有关更多信息，请参阅附魔条目中的数据生成部分
DataComponentMap.builder().set(
    // 选择 "minecraft:damage" 组件。
    EnchantmentEffectComponents.DAMAGE,

    // 构建一个没有要求的条件性 AddValue 列表。
    List.of(new ConditionalEffect<>(
        new AddValue(LevelBasedValue.perLevel(1.0F, 0.5F)),
        Optional.empty()))
).build()
```

</TabItem>
</Tabs>

``value`` 块内的对象是一个[基于等级的值]，可用于使数值效果组件根据等级改变其效果的强度。

``EnchantmentValueEffect#process`` 方法可用于根据提供的数值操作调整值，如下所示：

```java
// `valueEffect` 是一个 EnchantmentValueEffect 实例。
// `enchantLevel` 是一个整数，表示附魔等级
float baseValue = 1.0;
float modifiedValue = valueEffect.process(enchantLevel, server.random, baseValue);
```

### 原版附魔数值效果组件类型

#### 定义为 ``DataComponentType<EnchantmentValueEffect>``

- ``minecraft:crossbow_charge_time``：修改此弩的蓄力时间（以秒为单位）。用于快速装填。
- ``minecraft:trident_spin_attack_strength``：修改三叉戟旋转攻击的“强度”（参见 ``TridentItem#releaseUsing``）。用于激流。

#### 定义为 ``DataComponentType<List<ConditionalEffect<EnchantmentValueEffect>>>``

护甲相关：
- ``minecraft:armor_effectiveness``：确定武器对此护甲的有效性，范围从 0（无保护）到 1（正常保护）。用于穿透。
- ``minecraft:damage_protection``：每“点”伤害减免在使用此物品时减少 4% 的伤害，最大减免 80%。用于爆炸保护、摔落缓冲、火焰保护、保护和弹射物保护。

攻击相关：
- ``minecraft:damage``：修改使用此武器的攻击伤害。用于锋利、穿刺、节肢杀手、力量和不死杀手。
- ``minecraft:smash_damage_per_fallen_block``：为锤子添加每下落一个方块的伤害。用于致密。
- ``minecraft:knockback``：修改使用此武器时造成的击退量，以游戏单位衡量。用于击退和冲击。
- ``minecraft:mob_experience``：修改杀死生物获得的经验值。未使用。

耐久相关：
- ``minecraft:item_damage``：修改物品受到的耐久损伤。低于 1 的值作为物品受到伤害的几率。用于耐久。
- ``minecraft:repair_with_xp``：使物品使用获得的经验值修复自身，并确定其效果。用于经验修补。

弹射物相关：
- ``minecraft:ammo_use``：修改发射弓或弩时使用的弹药量。该值被钳位为整数，因此低于 1 的值将导致弹药使用量为 0。用于无限。
- ``minecraft:projectile_piercing``：修改从此武器发射的弹射物穿透的实体数量。用于穿透。
- ``minecraft:projectile_count``：修改射击此弓时生成的弹射物数量。用于多重射击。
- ``minecraft:projectile_spread``：修改弹射物发射方向的最大散布角度。用于多重射击。
- ``minecraft:trident_return_acceleration``：使三叉戟返回其所有者，并修改返回时应用到此三叉戟的加速度。用于忠诚。

其他：
- ``minecraft:block_experience``：修改破坏方块获得的经验值。用于精准采集。
- ``minecraft:fishing_time_reduction``：减少使用此鱼竿钓鱼时浮标下沉所需的时间（以秒为单位）。用于诱钓。
- ``minecraft:fishing_luck_bonus``：修改钓鱼战利品表中使用的[运气]值。用于海之眷顾。

#### 定义为 ``DataComponentType<List<TargetedConditionalEffect<EnchantmentValueEffect>>>``

- ``minecraft:equipment_drops``：修改被此武器杀死的实体掉落装备的几率。用于抢夺。

## 位置效果组件

_另请参阅：Minecraft Wiki 上的[位置效果组件]_

位置效果组件是实现 ``EnchantmentLocationBasedEffect`` 的组件。这些组件定义了需要知道附魔持有者在世界中位置的动作。它们使用两个主要方法操作：``EnchantmentEntityEffect#onChangedBlock``，在附魔物品被装备时以及持有者改变其 ``BlockPos`` 时调用；以及 ``onDeactivate``，在附魔物品被移除时调用。

以下是一个使用 ``minecraft:attributes`` 位置效果组件类型来改变持有者实体大小的示例：

<Tabs>
<TabItem value="attribute.json" label="JSON">

```json5
// 类型是 "minecraft:attributes"（如下所述）。
// 简而言之，这应用一个属性修饰符。
"minecraft:attributes": [
    {
        // 这个 "amount" 块是一个基于等级的值（LevelBasedValue）。
        "amount": {
            "type": "minecraft:linear",
            "base": 1,
            "per_level_above_first": 1
        },

        // 要修改的属性。在这种情况下，修改 "minecraft:scale"
        "attribute": "minecraft:scale",
        // 此属性修饰符的唯一标识符。不应与其他标识符重叠，但无需注册。
        "id": "examplemod:enchantment.size_change",
        // 对属性使用的操作。可以是 "add_value"、"add_multiplied_base" 或 "add_multiplied_total"。
        "operation": "add_value"
    }
],
```

</TabItem>
<TabItem value="attribute.datagen" label="数据生成">

```java
// 在数据生成期间传递给附魔的 effects
DataComponentMap.builder().set(
    // 指定 "minecraft:attributes" 组件类型。
    EnchantmentEffectComponents.ATTRIBUTES,

    // 此组件接受这些 EnchantmentAttributeEffect 对象的列表。
    List.of(new EnchantmentAttributeEffect(
        ResourceLocation.fromNamespaceAndPath("examplemod", "enchantment.size_change"),
        Attributes.SCALE,
        LevelBasedValue.perLevel(1F, 1F),
        AttributeModifier.Operation.ADD_VALUE
    ))
).build()
```

</TabItem>
</Tabs>

原版添加了以下位置效果事件：

- ``minecraft:all_of``：按顺序运行实体效果列表。
- ``minecraft:apply_mob_effect``：将[状态效果]应用于受影响的生物。
- ``minecraft:attribute``：将[属性修饰符]应用于附魔的持有者。
- ``minecraft:change_item_damage``：损坏此物品的耐久度。
- ``minecraft:damage_entity``：对受影响的实体造成伤害。如果在攻击上下文中，这与攻击伤害叠加。
- ``minecraft:explode``：召唤一次爆炸。
- ``minecraft:ignite``：点燃实体。
- ``minecraft:play_sound``：播放指定的声音。
- ``minecraft:replace_block``：替换给定偏移处的方块。
- ``minecraft:replace_disk``：替换一个方块圆盘。
- ``minecraft:run_function``：运行指定的[数据包函数]。
- ``minecraft:set_block_properies``：修改指定方块的方块状态属性。
- ``minecraft:spawn_particles``：生成粒子。
- ``minecraft:summon_entity``：召唤一个实体。

### 原版位置效果组件类型

#### 定义为 ``DataComponentType<List<ConditionalEffect<EnchantmentLocationBasedEffect>>>``

- ``minecraft:location_changed``：当持有者的方块位置改变以及此物品被装备时运行位置效果。用于冰霜行者和灵魂疾行。

#### 定义为 ``DataComponentType<List<EnchantmentAttributeEffect>>``

- ``minecraft:attributes``：将属性修饰符应用于持有者，并在附魔物品不再装备时移除它。

## 实体效果组件

_另请参阅 Minecraft Wiki 上的[实体效果组件]。_

实体效果组件是实现 ``EnchantmentEntityEffect`` 的组件，这是 ``EnchantmentLocationBasedEffect`` 的子类型。这些组件重写 ``EnchantmentLocationBasedEffect#onChangedBlock`` 以运行 ``EnchantmentEntityEffect#apply``；此外，根据组件的具体类型，``apply`` 方法也会在代码库的其他地方直接调用。这使得效果无需等待持有者的方块位置改变即可发生。

所有类型的位置效果组件也是有效的实体效果组件类型，除了 ``minecraft:attribute``，它仅作为位置效果组件注册。

以下是来自火焰附加附魔的此类组件的 JSON 定义示例：

<Tabs>
<TabItem value="fire.json" label="JSON">

```json5
// 此组件的类型是 "minecraft:post_attack"（见下文）。
"minecraft:post_attack": [
    {
        // 决定攻击的“受害者”、“攻击者”或“造成伤害的实体”（如果有弹射物则为弹射物，否则为攻击者）受到效果影响。
        "affected": "victim",
        
        // 决定应用哪个附魔实体效果。
        "effect": {
            // 此效果的类型是 "minecraft:ignite"。
            "type": "minecraft:ignite",
            // "minecraft:ignite" 需要一个基于等级的值（LevelBasedValue）作为实体被点燃的持续时间。
            "duration": {
                "type": "minecraft:linear",
                "base": 4.0,
                "per_level_above_first": 4.0
            }
        },

        // 决定谁（“受害者”、“攻击者”或“造成伤害的实体”）必须拥有附魔才能生效。
        "enchanted": "attacker",

        // 可选谓词，控制效果是否应用。
        "requirements": {
            "condition": "minecraft:damage_source_properties",
            "predicate": {
                "is_direct": true
            }
        }
    }
]
```

</TabItem>
<TabItem value="fire.datagen" label="数据生成">

```java
// 在数据生成期间传递给附魔的 effects
DataComponentMap.builder().set(
    // 指定 "minecraft:post_attack" 组件类型。
    EnchantmentEffectComponents.POST_ATTACK,

    // 定义此组件的数据。在这种情况下，是一个 TargetedConditionalEffect 的列表。
    List.of(
        new TargetedConditionalEffect<>(

            // 决定 "enchanted" 字段。
            EnchantmentTarget.ATTACKER,

            // 决定 "affected" 字段。
            EnchantmentTarget.VICTIM,

            // 附魔实体效果。
            new Ignite(LevelBasedValue.perLevel(4.0F, 4.0F)),

            // "requirements" 子句。
            // 在这种情况下，唯一激活的可选部分是 isDirect 布尔标志。
            Optional.of(
                new DamageSourceCondition(
                    Optional.of(
                        new DamageSourcePredicate(
                            List.of(),
                            Optional.empty(),
                            Optional.empty(),
                            Optional.of(true)
                        )
                    )
                )
            )
        )
    )
).build()
```

</TabItem>
</Tabs>

这里，实体效果组件是 ``minecraft:post_attack``。其效果是 ``minecraft:ignite``，由记录 ``Ignite`` 实现。该记录对 ``EnchantmentEntityEffect#apply`` 的实现将目标实体点燃。

### 原版附魔实体效果组件类型

#### 定义为 ``DataComponentType<List<ConditionalEffect<EnchantmentEntityEffect>>>``

- ``minecraft:hit_block``：当实体（例如弹射物）击中方块时运行实体效果。用于唤雷。
- ``minecraft:tick``：每游戏刻运行一次实体效果。用于灵魂疾行。
- ``minecraft:projectile_spawned``：在从弓或弩生成弹射物实体后运行实体效果。用于火矢。

#### 定义为 ``DataComponentType<List<TargetedConditionalEffect<EnchantmentEntityEffect>>>``

- ``minecraft:post_attack``：在攻击对实体造成伤害后运行实体效果。用于节肢杀手、唤雷、火焰附加、荆棘和风爆。

有关这些组件的更多详细信息，请查看[相关的 Minecraft Wiki 页面]。

## 其他原版附魔组件类型

#### 定义为 ``DataComponentType<List<ConditionalEffect<DamageImmunity>>>``

- ``minecraft:damage_immunity``：对指定伤害类型应用免疫。用于冰霜行者。

#### 定义为 ``DataComponentType<Unit>``

- ``minecraft:prevent_equipment_drop``：防止玩家死亡时掉落此物品。用于消失诅咒。
- ``minecraft:prevent_armor_change``：防止从盔甲槽位卸下此物品。用于绑定诅咒。

#### 定义为 ``DataComponentType<List<CrossbowItem.ChargingSounds>>``

- ``minecraft:crossbow_charge_sounds``：决定弩蓄力时发生的声音事件。每个条目代表附魔的一个等级。

#### 定义为 ``DataComponentType<List<Holder<SoundEvent>>>``

- ``minecraft:trident_sound``：决定使用三叉戟时发生的声音事件。每个条目代表附魔的一个等级。

[enchantment]: index.md
[数值效果组件]: https://minecraft.wiki/w/Enchantment_definition#Components_with_value_effects
[实体效果组件]: https://minecraft.wiki/w/Enchantment_definition#Components_with_entity_effects
[位置效果组件]: https://minecraft.wiki/w/Enchantment_definition#location_changed
[文本组件]: ../../client/i18n.md
[基于等级的值]: ../loottables/index.md#number-provider
[属性效果组件]: https://minecraft.wiki/w/Enchantment_definition#Attribute_effects
[数据包函数]: https://minecraft.wiki/w/Function_(Java_Edition)
[运气]: https://minecraft.wiki/w/Luck
[状态效果]: ../../../items/mobeffects.md
[属性修饰符]: ../../../entities/attributes.md#attribute-modifiers
[相关的 Minecraft Wiki 页面]: https://minecraft.wiki/w/Enchantment_definition#Components_with_entity_effects