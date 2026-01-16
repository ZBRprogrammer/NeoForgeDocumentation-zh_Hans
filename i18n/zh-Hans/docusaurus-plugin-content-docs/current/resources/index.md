# 资源 (Resources)

资源是游戏使用的外部文件，但不是代码。最突出的资源种类是纹理 (textures)，然而，在 Minecraft 生态系统中还存在许多其他类型的资源。当然，所有这些资源都需要在代码端有一个消费者 (consumer)，因此消费系统也分组在本节中。

Minecraft 通常有两种资源：用于[逻辑客户端 (logical client)][logicalsides]的资源，称为资产 (assets)，以及用于[逻辑服务器 (logical server)][logicalsides]的资源，称为数据 (data)。资产主要是仅供显示的信息，例如纹理、显示模型、翻译或声音，而数据包括影响游戏玩法的各种内容，例如战利品表 (loot tables)、配方 (recipes) 或世界生成信息 (worldgen information)。它们分别从资源包 (resource packs) 和数据包 (data packs) 加载。NeoForge 为每个模组生成一个内置的资源包和数据包。

资源包和数据包通常都需要一个 [`pack.mcmeta` 文件][packmcmeta]；但是，现代 NeoForge 在运行时为您生成这些文件，因此您无需担心。

如果您对某些内容的格式感到困惑，请查看 Vanilla 资源。您的 NeoForge 开发环境不仅包含 Vanilla 代码，还包含 Vanilla 资源。它们可以在“外部资源”部分（IntelliJ）/“项目库”部分（Eclipse）中找到，名称为 `ng_dummy_ng.net.minecraft:client:client-extra:<minecraft_version>`（用于 Minecraft 资源）或 `ng_dummy_ng.net.neoforged:neoforge:<neoforge_version>`（用于 NeoForge 资源）。

## 资产 (Assets)

_另请参阅：[Minecraft Wiki][mcwiki] 上的[资源包 (Resource Packs)][mcwikiresourcepacks]_

资产 (assets)，或客户端资源 (client-side resources)，是仅在[客户端 (client)][sides]相关的所有资源。它们从资源包 (resource packs) 加载，有时也称为旧术语“纹理包 (texture packs)”（源于旧版本中它们只能影响纹理）。资源包基本上是一个 `assets` 文件夹。`assets` 文件夹包含资源包包含的各种命名空间 (namespaces) 的子文件夹；每个命名空间都是一个子文件夹。例如，一个 ID 为 `coolmod` 的模组的资源包可能包含一个 `coolmod` 命名空间，但可能还包含其他命名空间，例如 `minecraft`。

NeoForge 自动将所有模组资源包收集到 `Mod resources` 包中，该包位于资源包菜单的“已选包 (Selected Packs)”侧面的底部。目前无法禁用 `Mod resources` 包。但是，位于 `Mod resources` 包之上的资源包会覆盖其下方资源包中定义的资源。这种机制允许资源包制作者覆盖您模组的资源，也允许模组开发者根据需要覆盖 Minecraft 资源。

资源包可能包含影响以下内容的文件夹和文件：

| 文件夹名称 (Folder Name) | 内容 (Contents) |
|------------------|-----------------------------------------|
| `atlases` | 纹理图集源 (Texture Atlas Sources) |
| `blockstates` | [方块状态文件 (Blockstate Files)][bsfile] |
| `equipment` | [装备信息 (Equipment Info)][equipment] |
| `font` | 字体定义 (Font Definitions) |
| `items` | [客户端物品 (Client Items)][citems] |
| `lang` | [翻译文件 (Translation Files)][translations] |
| `models` | [模型 (Models)][models] |
| `particles` | [粒子定义 (Particle Definitions)][particles] |
| `post_effect` | 后期处理屏幕特效 (Post Processing Screen Effects) |
| `shaders` | 元数据、片段和顶点着色器 (Metadata, Fragement, and Vertex Shaders) |
| `sounds` | [声音文件 (Sound Files)][sounds] |
| `texts` | 杂项文本文件 (Miscellaneous Text files) |
| `textures` | [纹理 (Textures)][textures] |
| `waypoint_style` | 路径点图标元数据 (Waypoint Icon Metadata) |

