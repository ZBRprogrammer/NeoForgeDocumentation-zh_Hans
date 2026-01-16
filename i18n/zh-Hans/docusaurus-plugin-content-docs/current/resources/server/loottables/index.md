# 战利品表

战利品表是用于定义随机掉落物的数据文件。可以掷出战利品表，返回一个（可能为空的）物品堆列表。此过程的输出取决于（伪）随机性。战利品表位于 `data/<mod_id>/loot_table/<名称>.json`。例如，泥土块使用的战利品表 `minecraft:blocks/dirt` 位于 `data/minecraft/loot_table/blocks/dirt.json`。

Minecraft 在游戏中的多个位置使用战利品表，包括[方块][block]掉落、[实体][entity]掉落、箱子战利品、钓鱼战利品等。如何引用战利品表取决于上下文：

- 默认情况下，每个方块都会获得一个关联的战利品表，位于 `<方块命名空间>:blocks/<方块名称>`。可以通过在方块的 `Properties` 上调用 `#noLootTable` 来禁用，导致不创建战利品表且方块不掉落任何物品；这主要由空气类或技术性方块完成。
- 每个未调用 `EntityType.Builder#noLootTable`（通常是 `MobCategory#MISC` 中的实体）的实体将默认获得一个关联的战利品表，位于 `<实体命名空间>:entities/<实体名称>`。可以通过重写 `#getLootTable` 来更改。例如，绵羊使用这个来根据羊毛颜色掷出不同的战利品表。
- 结构中的箱子在其方块实体数据中指定其战利品表。Minecraft 将所有箱子战利品表存储在 `minecraft:chests/<箱子名称>` 中；建议但不强制要求模组遵循此惯例。
- 袭击后村民可能向玩家投掷的礼物物品的战利品表在 [`neoforge:raid_hero_gifts` 数据映射][raidherogifts]中定义。
- 其他战利品表，例如钓鱼战利品表，在需要时从 `level.getServer().reloadableRegistries().getLootTable(lootTableKey)` 获取。所有原版战利品表位置的列表可以在 `BuiltInLootTables` 中找到。

:::warning
通常只应为属于你模组的东西创建战利品表。对于修改现有战利品表，应改用[全局战利品修改器 (GLMs)][glm]。
:::

由于战利品表系统的复杂性，战利品表由几个子系统组成，每个子系统都有不同的用途。

## 战利品条目

战利品条目（或战利品池条目），在代码中通过抽象类 `LootPoolEntryContainer` 表示，是一个单一的战利品元素。它可以指定一个或多个要掉落的物品。

原版总共提供了 8 种不同的战利品条目类型。通过公共的 `LootPoolEntryContainer` 超类，它们都具有以下属性：

- `weight`：权重值。默认为 1。用于某些物品应比其他物品更常见的情况。例如，给定两个战利品条目，一个权重为 3，另一个权重为 1，那么第一个条目被选中的几率为 75%，第二个条目为 25%。
- `quality`：质量值。默认为 0。如果此值非零，则将此值乘以运气值（在[战利品上下文][context]中设置）并在掷战利品表时添加到权重中。
- `conditions`：应用于此战利品条件的[战利品条件][lootcondition]列表。如果一个条件失败，该条目被视为不存在。
- `functions`：应用于此战利品条目输出的[战利品函数][lootfunction]列表。

战利品条目通常分为两组：单例（公共超类为 `LootPoolSingletonContainer`）和复合（公共超类为 `CompositeEntryBase`），其中复合条目由多个单例组成。Minecraft 提供了以下单例类型：

- `minecraft:empty`：一个空的战利品条目，表示没有物品。在代码中通过调用 `EmptyLootItem#emptyItem` 创建。
- `minecraft:item`：一个单一的战利品物品条目，掷出时掉落指定的物品。在代码中通过使用所需物品调用 `LootItem#lootTableItem` 创建。
    - 可以使用战利品函数设置堆叠大小、数据组件等。
- `minecraft:tag`：一个标签条目，掷出时掉落指定标签中的所有物品。根据布尔值 `expand` 属性的值，有两种变体。如果 `expand` 为 true，则为标签中的每个物品生成一个单独的条目；否则，使用一个条目来掉落所有物品。通过使用物品[标签键][tags]参数调用 `TagEntry#tagContents`（针对 `expand=false`）或 `TagEntry#expandTag`（针对 `expand=true`）创建。
    - 例如，如果 `expand` 为 true 且标签是 `#minecraft:planks`，则为每种木板类型生成一个条目（因此对于 11 种原版木板 + 每种模组木板有一个条目），每个条目都有指定的权重、质量和函数；而如果 `expand` 为 false，则使用一个掉落所有木板的单一条目。
