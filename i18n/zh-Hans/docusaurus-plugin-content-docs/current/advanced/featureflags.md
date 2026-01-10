# 功能标志(Feature Flags)

功能标志(Feature Flags)是一个允许开发者将一组功能置于某些所需标志之后的系统，这些标志可以是已注册的元素、游戏机制、数据包条目或您模组中其他独特的系统。

一个常见的用例是将实验性功能/元素置于实验标志之后，允许用户在功能最终确定前轻松切换并试用它们。

:::tip
您并非必须添加自己的标志。如果您找到一个符合您用例的原版(vanilla)标志，可以自由地用该标志标记您的方块/物品/实体等。

例如，在`1.21.3`中，如果您要添加到苍白橡木(Pale Oak)木方块系列，您可能只希望在`WINTER_DROP`标志启用时才显示这些方块。
:::

## 创建功能标志(Feature Flag)

要创建新的功能标志(Feature Flags)，需要创建一个JSON文件，并在`neoforge.mods.toml`文件中通过`[[mods]]`块内的`featureFlags`条目引用。指定的路径必须相对于`resources`目录：

```
# In neoforge.mods.toml:
[[mods]]
    # The file is relative to the output directory of the resources, or the root path inside the jar when compiled
    # The 'resources' directory represents the root output directory of the resources
    featureFlags="META-INF/feature_flags.json"
```

条目的定义包括一个功能标志(Feature Flag)名称列表，这些名称将在游戏初始化期间加载和注册。

```
{
    "flags": [
        // Identifier of a Feature flag to be registered
        "examplemod:experimental"
    ]
}
```

## 获取功能标志(Retrieving the Feature Flag)

可以通过`FeatureFlagRegistry.getFlag(ResourceLocation)`获取已注册的功能标志(Feature Flag)。这可以在您模组初始化的任何时间完成，建议将标志存储在某个地方以备将来使用，而不是每次需要时都查找注册表。

```
// Look up the 'examplemod:experimental' Feature flag
public static final FeatureFlag EXPERIMENTAL = FeatureFlags.REGISTRY.getFlag(ResourceLocation.fromNamespaceAndPath("examplemod", "experimental"));
```

## 功能元素(Feature Elements)

`FeatureElement`是可以被赋予一组所需标志的注册表值。仅当相应的所需标志与世界中启用的标志匹配时，这些值才对玩家可用。

当一个功能元素(Feature Element)被禁用时，它将完全隐藏于玩家的视野，所有交互都将被跳过。请注意，这些被禁用的元素仍将存在于注册表中，只是在功能上无法使用。

以下是直接实现功能元素(Feature Element)系统的所有注册表的完整列表：

- 物品(Item)
- 方块(Block)
- 实体类型(EntityType)
- 菜单类型(MenuType)
- 药水(Potion)
- 状态效果(MobEffect)

### 标记元素(Flagging Elements)

为了将给定的`FeatureElement`标记为需要您的功能标志(Feature Flag)，您只需将其和任何其他所需的标志传递到相应的注册方法中：

- `物品(Item)`: `Item.Properties#requiredFeatures`
- `方块(Block)`: `BlockBehaviour.Properties#requiredFeatures`
- `实体类型(EntityType)`: `EntityType.Builder#requiredFeatures`
- `菜单类型(MenuType)`: `MenuType#new`
- `药水(Potion)`: `Potion#requiredFeatures`
- `状态效果(MobEffect)`: `MobEffect#requiredFeatures`