## 数据 (Data)

_另请参阅：[Minecraft Wiki][mcwiki] 上的[数据包 (Data Packs)][mcwikidatapacks]_

与资产相反，数据 (data) 是所有[服务器 (server)][sides]资源的术语。类似于资源包，数据通过数据包 (data packs) 加载。像资源包一样，数据包由一个 [`pack.mcmeta` 文件][packmcmeta] 和一个名为 `data` 的根文件夹组成。然后，再次类似于资源包，该 `data` 文件夹包含数据包包含的各种命名空间的子文件夹；每个命名空间都是一个子文件夹。例如，一个 ID 为 `coolmod` 的模组的数据包可能包含一个 `coolmod` 命名空间，但可能还包含其他命名空间，例如 `minecraft`。

NeoForge 在创建新世界时自动将所有模组数据包应用于该世界。目前无法禁用模组数据包。但是，大多数数据文件可以被优先级更高的数据包覆盖（从而通过用空文件替换它们来删除）。可以通过将额外的数据包放置在世界 `datapacks` 子文件夹中，然后通过 [`/datapack`][datapackcmd] 命令启用或禁用它们来启用或禁用。

:::info
目前没有内置方法将一组自定义数据包应用于每个世界。但是，有一些模组可以实现这一点。
:::

数据包可能包含影响以下内容的文件夹和文件：

| 文件夹名称 (Folder Name) | 内容 (Contents) |
|------------------------------------------------------------------------------------------------|------------------------------|
| `advancement` | [进度 (Advancements)][advancements] |
| `banner_pattern` | 旗帜图案 (Banner patterns) |
| `cat_variant`、`chicken_variant`、`cow_variant`、`frog_variant`、`pig_variant`、`wolf_variant` | 实体变种 (Entity variants) |
| `damage_type` | [伤害类型 (Damage types)][damagetypes] |
| `datapacks` | 内置数据包 (Built-in datapacks) |
| `dialog` | 对话框菜单 (Dialog menus) |
| `enchantment`、`enchantment_provider` | [附魔 (Enchantments)][enchantment] |
| `instrument`、`jukebox_song`、`wolf_sound_variant` | 声音引用元数据 (Sound reference metadata) |
| `painting_variant` | 画 (Paintings) |
| `loot_table` | [战利品表 (Loot tables)][loottables] |
| `recipe` | [配方 (Recipes)][recipes] |
| `tags` | [标签 (Tags)][tags] |
| `test_environment`、`test_instance` | [游戏测试 (Game tests)][gmt] |
| `trial_spawner` | 战斗挑战 (Combat challenges) |
| `trim_material`、`trim_pattern` | 盔甲纹饰 (Armor trims) |
| `neoforge/data_maps` | [数据映射 (Data maps)][datamap] |
| `neoforge/loot_modifiers` | [全局战利品修改器 (Global loot modifiers)][glm] |
| `dimension`、`dimension_type`、`structure`、`worldgen`、`neoforge/biome_modifier` | 世界生成文件 (Worldgen files) |

此外，它们还可能包含一些与命令集成的系统的子文件夹。这些系统很少与模组结合使用，但无论如何值得提及：

| 文件夹名称 (Folder name) | 内容 (Contents) |
|-----------------|--------------------------------|
| `chat_type` | [聊天类型 (Chat types)][chattype] |
| `function` | [函数 (Functions)][function] |
| `item_modifier` | [物品修改器 (Item modifiers)][itemmodifier] |
| `predicate` | [谓词 (Predicates)][predicate] |

## `pack.mcmeta`

_另请参阅：[Minecraft Wiki][mcwiki] 上的 [`pack.mcmeta` (资源包)][packmcmetaresourcepack] 和 [`pack.mcmeta` (数据包)][packmcmetadatapack]_

`pack.mcmeta` 文件保存资源包或数据包的元数据。对于模组，NeoForge 使此文件过时，因为 `pack.mcmeta` 是合成生成的。如果您仍然需要 `pack.mcmeta` 文件，完整规范可以在链接的 Minecraft Wiki 文章中找到。

## 数据生成 (Data Generation)

