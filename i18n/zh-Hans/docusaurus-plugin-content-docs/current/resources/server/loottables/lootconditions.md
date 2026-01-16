# 战利品条件

战利品条件可用于检查[战利品条目][entry]或[战利品池][pool]是否应在当前上下文中使用。在这两种情况下，都会定义一个条件列表；只有当所有条件都通过时，才会使用该条目或池。在数据生成期间，通过使用所需条件的实例调用 `#when` 将它们添加到 `LootPoolEntryContainer.Builder<?>` 或 `LootPool.Builder`。本文将概述可用的战利品条件。要创建自己的战利品条件，请参见[自定义战利品条件][custom]。

## `minecraft:inverted`

此条件接受另一个条件并反转其结果。需要另一个条件所需的任何战利品参数。

```json5
{
    "condition": "minecraft:inverted",
    "term": {
        // 其他一些战利品条件。
    }
}
```

在数据生成期间，调用 `InvertedLootItemCondition#invert` 并传入要反转的条件以构建此条件的构建器。

## `minecraft:all_of`

此条件接受任意数量的其他条件，并且仅当所有子条件都返回 true 时才返回 true。如果列表为空，则返回 false。需要其他条件所需的任何战利品参数。

```json5
{
    "condition": "minecraft:all_of",
    "terms": [
        {
            // 一个战利品条件。
        },
        {
            // 另一个战利品条件。
        },
        {
            // 又一个战利品条件。
        }
    ]
}
```

在数据生成期间，调用 `AllOfCondition#allOf` 并传入所需的条件以构建此条件的构建器。

## `minecraft:any_of`

此条件接受任意数量的其他条件，并且只要至少一个子条件返回 true 就返回 true。如果列表为空，则返回 false。需要其他条件所需的任何战利品参数。

```json5
{
    "condition": "minecraft:any_of",
    "terms": [
        {
            // 一个战利品条件。
        },
        {
            // 另一个战利品条件。
        },
        {
            // 又一个战利品条件。
        }
    ]
}
```

在数据生成期间，调用 `AnyOfCondition#anyOf` 并传入所需的条件以构建此条件的构建器。

## `minecraft:random_chance`

此条件接受一个代表 0 到 1 之间几率的[数字提供器][numberprovider]，并根据该几率随机返回 true 或 false。数字提供器通常不应返回区间 `[0, 1]` 之外的值。

```json5
{
    "condition": "minecraft:random_chance",
    // 条件应用的恒定 50% 几率。
    "chance": 0.5
}
```

在数据生成期间，调用 `LootItemRandomChanceCondition#randomChance` 并传入数字提供器或一个（常量）浮点值以构建此条件的构建器。

## `minecraft:random_chance_with_enchanted_bonus`

此条件接受一个附魔 id、一个 [`LevelBasedValue`][numberprovider] 和一个常量回退浮点值。如果指定的附魔存在，则查询 `LevelBasedValue` 获取一个值。如果指定的附魔不存在，或者无法从 `LevelBasedValue` 检索到值，则使用常量回退值。然后条件随机返回 true 或 false，之前确定的值表示返回 true 的几率。需要 `minecraft:attacking_entity` 参数，如果不存在则回退到等级 0。

```json5
{
    "condition": "minecraft:random_chance_with_enchanted_bonus",
    // 每级抢夺增加 20% 的成功几率。
    "enchantment": "minecraft:looting",
    "enchanted_chance": {
        "type": "linear",
        "base": 0.2,
        "per_level_above_first": 0.2
    },
    // 如果不存在抢夺附魔，则总是失败。
    "unenchanted_chance": 0.0
}
```

在数据生成期间，调用 `LootItemRandomChanceWithEnchantedBonusCondition#randomChanceAndLootingBoost` 并传入注册表查找器（`HolderLookup.Provider`）、基础值和每级增加值以构建此条件的构建器。或者，调用 `new LootItemRandomChanceWithEnchantedBonusCondition` 以进一步指定值。