```
// These elements will only become available once the 'EXPERIMENTAL' flag is enabled

// Item
DeferredRegister.Items ITEMS = DeferredRegister.createItems("examplemod");
DeferredItem<Item> EXPERIMENTAL_ITEM = ITEMS.registerSimpleItem("experimental", new Item.Properties()
    .requiredFeatures(EXPERIMENTAL) // mark as requiring the 'EXPERIMENTAL' flag
);

// Block
DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("examplemod");
// Do note that BlockBehaviour.Properties#ofFullCopy and BlockBehaviour.Properties#ofLegacyCopy will copy over the required features.
// This means that in 1.21.3, using BlockBehaviour.Properties.ofFullCopy(Blocks.PALE_OAK_WOOD) would have your block require the 'WINTER_DROP' flag.
DeferredBlock<Block> EXPERIMENTAL_BLOCK = BLOCKS.registerSimpleBlock("experimental", BlockBehaviour.Properties.of()
    .requiredFeatures(EXPERIMENTAL) // mark as requiring the 'EXPERIMENTAL' flag
);

// BlockItems are special in that the required features are inherited from their respective Blocks.
// The same is also true for spawn eggs and their respective EntityTypes.
DeferredItem<BlockItem> EXPERIMENTAL_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(EXPERIMENTAL_BLOCK);

// EntityType
DeferredRegister<EntityType<?>> ENTITY_TYPES = DeferredRegister.create(Registries.ENTITY_TYPE, "examplemod");
DeferredHolder<EntityType<?>, EntityType<ExperimentalEntity>> EXPERIMENTAL_ENTITY = ENTITY_TYPES.register("experimental", registryName -> EntityType.Builder.of(ExperimentalEntity::new, MobCategory.AMBIENT)
    .requiredFeatures(EXPERIMENTAL) // mark as requiring the 'EXPERIMENTAL' flag
    .build(ResourceKey.create(Registries.ENTITY_TYPE, registryName))
);

// MenuType
DeferredRegister<MenuType<?>> MENU_TYPES = DeferredRegister.create(Registries.MENU, "examplemod");
DeferredHolder<MenuType<?>, MenuType<ExperimentalMenu>> EXPERIMENTAL_MENU = MENU_TYPES.register("experimental", () -> new MenuType<>(
    // Using vanilla's MenuSupplier:
    // This is used when your menu is not encoding complex data during `player.openMenu`. Example:
    // (windowId, inventory) -> new ExperimentalMenu(windowId, inventory),

    // Using NeoForge's IContainerFactory:
    // This is used when you wish to read complex data encoded during `player.openMenu`.
    // Casting is important here, as `MenuType` specifically expects a `MenuSupplier`.
    (IContainerFactory<ExperimentalMenu>) (windowId, inventory, buffer) -> new ExperimentalMenu(windowId, inventory, buffer),
    
    FeatureFlagSet.of(EXPERIMENTAL) // mark as requiring the 'EXPERIMENTAL' flag
));

// MobEffect
DeferredRegister<MobEffect> MOB_EFFECTS = DeferredRegister.create(Registries.MOB_EFFECT, "examplemod");
DeferredHolder<MobEffect, ExperimentalMobEffect> EXPERIMENTAL_MOB_EEFECT = MOB_EFFECTS.register("experimental", registryName -> new ExperimentalMobEffect(MobEffectCategory.NEUTRAL, CommonColors.WHITE)
    .requiredFeatures(EXPERIMENTAL) // mark as requiring the 'EXPERIMENTAL' flag
);

// Potion
DeferredRegister<Potion> POTIONS = DeferredRegister.create(Registries.POTION, "examplemod");
DeferredHolder<Potion, ExperimentalPotion> EXPERIMENTAL_POTION = POTIONS.register("experimental", registryName -> new ExperimentalPotion(registryName.toString(), new MobEffectInstance(EXPERIMENTAL_MOB_EEFECT))
    .requiredFeatures(EXPERIMENTAL) // mark as requiring the 'EXPERIMENTAL' flag
);
```

### 验证启用状态(Validating Enabled Status)

要验证功能是否应启用，您必须首先获取已启用功能的集合。这可以通过多种方式完成，但常见且推荐的方法是`LevelReader#enabledFeatures`。

```
level.enabledFeatures(); // from a 'LevelReader' instance
entity.level().enabledFeatures(); // from a 'Entity' instance

// Client Side
minecraft.getConnection().enabledFeatures();

// Server Side
server.getWorldData().enabledFeatures();
```

要验证任何`FeatureFlagSet`是否启用，您可以将已启用的功能传递给`FeatureFlagSet#isSubsetOf`；要验证特定的`FeatureElement`是否启用，可以调用`FeatureElement#isEnabled`。

:::note
`ItemStack`有一个特殊的`isItemEnabled(FeatureFlagSet)`方法。这是为了即使基础`Item`所需的功能与启用的功能不匹配，空堆栈也被视为已启用。建议在可能的情况下优先使用此方法而非`Item#isEnabled`。
:::

```
requiredFeatures.isSubsetOf(enabledFeatures);
featureElement.isEnabled(enabledFeatures);
itemStack.isItemEnabled(enabledFeatures);
```

## 功能包(Feature Packs)