- `minecraft:dynamic`：引用动态掉落的战利品条目。动态掉落是一个系统，用于向战利品表中添加无法预先指定的条目，而是在代码中添加。动态掉落条目由一个 id 和一个实际添加物品的 `Consumer<ItemStack>` 组成。要添加动态掉落条目，请指定一个具有所需 id 的 `minecraft:dynamic` 条目，然后在[战利品上下文][context]中添加相应的消费者。使用 `DynamicLoot#dynamicEntry` 创建。
- `minecraft:loot_table`：掷出另一个战利品表的战利品条目，将该战利品表的结果作为单个条目添加。另一个战利品表可以通过 id 指定，也可以完全内联。在代码中通过使用 `ResourceLocation` 参数调用 `NestedLootTable#lootTableReference`，或使用 `LootTable` 对象参数调用 `NestedLootTable#inlineLootTable` 以创建内联战利品表。

Minecraft 提供了以下复合类型：

- `minecraft:group`：包含其他战利品条目列表的战利品条目，按顺序运行。在代码中通过调用 `EntryGroup#list` 创建，或通过在其他 `LootPoolSingletonContainer.Builder` 上调用 `#append` 创建，每个都带有其他战利品条目构建器。
- `minecraft:sequence`：类似于 `minecraft:group`，但战利品条目在一个子条目失败时停止运行，丢弃该条目之后的所有条目。在代码中通过调用 `SequentialEntry#sequential` 创建，或通过在其他 `LootPoolSingletonContainer.Builder` 上调用 `#then` 创建，每个都带有其他战利品条目构建器。
- `minecraft:alternatives`：类似于 `minecraft:sequence` 的相反情况，但战利品条目在一个子条目成功时（而不是失败时）停止运行，丢弃该条目之后的所有条目。在代码中通过调用 `AlternativesEntry#alternatives` 创建，或通过在其他 `LootPoolSingletonContainer.Builder` 上调用 `#otherwise` 创建，每个都带有其他战利品条目构建器。

对于模组开发者，也可以定义[自定义战利品条目类型][customentry]。

## 战利品池

战利品池本质上是战利品条目的列表。战利品表可以包含多个战利品池，每个战利品池将独立于其他池掷出。

战利品池可能包含以下内容：

- `entries`：战利品条目列表。
- `conditions`：应用于此战利品池的[战利品条件][lootcondition]列表。如果一个条件失败，战利品池的所有条目都不会被掷出。
- `functions`：应用于此战利品池所有战利品条目输出的[战利品函数][lootfunction]列表。
- `rolls` 和 `bonus_rolls`：两个数字提供器（继续阅读）共同决定此战利品池将被掷出的次数。公式是 rolls + bonus_rolls * luck，其中运气值在[战利品参数][parameters]中设置。
- `name`：战利品池的名称。NeoForge 添加。可被 [GLMs][glm] 使用。如果未指定，则是战利品池的哈希码，前缀为 `custom#`。

## 数字提供器

数字提供器是一种在数据包上下文中获取（伪）随机数的方式。主要由战利品表使用，它们也用于其他上下文，例如世界生成。原版提供了以下六种数字提供器：