## `minecraft:value_check`

此条件接受一个[数字提供器][numberprovider]和一个 `IntRange`，如果数字提供器的结果在范围内则返回 true。

```json5
{
    "condition": "minecraft:value_check",
    // 可以是任何数字提供器。
    "value": {
        "type": "minecraft:uniform",
        "min": 0.0,
        "max": 10.0
    },
    // 带有最小/最大值的范围。
    "range": {
        "min": 2.0,
        "max": 5.0
    }
}
```

在数据生成期间，调用 `ValueCheckCondition#hasValue` 并传入数字提供器和范围以构建此条件的构建器。

## `minecraft:time_check`

此条件检查世界时间是否在 `IntRange` 内。可以选择提供一个 `period` 参数来对时间取模；这可用于例如检查时间（如果 `period` 是 24000，一个游戏内的白天/黑夜周期有 24000 游戏刻）。

```json5
{
    "condition": "minecraft:time_check",
    // 可选，可以省略。如果省略，将不进行取模操作。
    // 我们在此使用 24000，这是一个游戏内白天/黑夜周期的长度。
    "period": 24000,
    // 带有最小/最大值的范围。此示例检查时间是否在 0 和 12000 之间。
    // 结合上面指定的 24000 取模操作数，此示例检查当前是否是白天。
    "value": {
        "min": 0,
        "max": 12000
    }
}
```

在数据生成期间，调用 `TimeCheck#time` 并传入所需范围以构建此条件的构建器。然后可以在构建器上使用 `#setPeriod` 设置 `period` 值。

## `minecraft:weather_check`

此条件检查当前的天气是否在下雨和打雷。

```json5
{
    "condition": "minecraft:weather_check",
    // 可选。如果未指定，将不检查下雨状态。
    "raining": true,
    // 可选。如果未指定，将不检查打雷状态。
    // 指定 "raining": true 和 "thundering": true 在功能上等同于仅指定
    // "thundering": true，因为打雷时总是在下雨。
    "thundering": false
}
```

在数据生成期间，调用 `WeatherCheck#weather` 以构建此条件的构建器。然后可以在构建器上使用 `#setRaining` 和 `#setThundering` 分别设置 `raining` 和 `thundering` 值。

## `minecraft:location_check`

此条件接受一个 `LocationPredicate` 和每个轴方向的可选偏移值。`LocationPredicate` 允许检查条件，例如位置本身、该位置的方块或流体状态、该位置的维度、生物群系或结构、光照等级、天空是否可见等。所有可能的值可以在 `LocationPredicate` 类定义中查看。需要 `minecraft:origin` 战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "minecraft:location_check",
    "predicate": {
        // 如果我们的目标在下界任何地方则成功。
        "dimension": "the_nether"
    },
    // 可选的位置偏移值。仅当以某种方式检查位置时才相关。
    // 必须同时全部提供，或者完全不提供。
    "offsetX": 10,
    "offsetY": 10,
    "offsetZ": 10
}
```

在数据生成期间，调用 `LocationCheck#checkLocation` 并传入 `LocationPredicate` 和可选的 `BlockPos` 以构建此条件的构建器。

## `minecraft:block_state_property`

此条件检查破碎的方块状态中指定的方块状态属性是否具有指定的值。需要 `minecraft:block_state` 战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "minecraft:block_state_property",
    // 期望的方块。如果这与实际破碎的方块不匹配，则条件失败。
    "block": "minecraft:oak_slab",
    // 要匹配的方块状态属性。未指定的属性可以具有任意值。
    // 在此示例中，我们只想在打破一个顶部台阶（无论是否含水）时成功。
    // 如果此指定了方块上不存在的属性，将打印日志警告。
    "properties": {
        "type": "top"
    }
}
```

在数据生成期间，调用 `LootItemBlockStatePropertyCondition#hasBlockStateProperties` 并传入方块以构建此条件的构建器。然后可以在构建器上使用 `#setProperties` 设置所需的方块状态属性值。