_另请参阅：[资源包](../resources/index.md#assets)、[数据包](../resources/index.md#data) 和 [Pack.mcmeta](../resources/index.md#packmcmeta)_

功能包(Feature Packs)是一种不仅能加载资源和/或数据，还能启用一组给定功能标志(Feature Flags)的包类型。这些标志在此包根目录的`pack.mcmeta` JSON文件中定义，格式如下：

:::note
此文件与您模组`resources/`目录中的文件不同。此文件定义了一个全新的功能包，因此必须位于其自己的文件夹中。
:::

```
{
    "features": {
        "enabled": [
            // Identifier of a Feature flag to be enabled
            // Must be a valid registered flag
            "examplemod:experimental"
        ]
    },
    "pack": { /*...*/ }
}
```

用户有几种方式可以获取功能包(Feature Pack)，即从外部源作为数据包安装，或下载具有内置功能包的模组。这两种方式都需要根据[物理端(physical side)][../concepts/sides.md]进行不同的安装。

### 内置(Built-In)

内置包与您的模组捆绑在一起，并通过`AddPackFindersEvent`事件提供给游戏。

```
@SubscribeEvent // on the mod event bus
public static void addFeaturePacks(final AddPackFindersEvent event) {
    event.addPackFinders(
            // Path relative to your mods 'resources' pointing towards this pack
            // Take note this also defines your packs id using the following format
            // mod/<namespace>:<path>`, e.g. `mod/examplemod:data/examplemod/datapacks/experimental`
            ResourceLocation.fromNamespaceAndPath("examplemod", "data/examplemod/datapacks/experimental"),
            
            // What kind of resources are contained within this pack
            // 'CLIENT_RESOURCES' for packs with client assets (resource packs)
            // 'SERVER_DATA' for packs with server data (data packs)
            PackType.SERVER_DATA,
            
            // Display name shown in the Experiments screen
            Component.literal("ExampleMod: Experiments"),
            
            // In order for this pack to load and enable feature flags, this MUST be 'FEATURE',
            // any other PackSource type is invalid here
            PackSource.FEATURE,
            
            // If this is true, the pack is always active and cannot be disabled, should always be false for feature packs
            false,
            
            // Priority to load resources from this pack in
            // 'TOP' this pack will be prioritized over other packs
            // 'BOTTOM' other packs will be prioritized over this pack 
            Pack.Position.TOP
    );
}
```

#### 在单人游戏中启用(Enabling in Singleplayer)

1. 创建一个新世界。
2. 导航到“实验功能(Experiments)”屏幕。
3. 打开所需的包。
4. 点击`完成(Done)`确认更改。

#### 在多人游戏中启用(Enabling in Multiplayer)

1. 打开您服务器的`server.properties`文件。
2. 将功能包ID添加到`initial-enabled-packs`中，每个包用`,`分隔。包ID在注册您的包查找器时定义，如上所示。

### 外部(External)

外部包以数据包形式提供给您的用户。

#### 在单人游戏中安装(Installation in Singleplayer)

1. 创建一个新世界。
2. 导航到数据包选择屏幕。
3. 将数据包zip文件拖放到游戏窗口上。
4. 将新可用的数据包移动到`已选(Selected)`包列表中。
5. 点击`完成(Done)`确认更改。

游戏现在会警告您任何新选择的实验性功能、潜在的漏洞、问题和崩溃。您可以点击`继续(Proceed)`确认这些更改，或点击`详情(Details)`查看所有选定包及其将启用功能的详细列表。

:::note
外部功能包不会显示在“实验功能(Experiments)”屏幕中。“实验功能”屏幕仅显示内置功能包。

要在启用后禁用外部功能包，请导航回数据包屏幕，并将外部包从`已选(Selected)`移回`可用(Available)`。
:::

#### 在多人游戏中安装(Installation in Multiplayer)

启用功能包(Feature Packs)只能在初始世界创建期间完成，一旦启用就无法禁用。

1. 创建目录 `./world/datapacks`
2. 将数据包zip文件上传到新创建的目录中
3. 打开您服务器的`server.properties`文件
4. 将数据包zip文件名（不包括`.zip`）添加到`initial-enabled-packs`中（每个包用`,`分隔）
   - 示例：zip文件`examplemod-experimental.zip`将像这样添加：`initial-enabled-packs=vanilla,examplemod-experimental`

### 数据生成(Data Generation)

_另请参阅：[数据生成(Datagen)][../resources/index.md#data-generation]_

功能包(Feature Packs)可以在常规的模组数据生成期间生成。这最好与内置包结合使用，但也可以将生成的结果压缩并作为外部包共享。只需选择一种方式，即不要同时将其作为外部包提供并作为内置包捆绑。

```
@SubscribeEvent // on the mod event bus
public static void gatherData(final GatherDataEvent.Client event) {
    DataGenerator generator = event.getGenerator();
    
    // To generate a feature pack, you must first obtain a pack generator instance for the desired pack.
    // generator.getBuiltinDatapack(<shouldGenerate>, <namespace>, <path>);
    // This will generate the feature pack into the following path:
    // ./data/<namespace>/datapacks/<path>
    PackGenerator featurePack = generator.getBuiltinDatapack(true, "examplemod", "experimental");
        
    // Register a provider to generate the `pack.mcmeta` file.
    featurePack.addProvider(output -> PackMetadataGenerator.forFeaturePack(
            output,
            
            // Description displayed in the Experiments screen
            Component.literal("Enabled experimental features for ExampleMod"),
            
            // Set of Feature flags this pack should enable
            FeatureFlagSet.of(EXPERIMENTAL)
    ));
    
    // Register additional providers (recipes, loot tables) to `featurePack` to write any generated resources into this pack, rather than the root pack.
}
```