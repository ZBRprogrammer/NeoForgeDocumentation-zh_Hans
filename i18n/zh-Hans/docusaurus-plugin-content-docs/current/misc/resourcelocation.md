# 资源位置(Resource Locations)

`ResourceLocation` 是《我的世界》(Minecraft)中最重要的概念之一。它们用作[注册表(registries)][registries]中的键、数据或资源文件的标识符、代码中模型的引用，以及许多其他用途。`ResourceLocation` 由两部分组成：命名空间(Namespace)和路径(Path)，用 `:` 分隔。

命名空间表示位置引用的模组、资源包或数据包。例如，模组 ID 为 `examplemod` 的模组将使用 `examplemod` 命名空间。《我的世界》(Minecraft)使用 `minecraft` 命名空间。可以通过简单地创建相应的数据文件夹来随意定义额外的命名空间，这通常由数据包(Datapacks)完成，以将其逻辑与集成到原版的位置分开。

路径是您想要的对象在命名空间内的引用。例如，`minecraft:cow` 是对 `minecraft` 命名空间中名为 `cow` 的对象的引用——通常此位置用于从实体注册表中获取奶牛实体。另一个例子是 `examplemod:example_item`，它可能用于从物品注册表中获取您的模组的 `example_item`。

`ResourceLocation` 只能包含小写字母、数字、下划线、点和连字符。路径还可以包含正斜杠。请注意，由于 Java 模块限制，模组 ID 不能包含连字符，这同样意味着模组命名空间也不能包含连字符（但它们仍然允许出现在路径中）。

:::信息
`ResourceLocation` 本身并不说明我们将其用于何种对象。名为 `minecraft:dirt` 的对象存在于多个地方，例如。将由接收 `ResourceLocation` 的任何对象将其与对象关联起来。
:::

可以通过调用 `ResourceLocation.fromNamespaceAndPath("examplemod", "example_item")` 或 `ResourceLocation.parse("examplemod:example_item")` 来创建新的 `ResourceLocation`。如果使用 `withDefaultNamespace`，该字符串将用作路径，而 `minecraft` 将用作命名空间。因此，例如，`ResourceLocation.withDefaultNamespace("example_item")` 将产生 `minecraft:example_item`。

`ResourceLocation` 的命名空间和路径可以分别使用 `ResourceLocation.getNamespace()` 和 `getPath()` 检索，组合形式可以通过 `ResourceLocation.toString` 检索。

`ResourceLocation` 是不可变的。`ResourceLocation` 上的所有实用方法，例如 `withPrefix` 或 `withSuffix`，都返回一个新的 `ResourceLocation`。

## 解析 `ResourceLocation`

有些地方，例如注册表(Registries)，直接使用 `ResourceLocation`。然而，其他一些地方会根据需要解析 `ResourceLocation`。例如：

- `ResourceLocation` 用作图形用户界面(GUI)背景的标识符。例如，熔炉 GUI 使用资源位置 `minecraft:textures/gui/container/furnace.png`。这映射到磁盘上的文件 `assets/minecraft/textures/gui/container/furnace.png`。请注意，此资源位置中需要 `.png` 后缀。
- `ResourceLocation` 用作方块模型(Block Model)的标识符。例如，泥土的方块模型(Block Model)使用资源位置 `minecraft:block/dirt`。这映射到磁盘上的文件 `assets/minecraft/models/block/dirt.json`。请注意，此处不需要 `.json` 后缀。还要注意，此资源位置会自动映射到 `models` 子文件夹。
- `ResourceLocation` 用作客户端物品(Client Item)的标识符。例如，苹果的客户端物品使用资源位置 `minecraft:apple`（由 `DataComponents.ITEM_MODEL` 定义）。这映射到文件 `assets/minecraft/items/apple.json`。请注意，此处不需要 `.json` 后缀。还要注意，此资源位置会自动映射到 `items` 子文件夹。
- `ResourceLocation` 用作配方(Recipe)的标识符。例如，铁块合成配方使用资源位置 `minecraft:iron_block`。这映射到磁盘上的文件 `data/minecraft/recipe/iron_block.json`。请注意，此处不需要 `.json` 后缀。还要注意，此资源位置会自动映射到 `recipe` 子文件夹。

`ResourceLocation` 是否需要文件后缀，或者资源位置具体解析为什么，取决于具体用例。

## `ResourceKey`

`ResourceKey` 将注册表 ID 与注册表名称结合在一起。一个例子是注册表 ID 为 `minecraft:item` 且注册表名称为 `minecraft:diamond_sword` 的注册表键(Resource Key)。与 `ResourceLocation` 不同，`ResourceKey` 实际上引用一个唯一的元素，因此能够清楚标识一个元素。它们最常用于许多不同注册表相互接触的上下文中。一个常见的用例是数据包(Datapacks)，尤其是世界生成(Worldgen)。

可以通过静态方法 `ResourceKey.create(ResourceKey<? extends Registry<T>>, ResourceLocation)` 创建新的 `ResourceKey`。这里的第二个参数是注册表名称，而第一个参数是所谓的注册表键(Registry Key)。注册表键是一种特殊的 `ResourceKey`，其注册表是根注册表（即所有其他注册表的注册表）。可以通过 `ResourceKey.createRegistryKey(ResourceLocation)` 使用所需注册表的 ID 来创建注册表键。

`ResourceKey` 在创建时被内部化。这意味着通过引用相等性 (`==`) 进行比较是可能的且被鼓励的，但它们的创建相对昂贵。

[registries]: ../concepts/registries.md
[sides]: ../concepts/sides.md