数据生成 (Data generation)，俗称 datagen，是一种以编程方式生成 JSON 资源文件的方法，以避免手动编写它们所带来的繁琐且容易出错的过程。这个名字有点误导性，因为它既适用于资产也适用于数据。

Datagen 通过“数据 (Data)”运行配置运行，该配置与“客户端 (Client)”和“服务器 (Server)”运行配置一起为您生成。数据运行配置遵循[模组生命周期 (mod lifecycle)][lifecycle]，直到注册表事件 (registry events) 触发之后。然后它触发一个 [`GatherDataEvent`s][event]，在该事件中，您可以以数据提供者 (data providers) 的形式注册要生成的对象，将所述对象写入磁盘，并结束该过程。

有两个子类型在[**物理端 (physical side)**][physicalside]上操作：`GatherDataEvent.Client` 和 `GatherDataEvent.Server`。`GatherDataEvent.Client` 可能包含所有要生成的提供者。另一方面，`GatherDataEvent.Server` 可能只包含用于生成数据包条目的提供者。

:::note
关于如何注册您的提供者，有两个建议。前者是在 `GatherDataEvent.Client` 中注册所有提供者，并使用 `runClientData` 任务生成数据。后者是将客户端提供者注册到 `GatherDataEvent.Client`，将服务器提供者注册到 `GatherDataEvent.Server`，分别通过运行 `runClientData` 和 `runServerData` 任务来生成它们。

由于 MDK 通过设置默认的 `clientData` 配置使用前一种解决方案，所有显示的示例将使用前者，将所有提供者注册到 `GatherDataEvent.Client`。
:::

所有数据提供者都扩展 `DataProvider` 接口，并且通常需要重写一个方法。以下是 Minecraft 和 NeoForge 提供的值得注意的数据生成器列表（链接的文章添加了更多信息，例如辅助方法）：

| 类 (Class) | 方法 (Method) | 生成内容 (Generates) | 端 (Side) | 备注 (Notes) |
|------------------------------------------------------|----------------------------------|-------------------------------------------------------------------------|--------|-----------------------------------------------------------------------------------------------------------------|
| [`ModelProvider`][modelprovider] | `registerModels()` | 模型、方块状态文件、客户端物品 (Models, Blockstate Files, Client Items) | 客户端 (Client) | |
| [`LanguageProvider`][langprovider] | `addTranslations()` | 翻译 (Translations) | 客户端 (Client) | 还需要在构造函数中传递语言。 |
| [`EquipmentAssetProvider`][equipmentasset] | `registerModels()` | 盔甲模型资产 (Assets for armor models) | 客户端 (Client) | |
| [`ParticleDescriptionProvider`][particleprovider] | `addDescriptions()` | 粒子定义 (Particle definitions) | 客户端 (Client) | |
| [`SoundDefinitionsProvider`][soundprovider] | `registerSounds()` | 声音定义 (Sound definitions) | 客户端 (Client) | |
| `SpriteSourceProvider` | `gather()` | 精灵源 / 图集 (Sprite sources / atlases) | 客户端 (Client) | |
| [`AdvancementProvider`][advancementprovider] | `generate()` | 进度 (Advancements) | 服务器 (Server) | 需要额外的类才能正常工作，详情见链接文章。 |
| [`LootTableProvider`][loottableprovider] | `generate()` | 战利品表 (Loot tables) | 服务器 (Server) | 需要额外的方法和类才能正常工作，详情见链接文章。 |
| [`RecipeProvider`][recipeprovider] | `buildRecipes(RecipeOutput)` | 配方 (Recipes) | 服务器 (Server) | 需要额外的类才能正常工作，详情见链接文章。 |
| [`RecipePrioritiesProvider`][recipepriorities] | `start()` | 配方优先级顺序 (Priority order for recipes) | 服务器 (Server) | |
| [`TagsProvider` 的各种子类][tagsprovider] | `addTags(HolderLookup.Provider)` | 标签 (Tags) | 服务器 (Server) | 存在几个专门的子类，详情见链接文章。 |
| [`DataMapProvider`][datamapprovider] | `gather()` | 数据映射条目 (Data map entries) | 服务器 (Server) | |
| [`GlobalLootModifierProvider`][glmprovider] | `start()` | 全局战利品修改器 (Global loot modifiers) | 服务器 (Server) | |
| [`DatapackBuiltinEntriesProvider`][datapackprovider] | 无 (N/A) | 数据包内置条目，例如世界生成和[伤害类型 (damage types)][damagetypes] | 服务器 (Server) | 不重写方法，而是在构造函数的 lambda 中添加条目。详情见链接文章。 |
| `JsonCodecProvider` (抽象类) | `gather()` | 具有编解码器 (codec) 的对象 | 两者 (Both) | 可以扩展以用于任何具有[编解码器 (codec)]的对象以编码数据。 |

