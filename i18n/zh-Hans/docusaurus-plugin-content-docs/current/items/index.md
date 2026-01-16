# 物品(Items)

与方块一起，物品是 Minecraft 的关键组成部分。方块构成了你周围的世界，而物品则存在于物品栏中。

## 物品到底是什么？

在我们进一步创建物品之前，了解物品究竟是什么，以及它与[方块(Block)][block]等的区别非常重要。让我们用一个例子来说明：

- 在世界上，你遇到了一个泥土方块并想挖掘它。这是一个**方块**，因为它被放置在世界上。（实际上，它不是一个方块，而是一个方块状态。更多详细信息请参阅[方块状态文章][blockstates]。）
    - 并非所有方块在破坏时都会掉落自身（例如树叶），更多信息请参阅[战利品表文章][loottables]。
- 一旦你[挖掘了方块][breaking]，它被移除（= 被空气方块替换）并且泥土掉落。掉落的泥土是一个物品[实体(Entity)][entity]。这意味着像其他实体（猪、僵尸、箭等）一样，它本质上可以被水推动或火和熔岩燃烧等事物移动。
- 一旦你捡起泥土物品实体，它就变成你物品栏中的一个**物品堆叠(Item Stack)**。物品堆叠简单来说就是具有一些额外信息（例如堆叠大小）的物品实例。
- 物品堆叠由相应的**物品**（我们正在创建的东西）支持。物品持有[数据组件(Data Components)][datacomponents]，其中包含所有物品堆叠初始化的默认信息（例如，每把铁剑的最大耐久度为250），而物品堆叠可以修改这些数据组件，允许同一物品的两个不同堆叠具有不同的信息（例如，一把铁剑还剩100次使用，而另一把铁剑还剩200次使用）。有关通过物品完成什么和通过物品堆叠完成什么的更多信息，请继续阅读。
    - 物品和物品堆叠之间的关系大致类似于[方块(Blocks)][block]和[方块状态(Blockstates)][blockstates]之间的关系，因为方块状态总是由方块支持。这不是一个非常准确的比较（例如，物品堆叠不是单例），但它给出了这里概念的基本概念。

## 创建物品

现在我们了解了物品是什么，让我们来创建一个！

与基本方块一样，对于不需要特殊功能的基本物品（如木棍、糖等），可以直接使用 `Item` 类。为此，在注册期间，使用 `Item.Properties` 参数实例化 `Item`。可以使用 `Item.Properties#of` 创建此 `Item.Properties` 参数，并通过调用其方法进行自定义：

- `setId` - 设置物品的资源键。
    - 这**必须**在每个物品上设置；否则，将抛出异常。
- `overrideDescription` - 设置物品的翻译键。创建的 `Component` 存储在 `DataComponents#ITEM_NAME` 中。
- `useBlockDescriptionPrefix` - 方便助手，使用翻译键 `block.<modid>.<registry_name>` 调用 `overrideDescription`。这应在任何 `BlockItem` 上调用。
- `requiredFeatures` - 设置此物品所需的功能标志。这主要用于小版本中 Vanilla 的功能锁定系统。除非你正在集成到被 Vanilla 功能标志锁定的系统后面，否则不鼓励使用此功能。
- `stacksTo` - 设置此物品的最大堆叠大小（通过 `DataComponents#MAX_STACK_SIZE`）。默认为64。例如，末影珍珠或其他只堆叠到16的物品使用。
- `durability` - 设置此物品的耐久度（通过 `DataComponents#MAX_DAMAGE`）并将初始伤害设置为0（通过 `DataComponents#DAMAGE`）。默认为0，表示“无耐久度”。例如，铁工具在这里使用250。注意，设置耐久度会自动将最大堆叠大小锁定为1。
- `fireResistant` - 使使用此物品的物品实体免疫火和熔岩（通过 `DataComponents#FIRE_RESISTANT`）。各种下界合金物品使用。
- `rarity` - 设置此物品的稀有度（通过 `DataComponents#RARITY`）。目前，这只是改变物品的颜色。`Rarity` 是一个枚举，由四个值组成：`COMMON`（白色，默认）、`UNCOMMON`（黄色）、`RARE`（水色）和 `EPIC`（浅紫色）。请注意，模组可能会添加更多稀有度类型。
- `setNoCombineRepair` - 禁用此物品的砂轮和合成网格修复。Vanilla中未使用。
- `jukeboxPlayable` - 设置插入唱片机时要播放的数据包 `JukeboxSong` 的资源键。
- `food` - 设置此物品的 [`FoodProperties`][food]（通过 `DataComponents#FOOD`）。