- `minecraft:constant`：一个常量浮点值，在需要时舍入为整数。通过 `ConstantValue#exactly` 创建。
- `minecraft:uniform`：均匀分布的随机整数或浮点值，设置了最小值和最大值。最小值和最大值之间的所有值出现的几率相同。通过 `UniformGenerator#between` 创建。
- `minecraft:binomial`：二项式分布的随机整数值，设置了 n 和 p 值。有关这些值的含义的更多信息，请参见[二项式分布][binomial]。通过 `BinomialDistributionGenerator#binomial` 创建。
- `minecraft:score`：给定一个实体目标、一个计分板名称和（可选）一个比例值，检索实体目标的给定计分板值，将其乘以给定的比例值（如果可用）。通过 `ScoreboardValue#fromScoreboard` 创建。
- `minecraft:storage`：来自给定 NBT 路径的命令存储中的值。通过 `new StorageValue` 创建。
- `minecraft:enchantment_level`：每个附魔等级的值的提供器。通过提供一个 `LevelBasedValue` 的 `EnchantmentLevelProvider#forEnchantmentLevel` 创建。有效的 `LevelBasedValue` 有：
    - 只是一个常量值，没有指定类型。通过 `LevelBasedValue#constant` 创建。
    - `minecraft:linear`：每附魔等级线性增加的值，加上一个可选的常量基础值。通过 `LevelBasedValue#perLevel` 创建。
    - `minecraft:levels_squared`：对附魔值进行平方，然后向其添加一个可选的基础值。通过 `new LevelBasedValue.LevelsSquared` 创建。
    - `minecraft:fraction`：接受另外两个 `LevelBasedValue`，使用它们创建一个分数。通过 `new LevelBasedValue.Fraction` 创建。
    - `minecraft:clamped`：接受另一个 `LevelBasedValue`，以及最小值和最大值。使用另一个 `LevelBasedValue` 计算值并钳制结果。通过 `new LevelBasedValue.Clamped` 创建。
    - `minecraft:lookup`：接受一个 `List<Float>` 和一个回退 `LevelBasedValue`。在列表中查找要使用的值（等级 1 是列表中的第一个元素，等级 2 是第二个元素，等等），如果某个等级的值缺失，则使用回退值。通过 `LevelBasedValue#lookup` 创建。

如果需要，模组开发者还可以注册[自定义数字提供器][customnumber]和[自定义基于等级的值][customlevelbased]。

## 战利品参数

战利品参数，内部称为 `ContextKey<T>`，是掷出战利品表时提供的参数，其中 `T` 是所提供参数的类型，例如 `BlockPos` 或 `Entity`。它们可以被[战利品条件][lootcondition]和[战利品函数][lootfunction]使用。例如，`minecraft:killed_by_player` 战利品条件检查是否存在 `minecraft:player` 参数。

Minecraft 提供了以下战利品参数：

- `minecraft:this_entity`：与战利品表关联的实体，通常是被杀死的实体。通过 `LootContextParams.THIS_ENTITY` 访问。
- `minecraft:interacting_entity`：正在与战利品表交互的实体，例如挖掘方块的玩家。通过 `LootContextParams.INTERACTING_ENTITY` 访问。
- `minecraft:target_entity`：与战利品表关联的实体，通常是某个交互的目标。通过 `LootContextParams.TARGET_ENTITY` 访问。
- `minecraft:last_damage_player`：与战利品表关联的玩家，通常是最后攻击被杀死实体的玩家，即使玩家击杀是间接的（例如：玩家轻触了实体，然后它被尖刺杀死）。用于例如仅限玩家击杀的掉落物。通过 `LootContextParams.LAST_DAMAGE_PLAYER` 访问。
- `minecraft:damage_source`：与战利品表关联的[伤害源][damagesource]，通常是杀死实体的伤害源。通过 `LootContextParams.DAMAGE_SOURCE` 访问。
- `minecraft:attacking_entity`：与战利品表关联的攻击实体，通常是实体的杀手。通过 `LootContextParams.ATTACKING_ENTITY` 访问。
- `minecraft:direct_attacking_entity`：与战利品表关联的直接攻击实体。例如，如果攻击实体是骷髅，那么直接攻击实体就是箭。通过 `LootContextParams.DIRECT_ATTACKING_ENTITY` 访问。
- `minecraft:origin`：与战利品表关联的位置，例如战利品箱的位置。通过 `LootContextParams.ORIGIN` 访问。
- `minecraft:block_state`：与战利品表关联的方块状态，例如被破坏的方块状态。通过 `LootContextParams.BLOCK_STATE` 访问。
- `minecraft:block_entity`：与战利品表关联的方块实体，例如与被破坏方块关联的方块实体。例如由潜影盒使用，将其库存保存到掉落的物品中。通过 `LootContextParams.BLOCK_ENTITY` 访问。
- `minecraft:tool`：与战利品表关联的物品堆，例如用于破坏方块的物品。这不一定是工具。通过 `LootContextParams.TOOL` 访问。
- `minecraft:explosion_radius`：当前上下文中的爆炸半径。主要用于将爆炸衰减应用于掉落物。通过 `LootContextParams.EXPLOSION_RADIUS` 访问。
- `minecraft:enchantment_level`：附魔等级，用于附魔逻辑。通过 `LootContextParams.ENCHANTMENT_LEVEL` 访问。
- `minecraft:enchantment_active`：所用物品是否有附魔，例如用于精准采集检查。通过 `LootContextParams.ENCHANTMENT_ACTIVE` 访问。