所有这些提供者都遵循相同的模式。首先，创建一个子类并添加您自己的要生成的资源。然后，在[事件处理器 (event handler)][eventhandler]中将提供者添加到事件中。使用 `RecipeProvider` 的示例：

```java
public class MyRecipeProvider extends RecipeProvider {
    public MyRecipeProvider(HolderLookup.Provider registries, RecipeOutput output) {
        super(registries, output);
    }

    @Override
    protected void buildRecipes() {
        // 在此注册您的配方。
    }

    // 数据提供者类
    public static class Runner extends RecipeProvider.Runner {

        public Runner(PackOutput output, CompletableFuture<HolderLookup.Provider> registries) {
            super(output, registries);
        }

        @Override
        protected abstract RecipeProvider createRecipeProvider(HolderLookup.Provider registries, RecipeOutput output) {
            return new MyRecipeProvider(registries, output);
        }
    }
}

// 在某个事件处理器类中
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    // 数据提供者应从调用 event.createDatapackRegistryObjects(...) 开始，
    // 以注册它们的数据包注册表对象。这允许其他提供者
    // 在它们自己的数据生成过程中使用这些对象。

    // 从那里，提供者通常可以使用 event.createProvider(...) 注册，
    // 它充当一个函数，提供 PackOutput 和可选的
    // CompletableFuture<HolderLookup.Provider>。

    // 注册提供者。
    event.createProvider(MyRecipeProvider.Runner::new);
    // 其他数据提供者在此。

    // 如果要在全局包中创建一个数据包，可以调用
    // DataGenerator#getBuiltinDatapack。然后，必须使用
    // PackGenerator#addProvider 方法将任何提供者添加到该包中。
    DataGenerator.PackGenerator examplePack = event.getGenerator().getBuiltinDatapack(
        true, // 应始终为 true。
        "examplemod", // 模组 ID。
        "example_pack" // 包的名称。
    );
    
    examplePack.addProvider(output -> ...);
}
```

该事件提供了一些助手和上下文供您使用：

- `event.createDatapackRegistryObjects(...)` 使用提供的 `RegistrySetBuilder` 创建并注册一个 `DatapackBuiltinEntriesProvider`。它还强制任何将来对查找提供者的使用都包含您的数据生成条目。
- `event.createProvider(...)` 通过提供 `PackOutput` 和可选的 `CompletableFuture<HolderLookup.Provider>` 作为 lambda 的一部分来注册提供者。
- `event.createBlockAndItemTags(...)` 通过使用 `TagsProvider<Block>` 构造 `TagsProvider<Item>` 来注册 `TagsProvider<Block>` 和 `TagsProvider<Item>`。
- `event.getGenerator()` 返回您向其注册提供者的 `DataGenerator`。
- `event.getPackOutput()` 返回一个 `PackOutput`，某些提供者使用它来确定其文件输出位置。
- `event.getResourceManager(PackType)` 返回一个 `ResourceManager`，提供者可以使用它来检查已存在的文件。
- `event.getLookupProvider()` 返回一个 `CompletableFuture<HolderLookup.Provider>`，主要由标签和数据生成注册表用于引用其他可能尚未存在的元素。
- `event.includeDev()` 和 `event.includeReports()` 是 `boolean` 方法，允许您检查是否启用了特定的命令行参数（见下文）。

### 命令行参数 (Command Line Arguments)

数据生成器可以接受几个命令行参数：

