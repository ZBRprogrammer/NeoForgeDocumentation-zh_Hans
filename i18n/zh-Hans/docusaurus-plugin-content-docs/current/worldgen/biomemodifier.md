import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 生物群系修改器(Biome Modifiers)

生物群系修改器(Biome Modifiers)是一个数据驱动系统，允许更改生物群系的许多方面，包括注入或移除放置特征(PlacedFeatures)、添加或移除生物生成、更改气候以及调整植被和水体颜色。NeoForge 提供了几种默认的生物群系修改器，涵盖了玩家和模组开发者的大多数用例。

### 推荐阅读部分：

- 玩家或整合包开发者：
    - [应用生物群系修改器](#应用生物群系修改器)
    - [内置 NeoForge 生物群系修改器](#内置生物群系修改器)

- 进行简单添加或移除生物群系修改的模组开发者：
    - [应用生物群系修改器](#应用生物群系修改器)
    - [内置 NeoForge 生物群系修改器](#内置生物群系修改器)
    - [生物群系修改器的数据生成](#生物群系修改器的数据生成)
    - [定位可能不存在的生物群系](#定位可能不存在的生物群系)

- 希望进行自定义或复杂生物群系修改的模组开发者：
    - [应用生物群系修改器](#应用生物群系修改器)
    - [创建自定义生物群系修改器](#创建自定义生物群系修改器)
    - [生物群系修改器的数据生成](#生物群系修改器的数据生成)
    - [定位可能不存在的生物群系](#定位可能不存在的生物群系)

## 应用生物群系修改器

要让 NeoForge 将生物群系修改器 JSON 文件加载到游戏中，该文件需要位于模组资源或[数据包][datapacks]的 `data/<modid>/neoforge/biome_modifier/<path>.json` 文件夹下。然后，一旦 NeoForge 加载了生物群系修改器，它将在世界加载时读取其指令并将描述的修改应用到所有目标生物群系。模组中预先存在的生物群系修改器可以通过数据包在完全相同的位置和名称放置新的 JSON 文件来覆盖。

JSON 文件可以手动创建，遵循“[内置 NeoForge 生物群系修改器](#内置生物群系修改器)”部分中的示例，或者如“[生物群系修改器的数据生成](#生物群系修改器的数据生成)”部分所示进行数据生成。

## 内置生物群系修改器

这些生物群系修改器由 NeoForge 注册，供任何人使用。

### 无操作(None)

此生物群系修改器无操作，不会进行任何修改。整合包制作者和玩家可以在数据包中使用它，通过用下面的 JSON 覆盖模组的生物群系修改器 JSON 来禁用模组的生物群系修改器。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:none"
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> NO_OP_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "no_op_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 注册生物群系修改器。
    bootstrap.register(NO_OP_EXAMPLE, NoneBiomeModifier.INSTANCE);
});
```

</TabItem>
</Tabs>

### 添加特征(Add Features)

此生物群系修改器类型将 `PlacedFeature`s（例如树木或矿石）添加到生物群系中，以便它们在世界生成期间生成。修改器接受要添加特征的生物群系的生物群系 ID 或标签、要添加到选定生物群系的 `PlacedFeature` ID 或标签，以及特征将在其中生成的 [`GenerationStep.Decoration`](#可用装饰步骤的值)。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:add_features",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "#namespace:your_biome_tag",
    // 可以是放置特征 ID，例如 "examplemod:add_features_example"，
    // 或放置特征 ID 列表，例如 ["examplemod:add_features_example", "minecraft:ice_spike", ...]，
    // 或放置特征标签，例如 "#examplemod:placed_feature_tag"。
    "features": "namespace:your_feature",
    // 有关有效枚举名称的列表，请参阅代码中的 GenerationStep.Decoration 枚举。
    // 下面的装饰步骤部分也提供了参考值列表。
    "step": "underground_ores"
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一个名为 EXAMPLE_PLACED_FEATURE 的 PlacedFeature。
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> ADD_FEATURES_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_features_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<PlacedFeature> placedFeatures = bootstrap.lookup(Registries.PLACED_FEATURE);

    // 注册生物群系修改器。
    bootstrap.register(ADD_FEATURES_EXAMPLE,
        new AddFeaturesBiomeModifier(
            // 要生成特征的生物群系
            HolderSet.direct(biomes.getOrThrow(Biomes.PLAINS)),
            // 要在生物群系中生成的特征
            HolderSet.direct(placedFeatures.getOrThrow(EXAMPLE_PLACED_FEATURE)),
            // 生成步骤
            GenerationStep.Decoration.LOCAL_MODIFICATIONS
        )
    );
})
```

</TabItem>
</Tabs>

:::warning
向生物群系添加原版 `PlacedFeature`s 时应小心，因为这样做可能会导致所谓的特征循环违规（两个生物群系在其特征列表中具有相同的两个特征，但在同一 `GenerationStep` 中的顺序不同），从而导致崩溃。出于类似原因，不应在多个生物群系修改器中使用相同的 `PlacedFeature`。

原版 `PlacedFeature`s 可以在生物群系 JSON 中引用或通过生物群系修改器添加，但不应同时使用两者。如果仍然需要以这种方式添加它们，将原版 `PlacedFeature` 复制到你自己的命名空间下是避免这些问题的最简单解决方案。
:::

### 移除特征(Remove Features)

此生物群系修改器类型从生物群系中移除特征（例如树木或矿石），以便它们在世界生成期间不再生成。修改器接受要移除特征的生物群系的生物群系 ID 或标签、要从选定生物群系中移除的 `PlacedFeature` ID 或标签，以及将从其中移除特征的 [`GenerationStep.Decoration`](#可用装饰步骤的值)。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:remove_features",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "#namespace:your_biome_tag",
    // 可以是放置特征 ID，例如 "examplemod:add_features_example"，
    // 或放置特征 ID 列表，例如 ["examplemod:add_features_example", "minecraft:ice_spike", ...]，
    // 或放置特征标签，例如 "#examplemod:placed_feature_tag"。
    "features": "namespace:problematic_feature",
    // 可选字段，指定要从中移除特征的 GenerationStep 或 GenerationStep 列表。
    // 如果省略，默认为所有 GenerationStep。
    // 有关有效枚举名称的列表，请参阅代码中的 GenerationStep.Decoration 枚举。
    // 下面的装饰步骤部分也提供了参考值列表。
    "steps": ["underground_ores", "underground_decoration"]
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> REMOVE_FEATURES_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_features_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<PlacedFeature> placedFeatures = bootstrap.lookup(Registries.PLACED_FEATURE);

    // 注册生物群系修改器。
    bootstrap.register(REMOVE_FEATURES_EXAMPLE,
        new RemoveFeaturesBiomeModifier(
            // 要从中移除特征的生物群系
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // 要从生物群系中移除的特征
            HolderSet.direct(placedFeatures.getOrThrow(OrePlacements.ORE_DIAMOND)),
            // 要从中移除的生成步骤
            Set.of(
                GenerationStep.Decoration.LOCAL_MODIFICATIONS,
                GenerationStep.Decoration.UNDERGROUND_ORES
            )
        )
    );
});
```

</TabItem>
</Tabs>

### 添加生成(Add Spawns)

_另请参阅 [生物实体/自然生成][spawning]。_

此生物群系修改器类型向生物群系添加实体生成。修改器接受要添加实体生成的生物群系的生物群系 ID 或标签，以及要添加的实体的 `SpawnerData`。每个 `SpawnerData` 包含实体 ID、生成权重以及每次生成的最小/最大实体数量。

:::note
如果你是添加新实体的模组开发者，请确保实体已向 `RegisterSpawnPlacementsEvent` 注册生成限制。生成限制用于确保实体安全地在地表或水中生成。如果未注册生成限制，你的实体可能会在半空中生成，掉落并死亡。
:::

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:add_spawns",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "#namespace:biome_tag",
    // 可以是单个对象或对象列表。
    "spawners": [
        {
            "type": "namespace:entity_type", // 要生成的实体类型的 ID
            "weight": 100, // 非负整数，生成权重
            "minCount": 1, // 正整数，最小组大小
            "maxCount": 4 // 正整数，最大组大小
        },
        {
            "type": "minecraft:ghast",
            "weight": 1,
            "minCount": 5,
            "maxCount": 10
        }
    ]
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一个名为 EXAMPLE_ENTITY 的 EntityType<?>。
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> ADD_SPAWNS_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_spawns_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);

    // 注册生物群系修改器。
    bootstrap.register(ADD_SPAWNS_EXAMPLE,
        new AddSpawnsBiomeModifier(
            // 要在其中生成生物的生物群系
            HolderSet.direct(biomes.getOrThrow(Biomes.PLAINS)),
            // 要添加的实体的生成器
            List.of(
                new SpawnerData(EXAMPLE_ENTITY, 100, 1, 4),
                new SpawnerData(EntityType.GHAST, 1, 5, 10)
            )
        )
    );
});
```

</TabItem>
</Tabs>

### 移除生成(Remove Spawns)

此生物群系修改器类型从生物群系中移除实体生成。修改器接受要移除实体生成的生物群系的生物群系 ID 或标签，以及要移除的实体的 `EntityType` ID 或标签。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:remove_spawns",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "#namespace:biome_tag",
    // 可以是实体类型 ID，例如 "minecraft:ghast"，
    // 或实体类型 ID 列表，例如 ["minecraft:ghast", "minecraft:skeleton", ...]，
    // 或实体类型标签，例如 "#minecraft:skeletons"。
    "entity_types": "#namespace:entitytype_tag"
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> REMOVE_SPAWNS_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_spawns_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<EntityType<?>> entities = bootstrap.lookup(Registries.ENTITY_TYPE);

    // 注册生物群系修改器。
    bootstrap.register(REMOVE_SPAWNS_EXAMPLE,
        new RemoveSpawnsBiomeModifier(
            // 要从中移除生成的生物群系
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // 要移除其生成的实体
            entities.getOrThrow(EntityTypeTags.SKELETONS)
        )
    );
});
```

</TabItem>
</Tabs>

### 添加生成成本(Add Spawn Costs)

允许向生物群系添加新的生成成本。生成成本是一种较新的方式，用于使生物在生物群系中分散生成以减少聚集。它的工作原理是让实体释放出围绕它们的 `charge`，并与其他实体的 `charge` 累加。当生成新实体时，生成算法会寻找一个位置，该位置的总 `charge` 场乘以生成实体的 `charge` 值小于生成实体的 `energy_budget`。这是一种高级的生物生成方式，因此最好参考灵魂沙峡谷生物群系（这是该系统最突出的用户）以获取现有值作为参考。

修改器接受要添加生成成本的生物群系的生物群系 ID 或标签、要为其添加生成成本的实体类型的 `EntityType` ID 或标签，以及实体的 `MobSpawnSettings.MobSpawnCost`。`MobSpawnCost` 包含能量预算，该预算指示基于每个生成实体提供的电荷，可以在一个位置生成的实体的最大数量。

:::note
如果你是添加新实体的模组开发者，请确保实体已向 `RegisterSpawnPlacementsEvent` 注册生成限制。
:::

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:add_spawn_costs",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "#namespace:biome_tag",
    // 可以是实体类型 ID，例如 "minecraft:ghast"，
    // 或实体类型 ID 列表，例如 ["minecraft:ghast", "minecraft:skeleton", ...]，
    // 或实体类型标签，例如 "#minecraft:skeletons"。
    "entity_types": "#minecraft:skeletons",
    "spawn_cost": {
        // 能量预算
        "energy_budget": 1.0,
        // 每个实体从预算中占用的电荷量
        "charge": 0.1
    }
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> ADD_SPAWN_COSTS_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_spawn_costs_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<EntityType<?>> entities = bootstrap.lookup(Registries.ENTITY_TYPE);

    // 注册生物群系修改器。
    bootstrap.register(ADD_SPAWN_COSTS_EXAMPLE,
        new AddSpawnCostsBiomeModifier(
            // 要添加生成成本的生物群系
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // 要为其添加生成成本的实体
            entities.getOrThrow(EntityTypeTags.SKELETONS),
            new MobSpawnSettings.MobSpawnCost(
                1.0, // 能量预算
                0.1  // 每个实体从预算中占用的电荷量
            )
        )
    );
});
```

</TabItem>
</Tabs>

### 移除生成成本(Remove Spawn Costs)

允许从生物群系中移除生成成本。生成成本是一种较新的方式，用于使生物在生物群系中分散生成以减少聚集。修改器接受要移除生成成本的生物群系的生物群系 ID 或标签，以及要移除其生成成本的实体的 `EntityType` ID 或标签。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:remove_spawn_costs",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "#namespace:biome_tag",
    // 可以是实体类型 ID，例如 "minecraft:ghast"，
    // 或实体类型 ID 列表，例如 ["minecraft:ghast", "minecraft:skeleton", ...]，
    // 或实体类型标签，例如 "#minecraft:skeletons"。
    "entity_types": "#minecraft:skeletons"
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> REMOVE_SPAWN_COSTS_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_spawn_costs_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<EntityType<?>> entities = bootstrap.lookup(Registries.ENTITY_TYPE);

    // 注册生物群系修改器。
    bootstrap.register(REMOVE_SPAWN_COSTS_EXAMPLE,
        new RemoveSpawnCostsBiomeModifier(
            // 要从中移除生成成本的生物群系
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // 要移除其生成成本的实体
            entities.getOrThrow(EntityTypeTags.SKELETONS)
        )
    );
});
```

</TabItem>
</Tabs>

### 添加旧版雕刻器(Add Legacy Carvers)

此生物群系修改器类型允许向生物群系添加雕刻器洞穴和峡谷。这些是在洞穴与山崖更新之前用于洞穴生成的内容。它无法向生物群系添加噪声洞穴，因为噪声洞穴是某些基于噪声的区块生成器系统的一部分，实际上并不与生物群系绑定。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
    {
    "type": "neoforge:add_carvers",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "minecraft:plains",
    // 可以是雕刻器 ID，例如 "examplemod:add_carvers_example"，
    // 或雕刻器 ID 列表，例如 ["examplemod:add_carvers_example", "minecraft:canyon", ...]，
    // 或雕刻器标签，例如 "#examplemod:configured_carver_tag"。
    "carvers": "examplemod:add_carvers_example"
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 假设我们有一个名为 EXAMPLE_CARVER 的 ConfiguredWorldCarver。
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> ADD_CARVERS_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_carvers_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<ConfiguredWorldCarver<?>> carvers = bootstrap.lookup(Registries.CONFIGURED_CARVER);

    // 注册生物群系修改器。
    bootstrap.register(ADD_CARVERS_EXAMPLE,
        new AddCarversBiomeModifier(
            // 要在其中生成雕刻器的生物群系
            HolderSet.direct(biomes.getOrThrow(Biomes.PLAINS)),
            // 要在生物群系中生成的雕刻器
            HolderSet.direct(carvers.getOrThrow(EXAMPLE_CARVER))
        )
    );
});
```

</TabItem>
</Tabs>

### 移除旧版雕刻器(Removing Legacy Carvers)

此生物群系修改器类型允许从生物群系中移除雕刻器洞穴和峡谷。这些是在洞穴与山崖更新之前用于洞穴生成的内容。它无法从生物群系中移除噪声洞穴，因为噪声洞穴是维度噪声设置系统的一部分，实际上并不与生物群系绑定。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    "type": "neoforge:remove_carvers",
    // 可以是生物群系 ID，例如 "minecraft:plains"，
    // 或生物群系 ID 列表，例如 ["minecraft:plains", "minecraft:badlands", ...]，
    // 或生物群系标签，例如 "#c:is_overworld"。
    "biomes": "minecraft:plains",
    // 可以是雕刻器 ID，例如 "examplemod:add_carvers_example"，
    // 或雕刻器 ID 列表，例如 ["examplemod:add_carvers_example", "minecraft:canyon", ...]，
    // 或雕刻器标签，例如 "#examplemod:configured_carver_tag"。
    "carvers": "examplemod:add_carvers_example"
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> REMOVE_CARVERS_EXAMPLE = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_carvers_example") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<ConfiguredWorldCarver<?>> carvers = bootstrap.lookup(Registries.CONFIGURED_CARVER);

    // 注册生物群系修改器。
    bootstrap.register(REMOVE_CARVERS_EXAMPLE,
        new AddFeaturesBiomeModifier(
            // 要从中移除雕刻器的生物群系
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // 要从生物群系中移除的雕刻器
            HolderSet.direct(carvers.getOrThrow(Carvers.CAVE))
        )
    );
});
```

</TabItem>
</Tabs>

### 可用装饰步骤的值

上述许多 JSON 中的 `step` 或 `steps` 字段指的是 `GenerationStep.Decoration` 枚举。该枚举包含以下步骤，这些步骤的顺序与游戏在世界生成期间使用的顺序相同。尝试将特征放在对它们最有意义的步骤中。

|           步骤           | 描述                                                                             |
|:------------------------:|:----------------------------------------------------------------------------------------|
|     `raw_generation`     | 首先运行。用于特殊地形特征，例如小型末地岛屿。 |
|         `lakes`          | 专用于生成类似池塘的特征，例如熔岩湖。                             |
|  `local_modifications`   | 用于地形修改，例如晶洞、冰山、巨石或滴水石。          |
| `underground_structures` | 用于小型地下结构特征，例如地牢或化石。         |
|   `surface_structures`   | 用于小型仅地表结构特征，例如沙漠水井。                    |
|      `strongholds`       | 专用于要塞结构。在未修改的 Minecraft 中没有特征添加到这里。  |
|    `underground_ores`    | 所有矿石和矿脉添加到的步骤。这包括金、泥土、花岗岩等。 |
| `underground_decoration` | 通常用于装饰洞穴。滴水石簇和幽匿脉络在此处。         |
|     `fluid_springs`      | 小型熔岩瀑布和水瀑布来自此阶段的特征。                    |
|   `vegetal_decoration`   | 几乎所有植物（花、树、藤蔓等）都添加到此阶段。            |
| `top_layer_modification` | 最后运行。用于在寒冷生物群系表面放置雪和冰。               |

## 创建自定义生物群系修改器

### `BiomeModifier` 实现

在底层，生物群系修改器由三部分组成：

- [数据包注册的][datareg] `BiomeModifier`，用于修改生物群系构建器。
- [静态注册的][staticreg] `MapCodec`，用于编码和解码修改器。
- 构建 `BiomeModifier` 的 JSON，使用 `MapCodec` 的注册 ID 作为可索引类型。

`BiomeModifier` 包含两个方法：`#modify` 和 `#codec`。`modify` 接受当前 `Biome` 的 `Holder`、当前的 `BiomeModifier.Phase`，以及要修改的生物群系的构建器。每个 `BiomeModifier` 在每个 `Phase` 中被调用一次，以组织何时应对生物群系进行某些修改：

| 阶段               | 描述                                                              |
|:-------------------:|:-------------------------------------------------------------------------|
| `BEFORE_EVERYTHING` | 用于需要在标准阶段之前运行的所有内容的通用阶段。 |
| `ADD`               | 添加特征、生物生成等。                                        |
| `REMOVE`            | 移除特征、生物生成等。                                      |
| `MODIFY`            | 修改单个值（例如气候、颜色）。                         |
| `AFTER_EVERYTHING`  | 用于需要在标准阶段之后运行的所有内容的通用阶段。  |

所有 `BiomeModifier` 都包含一个 `type` 键，该键引用用于 `BiomeModifier` 的 `MapCodec` 的 ID。`codec` 接受用于编码和解码修改器的 `MapCodec`。此 `MapCodec` 是[静态注册的][staticreg]，其 ID 用作 `BiomeModifier` 的 `type`。

```java
public record ExampleBiomeModifier(HolderSet<Biome> biomes, int value) implements BiomeModifier {
    
    @Override
    public void modify(Holder<Biome> biome, Phase phase, ModifiableBiomeInfo.BiomeInfo.Builder builder) {
        if (phase == /* 选择最符合你要修改内容的阶段 */) {
            // 修改 'builder'，检查关于生物群系本身的任何信息
        }
    }

    @Override
    public MapCodec<? extends BiomeModifier> codec() {
        return EXAMPLE_BIOME_MODIFIER.get();
    }
}

// 在某个注册类中
private static final DeferredRegister<MapCodec<? extends BiomeModifier>> BIOME_MODIFIERS =
    DeferredRegister.create(NeoForgeRegistries.Keys.BIOME_MODIFIER_SERIALIZERS, MOD_ID);

public static final Supplier<MapCodec<ExampleBiomeModifier>> EXAMPLE_BIOME_MODIFIER =
    BIOME_MODIFIERS.register("example_biome_modifier", () -> RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            Biome.LIST_CODEC.fieldOf("biomes").forGetter(ExampleBiomeModifier::biomes),
            Codec.INT.fieldOf("value").forGetter(ExampleBiomeModifier::value)
        ).apply(instance, ExampleBiomeModifier::new)
    ));
```

## 生物群系修改器的数据生成

可以通过[数据生成][datagen]创建 `BiomeModifier` JSON，方法是将 `RegistrySetBuilder` 传递给 `DatapackBuiltinEntriesProvider`。JSON 将放置在 `data/<modid>/neoforge/biome_modifier/<path>.json`。

有关 `RegistrySetBuilder` 和 `DatapackBuiltinEntriesProvider` 如何工作的更多信息，请参阅[数据包注册表的数据生成][datapackdatagen]文章。

```java
// 为我们的 BiomeModifier 定义 ResourceKey。
public static final ResourceKey<BiomeModifier> EXAMPLE_MODIFIER = ResourceKey.create(
    NeoForgeRegistries.Keys.BIOME_MODIFIERS, // 此键所属的注册表
    ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_modifier") // 注册表名称
);

// BUILDER 是传递给 DatapackBuiltinEntriesProvider 的 RegistrySetBuilder，
// 在 `GatherDataEvent`s 的监听器中。
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // 查找任何必要的注册表。
    // 静态注册表仅在需要获取标签数据时才需要查找。
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);

    // 注册生物群系修改器。
    bootstrap.register(EXAMPLE_MODIFIER,
        new ExampleBiomeModifier(
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            20
        )
    );
});
```

这将导致创建以下 JSON：

```json5
// 在 data/examplemod/neoforge/biome_modifier/example_modifier.json
{
    // 修改器的 MapCodec 的注册表键
    "type": "examplemod:example_biome_modifier",
    // 所有附加设置都应用于根对象
    "biomes": "#c:is_overworld",
    "value": 20
}
```

## 定位可能不存在的生物群系

有时，生物群系修改器可能需要定位游戏中并不总是存在的生物群系。如果生物群系修改器直接定位未注册的生物群系，将在世界加载时崩溃。解决方法是创建一个生物群系标签，并将目标生物群系添加为可选标签条目，方法是将条目的 required 设置为 false。示例如下：

```json5
{
    "replace": false,
    "values": [
        {
            "id": "minecraft:pale_garden",
            "required": false
        }
    ]
}
```

现在，使用该生物群系标签进行生物群系修改器将不会在生物群系未注册时崩溃。一个这样的用例是苍白花园生物群系，该生物群系仅在 1.21.3 版本中打开冬季掉落数据包时创建。否则，该生物群系根本不存在于生物群系注册表中。另一个用例可以是定位模组生物群系，同时在添加这些生物群系的模组不存在时仍然能正常工作。

要为生物群系标签数据生成可选条目，数据生成代码将类似于以下内容：

```java
// 在 KeyTagProvider<Biome> 子类中
// 假设我们有一些示例 TagKey<Biome> OPTIONAL_BIOMES_TAG
@Override
protected void addTags(HolderLookup.Provider registries) {
    this.tag(OPTIONAL_BIOMES_TAG)
        // 必须是 ResourceKey<Biome>
        .addOptional(Biomes.PALE_GARDEN);
}
```

[datagen]: ../resources/index.md#data-generation
[datapackdatagen]: ../concepts/registries#data-generation-for-datapack-registries
[datapacks]: ../resources/index.md#data
[datareg]: ../concepts/registries.md#datapack-registries
[spawning]: ../entities/livingentity.md#natural-spawning
[staticreg]: ../concepts/registries.md#methods-for-registering