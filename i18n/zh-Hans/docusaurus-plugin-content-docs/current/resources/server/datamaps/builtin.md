# 内置数据地图

NeoForge 为常见用例提供了各种内置的[数据地图][datamap]，用于替换硬编码的模组原生字段。原版值由 NeoForge 中的数据地图文件提供，因此对玩家而言没有功能上的差异。

## `neoforge:acceptable_villager_distances`

允许配置村民能注意到实体的最大方块距离，作为 `VillagerHostilesSensor.ACCEPTABLE_DISTANCE_FROM_HOSTILES` 的替代（该字段将在 1.22 中被忽略）。此数据地图位于 `neoforge/data_maps/entity_type/acceptable_villager_distances.json`，其对象结构如下：

```json5
{
    // 村民会将此实体检测为敌对的最大方块距离
    "acceptable_villager_distance": 4.0
}
```

示例：

```json5
{
    "values": {
        // 如果烈焰人位于村民位置 4 格范围内，村民会将其检测为敌对
        "minecraft:blaze": {
            "acceptable_villager_distance": 4.0
        }
    }
}
```

## `neoforge:compostables`

允许配置堆肥桶值，作为 `ComposterBlock.COMPOSTABLES` 的替代（该字段现已被忽略）。此数据地图位于 `neoforge/data_maps/item/compostables.json`，其对象结构如下：

```json5
{
    // 一个 0 到 1（包含）之间的浮点数，表示该物品将更新堆肥桶等级的几率
    "chance": 1,
    // 可选，默认为 false - 农民村民是否可以将此物品堆肥
    "can_villager_compost": false
}
```

示例：

```json5
{
    "values": {
        // 赋予金合欢原木 50% 的几率来填充堆肥桶
        "minecraft:acacia_log": {
            "chance": 0.5
        }
    }
}
```

## `neoforge:furnace_fuels`

允许配置物品的燃烧时间。此数据地图位于 `neoforge/data_maps/item/furnace_fuels.json`，其对象结构如下：

```json5
{
    // 一个正整数，表示物品的燃烧时间（以游戏刻为单位）
    "burn_time": 1000
}
```

示例：

```json5
{
    "values": {
        // 赋予铁砧 2 秒的燃烧时间
        "minecraft:anvil": {
            "burn_time": 40
        }
    }
}
```

:::info
NeoForge 额外添加了 `IItemExtension#getBurnTime` 方法，可在自定义物品中重写，此方法会覆盖此数据地图。`#getBurnTime` 应仅用于数据地图不够用的情况，例如依赖于[数据组件][datacomponent]的燃烧时间。
:::

:::warning
原版为 `#minecraft:logs` 和 `#minecraft:planks` 隐式添加了 300 刻（15 秒）的燃烧时间，然后硬编码移除了绯红和诡异物品的燃烧时间。这意味着如果你添加了另一种不可燃的木材，你应该从此地图中移除该木材类型的物品，如下所示：

```json5
{
    "replace": false,
    "values": [
        // 值放在这里
    ],
    "remove": [
        "examplemod:example_nether_wood_planks",
        "#examplemod:example_nether_wood_stems",
        "examplemod:example_nether_wood_door",
        // 等等。
        // 其他要移除的项放在这里
    ]
}
```
:::

## `neoforge:monster_room_mobs`

允许配置可能出现在怪物房间刷怪笼中的生物，作为 `MonsterRoomFeature#MOBS` 的替代（该字段现已被忽略）。此数据地图位于 `neoforge/data_maps/entity_type/monster_room_mobs.json`，其对象结构如下：

```json5
{
    // 此生物的权重，相对于数据地图中的其他生物
    "weight": 100
}
```

示例：

```json5
{
    "values": {
        // 让鱿鱼以权重 100 出现在怪物房间刷怪笼中
        "minecraft:squid": {
            "weight": 100
        }
    }
}
```

## `neoforge:oxidizables`

允许配置氧化阶段，作为 `WeatheringCopper#NEXT_BY_BLOCK` 的替代。此数据地图也用于构建反向脱氧地图（用于用斧头刮削）。它位于 `neoforge/data_maps/block/oxidizables.json`，其对象结构如下：

```json5
{
    // 该方块氧化后将变成的方块
    "next_oxidation_stage": "examplemod:oxidized_block"
}
```

:::note
自定义方块必须实现 `WeatheringCopperFullBlock` 或 `WeatheringCopper`，并在 `randomTick` 中调用 `changeOverTime` 以自然氧化。
:::

示例：