- `--mod examplemod`：告诉数据生成器为此模组运行 datagen。由 NeoGradle 自动为所属模组 ID 添加，如果您在一个项目中有多个模组，请添加此参数。
- `--output path/to/folder`：告诉数据生成器输出到给定文件夹。建议使用 Gradle 的 `file(...).getAbsolutePath()` 为您生成绝对路径（相对于项目根目录的路径）。默认为 `file('src/generated/resources').getAbsolutePath()`。
- `--existing path/to/folder`：告诉数据生成器在检查现有文件时考虑给定文件夹。与输出一样，建议使用 Gradle 的 `file(...).getAbsolutePath()`。
- `--existing-mod examplemod`：告诉数据生成器在检查现有文件时考虑给定模组 JAR 文件中的资源。
- 生成器模式 (Generator modes)（所有这些参数都是布尔参数，不需要任何额外参数）：
    - `--includeDev`：是否运行开发工具。通常不应由模组使用。在运行时使用 `GatherDataEvent#includeDev()` 检查。
    - `--includeReports`：是否转储已注册对象的列表。在运行时使用 `GatherDataEvent#includeReports()` 检查。
    - `--all`：启用所有生成器模式。

所有参数都可以通过将以下内容添加到您的 `build.gradle` 中来添加到运行配置中：

```groovy
runs {
    // 其他运行配置在此

    clientData {
        arguments.addAll '--arg1', 'value1', '--arg2', 'value2', '--all' // 布尔参数没有值
    }
}
```

例如，要复制默认参数，您可以指定以下内容：

```groovy
runs {
    // 其他运行配置在此

    clientData {
        arguments.addAll '--mod', 'examplemod', // 插入您自己的模组 ID
                '--output', file('src/generated/resources').getAbsolutePath(),
                '--all'
    }
}
```

[advancementprovider]: server/advancements.md#data-generation
[advancements]: server/advancements.md
[bsfile]: client/models/index.md#blockstate-files
[chattype]: https://minecraft.wiki/w/Chat_type
[citems]: client/models/items.md
[codec]: ../datastorage/codecs.md
[damagetypes]: server/damagetypes.md
[datamap]: server/datamaps/index.md
[datamapprovider]: server/datamaps/index.md#data-generation
[datapackcmd]: https://minecraft.wiki/w/Commands/datapack
[datapackprovider]: ../concepts/registries.md#data-generation-for-datapack-registries
[enchantment]: server/enchantments/index.md
[equipment]: ../items/armor.md#equipment-models
[equipmentasset]: ../items/armor.md#equipment-assets
[event]: ../concepts/events.md
[eventhandler]: ../concepts/events.md#registering-an-event-handler
[function]: https://minecraft.wiki/w/Function_(Java_Edition)
[glm]: server/loottables/glm.md
[glmprovider]: server/loottables/glm.md#datagen
[gmt]: ../misc/gametest.md
[itemmodifier]: https://minecraft.wiki/w/Item_modifier
[langprovider]: client/i18n.md#datagen
[lifecycle]: ../concepts/events.md#the-mod-lifecycle
[logicalsides]: ../concepts/sides.md#the-logical-side
[loottableprovider]: server/loottables/index.md#datagen
[loottables]: server/loottables/index.md
[mcwiki]: https://minecraft.wiki
[mcwikidatapacks]: https://minecraft.wiki/w/Data_pack
[mcwikiresourcepacks]: https://minecraft.wiki/w/Resource_pack
[modelprovider]: client/models/datagen.md
[models]: client/models/index.md
[packmcmeta]: #packmcmeta
[packmcmetadatapack]: https://minecraft.wiki/w/Data_pack#pack.mcmeta
[packmcmetaresourcepack]: https://minecraft.wiki/w/Resource_pack#Contents
[particleprovider]: client/particles.md#datagen
[particles]: client/particles.md
[physicalside]: ../concepts/sides.md#the-physical-side
[predicate]: https://minecraft.wiki/w/Predicate
[recipeprovider]: server/recipes/index.md#data-generation
[recipes]: server/recipes/index.md
[recipepriorities]: server/recipes/index.md#recipe-priorities
[sides]: ../concepts/sides.md
[soundprovider]: client/sounds.md#datagen
[sounds]: client/sounds.md
[tags]: server/tags.md
[tagsprovider]: server/tags.md#datagen
[textures]: client/textures.md
[translations]: client/i18n.md#language-files