可以通过使用所需的 id 调用 `new ContextKey<T>` 来创建自定义战利品参数。由于它们仅仅是资源位置的包装器，因此不需要注册。

### 实体目标

实体目标是战利品条件和函数中使用的一种类型，在代码中由 `LootContext.EntityTarget` 枚举表示。它们用于指定在条件或函数上下文中要查询的实体战利品参数。有效值有：

- `"this"` 或 `LootContext.EntityTarget.THIS`：代表 `"minecraft:this_entity"` 参数。
- `"attacker"` 或 `LootContext.EntityTarget.ATTACKER`：代表 `"minecraft:attacking_entity"` 参数。
- `"direct_attacker"` 或 `LootContext.EntityTarget.DIRECT_ATTACKER`：代表 `"minecraft:direct_attacking_entity"` 参数。
- `"attacking_player"` 或 `LootContext.EntityTarget.ATTACKING_PLAYER`：代表 `"minecraft:last_damage_player"` 参数。
- `"target_entity"` 或 `LootContext.EntityTarget.TARGET_ENTITY`：代表 `"minecraft:target_entity"` 参数。
- `"interacting_entity"` 或 `LootContext.EntityTarget.INTERACTING_ENTITY`：代表 `"minecraft:interacting_entity"` 参数。

例如，`minecraft:entity_properties` 战利品条件接受一个实体目标，以允许检查所有四个战利品参数（如果你（作为战利品表作者）需要的话）。

### 战利品参数集

战利品参数集，也称为战利品表类型，在代码中称为 `ContextKeySet`s，是必需和可选战利品参数的集合。尽管名称如此，它们不是 `Set`s（甚至不是 `Collection`s）。相反，它们是围绕两个 `Set<ContextKey<?>>` 的包装器，一个保存必需参数（`#required`），一个保存可选参数（`#allowed`）。它们用于验证战利品参数的用户只使用预期可用的参数，并在掷表时验证必需参数是否存在。除此之外，它们还用于进度和附魔逻辑。

原版提供以下战利品参数集（必需参数为**粗体**，可选参数为*斜体*；代码内名称是 `LootContextParamSets` 中的常量）：