有关示例或查看 Minecraft 使用的各种值，请查看 `Items` 类。

### 剩余物和冷却时间

物品在被使用时可能有额外的属性，或者在一段时间内阻止使用物品：

- `craftRemainder` - 设置此物品的合成剩余物。Vanilla 使用这个用于装满的桶，在合成后留下空桶。
- `usingConvertsTo` - 设置通过 `Item#use`、`IItemExtension#finishUsingItem` 或 `Item#releaseUsing` 使用物品后要返回的物品。`ItemStack` 存储在 `DataComponents#USE_REMAINDER` 上。
- `useCooldown` - 设置再次使用物品之前的秒数（通过 `DataComponents#USE_COOLDOWN`）。

### 工具和盔甲

有些物品像[工具(Tools)][tools]和[盔甲(Armor)][armor]一样。这些是通过一系列物品属性构建的，只有一些使用委托给它们的关联类：

- `enchantable` - 设置堆叠的最大[附魔(Enchantment)][enchantment]值，允许物品被附魔（通过 `DataComponents#ENCHANTABLE`）。
- `repairable` - 设置可用于修复此物品耐久度的物品或标签（通过 `DataComponents#REPAIRABLE`）。必须具有耐久度组件，而不是 `DataComponents#UNBREAKABLE`。
- `equippable` - 设置物品可以装备到的槽位（通过 `DataComponents#EQUIPPABLE`）。
- `equippableUnswappable` - 与 `equippable` 相同，但禁用通过使用物品按钮（默认为右键单击）快速交换。

更多信息可以在相关页面上找到。

### 更多功能

直接使用 `Item` 只允许非常基本的物品。如果你想添加功能，例如右键交互，则需要一个扩展 `Item` 的自定义类。`Item` 类有许多可以覆盖以做不同事情的方法；有关更多信息，请参阅类 `Item` 和 `IItemExtension`。

物品的两个最常见用例是左键单击和右键单击。由于它们的复杂性以及它们触及其他系统，它们在单独的[交互文章][interactions]中进行了解释。

### `DeferredRegister.Items`

所有注册都使用 `DeferredRegister` 来注册其内容，物品也不例外。然而，由于添加新物品是绝大多数模组的重要功能，NeoForge 提供了 `DeferredRegister.Items` 辅助类，它扩展了 `DeferredRegister<Item>` 并提供了一些特定于物品的帮助程序：

``` java
public static final DeferredRegister.Items ITEMS = DeferredRegister.createItems(ExampleMod.MOD_ID);

public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerItem(
    "example_item",
    Item::new, // 属性将传递给的工厂。
    new Item.Properties() // 要使用的属性。
);
```

在内部，这将简单地调用 `ITEMS.register("example_item", registryName -> new Item(new Item.Properties().setId(ResourceKey.create(Registries.ITEM, registryName))))`，通过将属性参数应用于提供的物品工厂（通常是构造函数）。id 设置在属性上。

如果你想使用 `Item::new`，你可以完全省略工厂并使用 `simple` 方法变体：

``` java
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerSimpleItem(
    "example_item",
    new Item.Properties() // 要使用的属性。
);
```

这与前面的示例完全相同，但稍微短一些。当然，如果你想使用 `Item` 的子类而不是 `Item` 本身，你将不得不使用前面的方法。

这两种方法还有省略 `new Item.Properties()` 参数的重载：

``` java
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerItem("example_item", Item::new);

// 也省略 Item::new 参数的变体
public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerSimpleItem("example_item");
```

最后，还有方块的快捷方式。与 `setId` 一起，这些还调用 `useBlockDescriptionPrefix` 将翻译键设置为方块使用的翻译键：