## `minecraft:survives_explosion`

此条件随机销毁掉落物。掉落物存活的几率是 1 / `explosion_radius` 战利品参数。此函数被所有方块掉落物使用，极少数例外，如信标或龙蛋。需要 `minecraft:explosion_radius` 战利品参数，如果该参数不存在则总是成功。

```json5
{
    "condition": "minecraft:survives_explosion"
}
```

在数据生成期间，调用 `ExplosionCondition#survivesExplosion` 以构建此条件的构建器。

## `minecraft:match_tool`

此条件接受一个 `ItemPredicate`，该谓词针对 `tool` 战利品参数进行检查。`ItemPredicate` 可以指定一个有效物品 id 列表（`items`）、物品数量的最小/最大范围（`count`）、一个 `DataComponentPredicate`（`components`）和一个 `ItemSubPredicate`s 映射（`predicates`）；所有字段都是可选的。需要 `minecraft:tool` 战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "minecraft:match_tool",
    // 匹配下界合金镐或斧。
    "predicate": {
        "items": [
            "minecraft:netherite_pickaxe",
            "minecraft:netherite_axe"
        ]
    }
}
```

在数据生成期间，调用 `MatchTool#toolMatches` 并传入一个 `ItemPredicate.Builder` 以反转来构建此条件的构建器。

## `minecraft:enchantment_active`

此条件返回附魔是否处于激活状态。需要 `minecraft:enchantment_active` 战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "minecraft:enchantment_active",
    // 附魔应该是激活的（true）还是不激活的（false）。
    "active": true
}
```

在数据生成期间，调用 `EnchantmentActiveCheck#enchantmentActiveCheck` 或 `#enchantmentInactiveCheck` 以构建此条件的构建器。

## `minecraft:table_bonus`

此条件类似于 `minecraft:random_chance_with_enchanted_bonus`，但使用固定值而不是随机值。需要 `minecraft:tool` 战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "minecraft:table_bonus",
    // 如果存在时运附魔则应用奖励。
    "enchantment": "minecraft:fortune",
    // 每级使用的几率。此示例在未附魔时有 20% 的几率成功，
    // 在 1 级附魔时为 30%，在 2 级或以上附魔时为 60%。
    "chances": [0.2, 0.3, 0.6]
}
```

在数据生成期间，调用 `BonusLevelTableCondition#bonusLevelFlatChance` 并传入附魔 id 和几率以构建此条件的构建器。

## `minecraft:entity_properties`

此条件针对一个[实体目标][entitytarget]检查给定的 `EntityPredicate`。`EntityPredicate` 可以检查实体类型、状态效果、NBT 值、装备、位置等。

```json5
{
    "condition": "minecraft:entity_properties",
    // 要使用的实体目标。有效值是 "this"、"attacker"、"direct_attacker" 或 "attacking_player"。
    // 这些分别对应于 "this_entity"、"attacking_entity"、"direct_attacking_entity" 和
    // "last_damage_player" 战利品参数。
    "entity": "attacker",
    // 仅当目标是猪时才成功。谓词也可以为空，这可以用于
    // 检查指定的实体目标是否被设置。
    "predicate": {
        "type": "minecraft:pig"
    }
}
```

在数据生成期间，调用 `LootItemEntityPropertyCondition#entityPresent` 并传入实体目标，或调用 `LootItemEntityPropertyCondition#hasProperties` 并传入实体目标和 `EntityPredicate`，以构建此条件的构建器。

## `minecraft:damage_source_properties`

此条件针对伤害源战利品参数检查给定的 `DamageSourcePredicate`。需要 `minecraft:origin` 和 `minecraft:damage_source` 战利品参数，如果这些参数不存在则总是失败。