```json5
{
    "values": {
        "mymod:custom_copper": {
            // 让自定义铜方块氧化成自定义的氧化铜
            "next_oxidation_stage": "mymod:custom_oxidized_copper"
        }
    }
}
```

## `neoforge:parrot_imitations`

允许配置鹦鹉想要模仿生物时产生的声音，作为 `Parrot#MOB_SOUND_MAP` 的替代（该字段现已被忽略）。此数据地图位于 `neoforge/data_maps/entity_type/parrot_imitations.json`，其对象结构如下：

```json5
{
    // 鹦鹉模仿该生物时产生的声音的 ID
    "sound": "minecraft:entity.parrot.imitate.creeper"
}
```

示例：

```json5
{
    "values": {
        // 让鹦鹉模仿悦灵时产生洞穴环境音
        "minecraft:allay": {
            "sound": "minecraft:ambient.cave"
        }
    }
}
```

## `neoforge:raid_hero_gifts`

允许配置当你阻止袭击后，拥有特定 `VillagerProfession`（村民职业）的村民可能赠送给你的礼物，作为 `GiveGiftToHero#GIFTS` 的替代（该字段现已被忽略）。此数据地图位于 `neoforge/data_maps/villager_profession/raid_hero_gifts.json`，其对象结构如下：

```json5
{
    // 袭击后村民职业将分发的战利品表的 ID
    "loot_table": "minecraft:gameplay/hero_of_the_village/armorer_gift"
}
```

示例：

```json5
{
    "values": {
        "minecraft:armorer": {
            // 让盔甲匠给予袭击英雄盔甲匠礼物的战利品表
            "loot_table": "minecraft:gameplay/hero_of_the_village/armorer_gift"
        }
    }
}
```

## `neoforge:strippables`

允许配置方块被剥离（用斧头或具有 `ItemAbilities#AXE_STRIP` 物品能力的物品右击）时将变成的方块，作为 `AxeItem#STRIPPABLES` 的替代（该字段将在 1.22 中被忽略）。此数据地图位于 `neoforge/data_maps/block/strippables.json`，其对象结构如下：

```json5
{
    // 当被具有 `ItemAbilities#AXE_STRIP` 物品能力的工具剥离时，此方块将变成的方块
    "stripped_block": "examplemod:stripped_wood"
}
```

示例：

```json5
{
    "values": {
        "examplemod:wood": {
            // 让自定义木方块剥离成自定义的去皮木方块
            "stripped_block": "examplemod:stripped_wood"
        }
    }
}
```

## `neoforge:vibration_frequencies`

允许配置由游戏事件触发的潜声震动频率，作为 `VibrationSystem#VIBRATION_FREQUENCY_FOR_EVENT` 的替代（该字段现已被忽略）。此数据地图位于 `neoforge/data_maps/game_event/vibration_frequencies.json`，其对象结构如下：

```json5
{
    // 一个 1 到 15（包含）之间的整数，表示事件的震动频率
    "frequency": 2
}
```

示例：

```json5
{
    "values": {
        // 让水中飞溅的游戏事件在第二个频率上震动
        "minecraft:splash": {
            "frequency": 2
        }
    }
}
```

## `neoforge:villager_types`

允许配置基于生物群系生成的村民类型，作为 `VillagerType#BY_BIOME` 的替代（该字段将在 1.22 中被忽略）。它位于 `neoforge/data_maps/worldgen/biome/villager_types.json`，其对象结构如下：

```json5
{
    // 将在此生物群系中生成的村民类型
    // 如果未为某个生物群系指定村民类型，则将使用 `minecraft:plains`
    "villager_type": "minecraft:desert"
}
```

示例：

```json5
{
    "values": {
        // 让丛林生物群系中的村民变为沙漠类型
        "minecraft:jungle": {
            "villager_type": "minecraft:desert"
        }
    }
}
```

## `neoforge:waxables`

允许配置方块被涂蜡（用蜜脾右击）时将变成的方块，作为 `HoneycombItem#WAXABLES` 的替代。此数据地图也用于构建反向去蜡地图（用于用斧头刮削）。它位于 `neoforge/data_maps/block/waxables.json`，其对象结构如下：

```json5
{
    // 此方块的涂蜡变种
    "waxed": "minecraft:iron_block"
}
```

示例：

```json5
{
    "values": {
        // 让金块涂蜡后变成铁块
        "minecraft:gold_block": {
            "waxed": "minecraft:iron_block"
        }
    }
}
```

[datacomponent]: ../../../items/datacomponents.md
[datamap]: index.md