``` java
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    "example_block",
    ExampleBlocksClass.EXAMPLE_BLOCK,
    new Item.Properties()
);

// 省略属性参数的变体：
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    "example_block",
    ExampleBlocksClass.EXAMPLE_BLOCK
);

// 省略名称参数的变体，而是使用方块的注册名：
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    // 必须是 `Holder<Block>` 的实例
    // DeferredBlock<T> 也可以工作
    ExampleBlocksClass.EXAMPLE_BLOCK,
    new Item.Properties()
);

// 省略名称和属性的变体：
public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem(
    // 必须是 `Holder<Block>` 的实例
    // DeferredBlock<T> 也可以工作
    ExampleBlocksClass.EXAMPLE_BLOCK
);
```

:::note
如果你在单独的类中保留注册的方块，你应该在物品类之前加载方块类。
:::

### 资源

如果你注册物品并通过 `/give` 或[创造模式标签页(Creative Tabs)][creativetabs]获取物品，你会发现它缺少适当的模型和纹理。这是因为纹理和模型由 Minecraft 的资源系统处理。

对于每个物品，你将需要添加 - 或[生成(Generate)][datagen] - 以下 JSON 文件：

- 具有关联[纹理(Texture)][texture]的[客户端物品(Client Item)][citems]
- 一个[翻译(Translation)][i18n]
- 一个[配方(Recipe)][recipes]（可选）
- 一些物品[标签(Tags)][tags]（可选）

对于以上所有内容，也参考类似 Vanilla 方块的文件和数据生成器。

## `ItemStack`s

与方块和方块状态一样，大多数你期望使用 `Item` 的地方实际上都使用 `ItemStack`。`ItemStack` 表示容器中的一个或多个物品堆叠，例如物品栏。再次与方块和方块状态一样，方法应由 `Item` 覆盖并在 `ItemStack` 上调用，并且 `Item` 中的许多方法都会传递一个 `ItemStack` 实例。

`ItemStack` 由三个主要部分组成：

- 它代表的 `Item`，可通过 `ItemStack#getItem` 或 `getItemHolder` 获取 `Holder<Item>`。
- 堆叠大小，通常在1到64之间，可通过 `getCount` 获取，并通过 `setCount` 或 `shrink` 更改。
- [数据组件(Data Components)][datacomponents] 映射，其中存储特定于堆叠的数据。可通过 `getComponents` 获取。组件值通常通过 `has`、`get`、`set`、`update` 和 `remove` 访问和更改。

要创建新的 `ItemStack`，请调用 `new ItemStack(Item)`，传入支撑物品。默认情况下，这使用计数1且没有NBT数据；如果需要，也有接受计数和NBT数据的构造函数重载。

`ItemStack` 是可变对象（见下文），但有时需要将它们视为不可变对象。如果你需要修改被视为不可变的 `ItemStack`，可以使用 `#copy` 克隆堆叠，或者如果需要特定堆叠大小，则使用 `#copyWithCount`。

如果你想表示堆叠没有物品，请使用 `ItemStack.EMPTY`。如果你想检查 `ItemStack` 是否为空，请调用 `#isEmpty`。

### `ItemStack` 的可变性

`ItemStack` 是可变对象。这意味着如果你调用例如 `#setCount` 或任何数据组件映射方法，`ItemStack` 本身将被修改。Vanilla 广泛使用 `ItemStack` 的可变性，并且一些方法依赖于它。例如，`#split` 从调用它的堆叠中分离出给定的数量，同时修改调用者并在此过程中返回一个新的 `ItemStack`。

然而，这有时在处理多个 `ItemStack` 时会导致问题。最常见的情况是处理物品栏槽位，因为你必须考虑当前光标选择的 `ItemStack`，以及你尝试插入/提取的 `ItemStack`。

:::tip
如有疑问，为了安全起见，最好 `#copy` 堆叠。
:::

### JSON 表示

在许多情况下，例如[配方(Recipes)][recipes]，物品堆叠需要表示为 JSON 对象。物品堆叠的 JSON 表示如下所示：

``` json5
{
    // 物品 ID。必需。
    "id": "minecraft:dirt",
    // 物品堆叠计数 [1, 99]。可选，默认为1。
    "count": 4,
    // 数据组件映射。可选，默认为空映射。
    "components": {
        "minecraft:enchantment_glint_override": true
    }
}
```

## 创造模式标签页(Creative Tabs)

默认情况下，你的物品只能通过 `/give` 获得，不会出现在创造模式物品栏中。让我们改变这一点！