| ID                               | 代码内名称           | 指定的战利品参数                                                                                                                                                                                                                                                                                            | 用途                                                     |
|----------------------------------|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| `minecraft:empty`                | `EMPTY`                | 不适用                                                                                                                                                                                                                                                                                                                  | 回退目的。                                        |
| `minecraft:generic`              | `ALL_PARAMS`           | **`minecraft:origin`**, **`minecraft:tool`**, **`minecraft:block_state`**, **`minecraft:block_entity`**, **`minecraft:explosion_radius`**, **`minecraft:this_entity`**, **`minecraft:damage_source`**, **`minecraft:attacking_entity`**, **`minecraft:direct_attacking_entity`**, **`minecraft:last_damage_player`** | 验证。                                               |
| `minecraft:command`              | `COMMAND`              | **`minecraft:origin`**, _`minecraft:this_entity`_                                                                                                                                                                                                                                                                    | 命令。                                                 |
| `minecraft:selector`             | `SELECTOR`             | **`minecraft:origin`**, _`minecraft:this_entity`_                                                                                                                                                                                                                                                                    | 命令中的实体选择器。                             |
| `minecraft:block`                | `BLOCK`                | **`minecraft:origin`**, **`minecraft:tool`**, **`minecraft:block_state`**, _`minecraft:block_entity`_, _`minecraft:explosion_radius`_, _`minecraft:this_entity`_                                                                                                                                                     | 方块破坏。                                           |
| `minecraft:block_use`            | `BLOCK_USE`            | **`minecraft:origin`**, **`minecraft:block_state`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                     | 无原版用途。                                          |
| `minecraft:block_interact`       | `BLOCK_INTERACT`       | **`minecraft:block_state`**, _`minecraft:block_entity`_, _`minecraft:interacting_entity`_, _`minecraft:tool`_                                                                                                                                                                 | 方块交互。                                         |
| `minecraft:hit_block`            | `HIT_BLOCK`            | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:block_state`**, **`minecraft:this_entity`**                                                                                                                                                                                                  | 引雷附魔。                               |
| `minecraft:chest`                | `CHEST`                | **`minecraft:origin`**, _`minecraft:this_entity`_, _`minecraft:attacking_entity`_                                                                                                                                                                                                                                    | 战利品箱和类似容器，战利品箱矿车。 |
| `minecraft:archaeology`          | `ARCHAEOLOGY`          | **`minecraft:origin`**, **`minecraft:this_entity`**, **`minecraft:tool`**                                                                                                                                                                                                                                                                    | 考古。                                              |
| `minecraft:vault`                | `VAULT`                | **`minecraft:origin`**, _`minecraft:this_entity`_, _`minecraft:tool`_                                                                                                                                                                                                                                                                    | 试炼密室保险库奖励。                              |
| `minecraft:entity`               | `ENTITY`               | **`minecraft:origin`**, **`minecraft:this_entity`**, **`minecraft:damage_source`**, _`minecraft:attacking_entity`_, _`minecraft:direct_attacking_entity`_, _`minecraft:last_damage_player`_                                                                                                                          | 实体击杀。                                             |
| `minecraft:entity_interact`      | `ENTITY_INTERACT`      | **`minecraft:target_entity`**, **`minecraft:tool`**, _`minecraft:interacting_entity`_ | 实体交互。                                            |
| `minecraft:shearing`             | `SHEARING`             | **`minecraft:origin`**, **`minecraft:this_entity`**, **`minecraft:tool`**                                                                                                                                                                                                                                                                    | 剪切实体，例如绵羊。                            |
| `minecraft:equipment`            | `EQUIPMENT`            | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | 实体装备，例如僵尸。                        |
| `minecraft:gift`                 | `GIFT`                 | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | 袭击英雄礼物。                                          |
| `minecraft:barter`               | `PIGLIN_BARTER`        | **`minecraft:this_entity`**                                                                                                                                                                                                                                                                                          | 猪灵以物易物。                                         |
| `minecraft:fishing`              | `FISHING`              | **`minecraft:origin`**, **`minecraft:tool`**, _`minecraft:this_entity`_, _`minecraft:attacking_entity`_                                                                                                                                                                                                              | 钓鱼。                                                  |
| `minecraft:enchanted_item`       | `ENCHANTED_ITEM`       | **`minecraft:tool`**, **`minecraft:enchantment_level`**                                                                                                                                                                                                                                                              | 几个附魔。                                     |
| `minecraft:enchanted_entity`     | `ENCHANTED_ENTITY`     | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:this_entity`**                                                                                                                                                                                                                               | 几个附魔。                                     |
| `minecraft:enchanted_damage`     | `ENCHANTED_DAMAGE`     | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:this_entity`**, **`minecraft:damage_source`**, _`minecraft:attacking_entity`_, _`minecraft:direct_attacking_entity`_                                                                                                                         | 伤害和保护附魔。                       |
| `minecraft:enchanted_location`   | `ENCHANTED_LOCATION`   | **`minecraft:origin`**, **`minecraft:enchantment_level`**, **`minecraft:enchantment_active`**, **`minecraft:this_entity`**                                                                                                                                                                                           | 冰霜行者和灵魂疾行附魔。                 |
| `minecraft:advancement_entity`   | `ADVANCEMENT_ENTITY`   | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | 几个[进度条件][advancement]。              |
| `minecraft:advancement_location` | `ADVANCEMENT_LOCATION` | **`minecraft:origin`**, **`minecraft:tool`**, **`minecraft:block_state`**, **`minecraft:this_entity`**                                                                                                                                                                                                               | 几个[进度触发器][advancement]。              |
| `minecraft:advancement_reward`   | `ADVANCEMENT_REWARD`   | **`minecraft:origin`**, **`minecraft:this_entity`**                                                                                                                                                                                                                                                                  | [进度奖励][advancement]。                       |

### 战利品上下文

战利品上下文是一个包含掷战利品表时情境信息的对象。信息包括：

- 掷战利品表所在的 `ServerLevel`。通过 `#getLevel` 获取。
- 用于掷战利品表的 `RandomSource`。通过 `#getRandom` 获取。
- 战利品参数。使用 `#hasParameter` 检查存在性，并使用 `#getParameter` 获取单个参数。
- 运气值，用于计算奖励掷骰和质量值。通常通过实体的运气属性填充。通过 `#getLuck` 获取。
- 动态掉落消费者。更多信息见[上文][entry]。通过 `#addDynamicDrops` 设置。没有获取器可用。