```json5
{
    "condition": "minecraft:damage_source_properties",
    "predicate": {
        // 检查源实体是否是僵尸。
        "source_entity": {
            "type": "zombie"
        }
    }
}
```

在数据生成期间，调用 `DamageSourceCondition#hasDamageSource` 并传入一个 `DamageSourcePredicate.Builder` 以构建此条件的构建器。

## `minecraft:killed_by_player`

此条件确定击杀是否是玩家击杀。被一些实体掉落物使用，例如烈焰人掉落的烈焰棒。需要 `minecraft:last_player_damage` 战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "minecraft:killed_by_player"
}
```

在数据生成期间，调用 `LootItemKilledByPlayerCondition#killedByPlayer` 以构建此条件的构建器。

## `minecraft:entity_scores`

此条件检查[实体目标][entitytarget]的计分板。需要与指定实体目标对应的战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "minecraft:entity_scores"
    // 要使用的实体目标。有效值是 "this"、"attacker"、"direct_attacker" 或 "attacking_player"。
    // 这些分别对应于 "this_entity"、"attacking_entity"、"direct_attacking_entity" 和
    // "last_damage_player" 战利品参数。
    "entity": "attacker",
    // 必须在给定范围内的计分板值列表。
    "scores": {
        "score1": {
            "min": 0,
            "max": 100
        },
        "score2": {
            "min": 10,
            "max": 20
        }
    }
}
```

在数据生成期间，调用 `EntityHasScoreCondition#hasScores` 并传入一个实体目标以构建此条件的构建器。然后，使用 `#withScore` 将所需的分数添加到构建器。

## `minecraft:reference`

此条件引用一个谓词文件并返回其结果。更多信息请参见[物品谓词][predicate]。

```json5
{
    "condition": "minecraft:reference",
    // 引用 data/examplemod/predicate/example_predicate.json 处的谓词文件。
    "name": "examplemod:example_predicate"
}
```

在数据生成期间，调用 `ConditionReference#conditionReference` 并传入引用的谓词文件的 id 以构建此条件的构建器。

## `neoforge:loot_table_id`

此条件仅在周围的战利品表 id 匹配时才返回 true。这通常在[全局战利品修改器 (GLMs)][glm] 内部使用。

```json5
{
    "condition": "neoforge:loot_table_id",
    // 仅当战利品表是用于泥土时才应用
    "loot_table_id": "minecraft:blocks/dirt"
}
```

在数据生成期间，调用 `LootTableIdCondition#builder` 并传入所需的战利品表 id 以构建此条件的构建器。

## `neoforge:can_item_perform_ability`

此条件仅当 `tool` 战利品上下文参数（`LootContextParams.TOOL`）中的物品（通常是用于破坏方块或杀死实体的物品）可以执行指定的 [`ItemAbility`][itemability] 时才返回 true。需要 `minecraft:tool` 战利品参数，如果该参数不存在则总是失败。

```json5
{
    "condition": "neoforge:can_item_perform_ability",
    // 仅当工具可以像斧头一样剥去皮时应用
    "ability": "axe_strip"
}
```

在数据生成期间，调用 `CanItemPerformAbility#canItemPerformAbility` 并传入所需物品能力的 id 以构建此条件的构建器。

## 另请参阅

- [Minecraft Wiki][mcwiki] 上的[物品谓词][predicatejson]

[custom]: custom.md#custom-loot-conditions
[entitytarget]: index.md#entity-targets
[entry]: index.md#loot-entry
[glm]: glm.md
[itemability]: ../../../items/tools.md#itemabilitys
[mcwiki]: https://minecraft.wiki
[numberprovider]: index.md#number-provider
[pool]: index.md#loot-pool
[predicate]: https://minecraft.wiki/w/Predicate
[predicatejson]: https://minecraft.wiki/w/Predicate#JSON_format
[registry]: ../../../concepts/registries.md