将物品放入创造菜单的方式取决于你想要添加到哪个标签页。

### 现有创造模式标签页

:::note
此方法用于将你的物品添加到 Minecraft 的标签页或其他模组的标签页中。要将物品添加到自己的标签页，请参阅下文。
:::

可以通过 `BuildCreativeModeTabContentsEvent` 将物品添加到现有的 `CreativeModeTab` 中，该事件在[模组事件总线][modbus]上触发，仅在[逻辑客户端][sides]上。通过调用 `event#accept` 添加物品。

``` java
//MyItemsClass.MY_ITEM 是 Supplier<? extends Item>，MyBlocksClass.MY_BLOCK 是 Supplier<? extends Block>
@SubscribeEvent // 在模组事件总线上
public static void buildContents(BuildCreativeModeTabContentsEvent event) {
    // 这是我们要添加的标签页吗？
    if (event.getTabKey() == CreativeModeTabs.INGREDIENTS) {
        event.accept(MyItemsClass.MY_ITEM.get());
        // 接受一个 ItemLike。这假设 MY_BLOCK 有一个对应的物品。
        event.accept(MyBlocksClass.MY_BLOCK.get());
    }
}
```

该事件还提供了一些额外信息，例如 `getFlags` 用于获取启用的功能标志列表，或 `hasPermissions` 用于检查玩家是否有权限查看操作员物品标签页。

### 自定义创造模式标签页

`CreativeModeTab` 是一个注册表，意味着自定义的 `CreativeModeTab` 必须[注册][registering]。创建创造模式标签页使用构建器系统，可通过 `CreativeModeTab#builder` 获取构建器。构建器提供选项来设置标题、图标、默认物品和许多其他属性。此外，NeoForge 提供了额外的方法来自定义标签页的图像、标签和槽位颜色、标签页应排序的位置等。

``` java
//CREATIVE_MODE_TABS 是一个 DeferredRegister<CreativeModeTab>
public static final Supplier<CreativeModeTab> EXAMPLE_TAB = CREATIVE_MODE_TABS.register("example", () -> CreativeModeTab.builder()
    //设置标签页的标题。别忘了添加翻译！
    .title(Component.translatable("itemGroup." + MOD_ID + ".example"))
    //设置标签页的图标。
    .icon(() -> new ItemStack(MyItemsClass.EXAMPLE_ITEM.get()))
    //将你的物品添加到标签页。
    .displayItems((params, output) -> {
        output.accept(MyItemsClass.MY_ITEM.get());
        // 接受一个 ItemLike。这假设 MY_BLOCK 有一个对应的物品。
        output.accept(MyBlocksClass.MY_BLOCK.get());
    })
    .build()
);
```

## `ItemLike`

`ItemLike` 是 Vanilla 中由 `Item` 和[`Block`s][block] 实现的接口。它定义了方法 `#asItem`，该方法返回对象实际的物品表示：`Item` 只返回自身，而 `Block` 返回其关联的 `BlockItem`（如果可用），否则返回 `Blocks.AIR`。`ItemLike` 用于各种上下文中，其中物品的“来源”并不重要，例如在许多[数据生成器(Data Generators)][datagen]中。

也可以在你的自定义对象上实现 `ItemLike`。只需覆盖 `#asItem` 即可。

[armor]: armor.md
[block]: ../blocks/index.md
[blockstates]: ../blocks/states.md
[breaking]: ../blocks/index.md#breaking-a-block
[citems]: ../resources/client/models/items.md
[creativetabs]: #creative-tabs
[datacomponents]: datacomponents.md
[datagen]: ../resources/index.md#data-generation
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[entity]: ../entities/index.md
[food]: consumables.md#food
[hunger]: https://minecraft.wiki/w/Hunger#Mechanics
[interactions]: interactions.md
[loottables]: ../resources/server/loottables/index.md
[modbus]: ../concepts/events.md#event-buses
[recipes]: ../resources/server/recipes/index.md
[registering]: ../concepts/registries.md#methods-for-registering
[sides]: ../concepts/sides.md
[tools]: tools.md
[datagen]: ../resources/index.md#data-generation
[i18n]: ../resources/client/i18n.md
[texture]: ../resources/client/textures.md
[tags]: ../resources/server/tags.md