## 战利品表

结合所有前面的元素，我们最终得到一个战利品表。战利品表 JSON 可以指定以下值：

- `pools`：战利品池列表。
- `neoforge:conditions`：[数据加载条件][conditions]列表。**警告：这些是数据加载条件，不是[战利品条件][lootcondition]！**
- `functions`：应用于此战利品表所有战利品条目输出的[战利品函数][lootfunction]列表。
- `type`：一个战利品参数集，用于验证战利品参数的正确使用。可选；如果不存在，将跳过验证。
- `random_sequence`：此战利品表的随机序列，以资源位置的形式。随机序列由 `Level` 提供，用于在相同条件下进行一致的战利品表掷骰。这通常使用战利品表的位置。

一个示例战利品表可以具有以下格式：

```json5
{
    "type": "chest", // 战利品参数集
    "neoforge:conditions": [
        // 数据加载条件
    ],
    "functions": [
        // 表级战利品函数
    ],
    "pools": [ // 战利品池列表
        {
            "rolls": 1, // 战利品表的掷骰次数，此处使用 5 将从池中产生 5 个结果
            "bonus_rolls": 0.5, // 奖励掷骰次数
            "name": "my_pool",
            "conditions": [
                // 池级战利品条件
            ],
            "functions": [
                // 池级战利品函数
            ],
            "entries": [ // 战利品表条目列表
                {
                    "type": "minecraft:item", // 战利品条目类型
                    "name": "minecraft:dirt", // 类型特定属性，例如物品的名称
                    "weight": 3, // 条目的权重
                    "quality": 1, // 条目的质量
                    "conditions": [
                        // 条目级战利品条件
                    ],
                    "functions": [
                        // 条目级战利品函数
                    ]
                }
            ]
        }
    ]
}
```

## 掷出战利品表

要掷出战利品表，我们需要两样东西：战利品表本身和战利品上下文。

让我们从获取战利品表本身开始。我们可以使用 `level.getServer().reloadableRegistries().getLootTable(lootTableId)` 获取战利品表。由于战利品数据仅通过服务器可用，此逻辑必须在[逻辑服务器][sides]上运行，而不是逻辑客户端。

:::tip
Minecraft 的内置战利品表 ID 可以在 `BuiltInLootTables` 类中找到。方块战利品表可以通过 `BlockBehaviour#getLootTable` 获取，实体战利品表可以通过 `EntityType#getDefaultLootTable` 或 `Entity#getLootTable` 获取。
:::

现在我们有了战利品表，让我们构建我们的参数集。我们首先创建一个 `LootParams.Builder` 实例：

```java
// 确保你在服务器上，否则转换将失败。
LootParams.Builder builder = new LootParams.Builder((ServerLevel) level);
```

然后我们可以添加战利品上下文参数，像这样：

```java
// 使用你需要的任何上下文参数和值。原版参数可以在 LootContextParams 中找到。
builder.withParameter(LootContextParams.ORIGIN, position);
// 此变体可以接受 null 作为值，在这种情况下，将删除该参数的现有值。
builder.withOptionalParameter(LootContextParams.ORIGIN, null);
// 添加动态掉落。
builder.withDynamicDrop(ResourceLocation.fromNamespaceAndPath("examplemod", "example_dynamic_drop"), stackAcceptor -> {
    // 一些逻辑在这里
});
// 设置我们的运气值。假设玩家可用。没有玩家的上下文应在此处使用 0。
builder.withLuck(player.getLuck());
```

最后，我们可以从构建器创建 `LootParams`，并使用它们掷出战利品表：

```java
// 如果需要，在此指定战利品上下文参数集。
LootParams params = builder.create(LootContextParamSets.EMPTY);
// 获取战利品表。
LootTable table = level.getServer().reloadableRegistries().getLootTable(location);
// 实际掷出战利品表。
List<ItemStack> list = table.getRandomItems(params);
// 如果你是为容器内容（例如战利品箱）掷战利品表，请改用此方法。
// 此方法负责将战利品物品正确分割到容器中。
List<ItemStack> containerList = table.fill(container, params, someSeed);
```

:::danger
`LootTable` 还公开了一个名为 `#getRandomItemsRaw` 的方法。与各种 `#getRandomItems` 变体不同，`#getRandomItemsRaw` 方法不会应用[全局战利品修改器][glm]。仅在你清楚你在做什么的情况下使用此方法。
:::

## 数据生成

可以通过注册一个 `LootTableProvider` 并在构造函数中提供 `LootTableSubProvider` 列表来[数据生成][datagen]战利品表：

```java
@SubscribeEvent // 在模组事件总线上
public static void onGatherData(GatherDataEvent.Client event) {
    // 如果添加数据包对象，请先调用 event.createDatapackRegistryObjects(...)

    event.createProvider((output, lookupProvider) -> new LootTableProvider(
        output,
        // 必需表资源位置的集合。这些随后会被验证是否存在。
        // 通常不建议模组验证存在性，
        // 因此我们传入一个空集合。
        Set.of(),
        // 子提供器条目列表。有关此处使用什么值，请参见下文。
        List.of(...),
        // 注册表访问器
        lookupProvider
    ));
}
```

### `LootTableSubProvider`s

`LootTableSubProvider`s 是实际生成发生的地方。首先，我们实现 `LootTableSubProvider` 并重写 `#generate`：

```java
public class MyLootTableSubProvider implements LootTableSubProvider {
    // 参数由 lambda 提供（见下文）。它可以存储并用于查找其他注册表条目。
    public MyLootTableSubProvider(HolderLookup.Provider lookupProvider) {
        // 将 lookupProvider 存储在字段中
    }

    @Override
    public void generate(BiConsumer<ResourceLocation, LootTable.Builder> consumer) {
        // LootTable.lootTable() 返回一个我们可以添加战利品表的战利品表构建器。
        consumer.accept(ResourceLocation.fromNamespaceAndPath(ExampleMod.MOD_ID, "example_loot_table"), LootTable.lootTable()
                // 添加战利品表级战利品函数。此示例使用数字提供器（见下文）。
                .apply(SetItemCountFunction.setCount(ConstantValue.exactly(5)))
                // 添加战利品池。
                .withPool(LootPool.lootPool()
                        // 添加战利品池级函数，类似于上面。
                        .apply(...)
                        // 添加战利品池级条件。此示例仅在雨天掷池。
                        .when(WeatherCheck.weather().setRaining(true))
                        // 分别设置掷骰次数和奖励掷骰次数。
                        // 这两种方法都利用数字提供器。
                        .setRolls(UniformGenerator.between(5, 9))
                        .setBonusRolls(ConstantValue.exactly(1))
                        // 添加战利品条目。此示例返回一个物品战利品条目。更多战利品条目见下文。
                        .add(LootItem.lootTableItem(Items.DIRT))
                )
        );
    }
}
```

一旦我们有了战利品表子提供器，我们将其添加到战利品提供器的构造函数中，像这样：

```java
new LootTableProvider(output, Set.of(), List.of(
        new SubProviderEntry(
                // 子提供器构造函数的引用。
                // 这是一个 Function<HolderLookup.Provider, ? extends LootTableSubProvider>。
                MyLootTableSubProvider::new,
                // 关联的战利品上下文集。如果不确定使用什么，请使用 empty。
                LootContextParamSets.EMPTY
        ),
        // 其他子提供器在此（如果适用）
    ), lookupProvider
);
```

### `BlockLootSubProvider`

`BlockLootSubProvider` 是一个抽象的辅助类，包含许多用于创建常见方块战利品表的辅助方法，例如单个物品掉落（`#createSingleItemTable`）、掉落为其创建表的方块（`#dropSelf`）、精准采集独有掉落（`#createSilkTouchOnlyTable`）、台阶类方块的掉落（`#createSlabItemTable`）等等。不幸的是，为模组使用设置 `BlockLootSubProvider` 涉及更多样板代码：

```java
public class MyBlockLootSubProvider extends BlockLootSubProvider {
    // 如果此类是你的战利品表提供器的内部类，则构造函数可以是私有的。
    // 参数由 LootTableProvider 构造函数中的 lambda 提供。
    public MyBlockLootSubProvider(HolderLookup.Provider lookupProvider) {
        // 第一个参数是我们为其创建战利品表的方块集合。我们不用硬编码，
        // 而是使用我们的方块注册表并在此处传递一个空集合。
        // 第二个参数是特性标志集，这将是默认标志，
        // 除非你添加自定义标志（这超出了本文的范围）。
        super(Set.of(), FeatureFlags.DEFAULT_FLAGS, lookupProvider);
    }

    // 此 Iterable 的内容用于验证。
    // 我们在此返回遍历我们方块注册表值的 Iterable。
    @Override
    protected Iterable<Block> getKnownBlocks() {
        // 我们的 DeferredRegister 的内容。
        return MyRegistries.BLOCK_REGISTRY.getEntries()
                .stream()
                // 在此转换为 Block，否则它将是 ? extends Block 并且 Java 会报错。
                .map(e -> (Block) e.value())
                .toList();
    }

    // 实际添加我们的战利品表。
    @Override
    protected void generate() {
        // 等效于调用 add(MyBlocks.EXAMPLE_BLOCK.get(), createSingleItemTable(MyBlocks.EXAMPLE_BLOCK.get()));
        this.dropSelf(MyBlocks.EXAMPLE_BLOCK.get());
        // 添加一个带有精准采集独有战利品表的表。
        this.add(MyBlocks.EXAMPLE_SILK_TOUCHABLE_BLOCK.get(),
                this.createSilkTouchOnlyTable(MyBlocks.EXAMPLE_SILK_TOUCHABLE_BLOCK.get()));
        // 其他战利品表添加在此
    }
}
```

然后，我们像添加任何其他子提供器一样将我们的子提供器添加到战利品表提供器的构造函数中：

```java
new LootTableProvider(output, Set.of(), List.of(new SubProviderEntry(
        MyBlockLootTableSubProvider::new,
        LootContextParamSets.BLOCK // 在此使用 BLOCK 是有意义的
    )), lookupProvider
);
```

### `EntityLootSubProvider`

类似于 `BlockLootSubProvider`，`EntityLootSubProvider` 提供了许多用于实体战利品表生成的辅助方法。也类似于 `BlockLootSubProvider`，我们必须提供一个提供器已知的 `Stream<EntityType<?>>`（而不是之前使用的 `Iterable<Block>`）。总的来说，我们的实现看起来非常类似于我们的 `BlockLootSubProvider`，但所有提到的方块都换成了实体类型：

```java
public class MyEntityLootSubProvider extends EntityLootSubProvider {
    public MyEntityLootSubProvider(HolderLookup.Provider lookupProvider) {
        // 与方块不同，我们不提供已知实体类型的集合。原版在此使用自定义检查。
        super(FeatureFlags.DEFAULT_FLAGS, lookupProvider);
    }

    // 这个类使用 Stream 而不是 Iterable，所以我们需要稍微调整一下。
    @Override
    protected Stream<EntityType<?>> getKnownEntityTypes() {
        return MyRegistries.ENTITY_TYPES.getEntries()
                .stream()
                .map(e -> (EntityType<?>) e.value());
    }

    @Override
    protected void generate() {
        this.add(MyEntities.EXAMPLE_ENTITY.get(), LootTable.lootTable());
        // 其他战利品表添加在此
    }
}
```

再次，我们然后将子提供器添加到战利品表提供器的构造函数中：

```java
new LootTableProvider(output, Set.of(), List.of(new SubProviderEntry(
        MyEntityLootTableSubProvider::new,
        LootContextParamSets.ENTITY
    )), lookupProvider
);
```

[advancement]: ../advancements.md
[binomial]: https://en.wikipedia.org/wiki/Binomial_distribution
[block]: ../../../blocks/index.md
[conditions]: ../conditions.md
[context]: #loot-context
[customentry]: custom.md#custom-loot-entry-types
[customlevelbased]: custom.md#custom-level-based-values
[customnumber]: custom.md#custom-number-providers
[damagesource]: ../damagetypes.md#creating-and-using-damage-sources
[datagen]: ../../index.md#data-generation
[entity]: ../../../entities/index.md
[entry]: #loot-entry
[glm]: glm.md
[lootcondition]: lootconditions
[lootfunction]: lootfunctions
[parameters]: #loot-parameters
[raidherogifts]: ../datamaps/builtin.md#neoforgeraid_hero_gifts
[sides]: ../../../concepts/sides.md
[tags]: ../tags.md