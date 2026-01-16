---
sidebar_position: 4
---
# 工具(Tools)

工具是主要用于破坏[方块][block]的[物品][item]。许多模组添加了新的工具套装（例如铜工具）或新的工具类型（例如锤子）。

## 自定义工具套装

工具套装通常由五个物品组成：镐、斧、锹、锄和剑（剑在传统意义上不是工具，但为了一致性也包括在这里）。所有这些工具都使用以下八个[数据组件][datacomponents]实现：

- `DataComponents#MAX_DAMAGE` 和 `#DAMAGE` 用于耐久度
- `#MAX_STACK_SIZE` 将堆叠大小设置为 `1`
- `#REPAIRABLE` 用于在铁砧中修复工具
- `#ENCHANTABLE` 用于最大[附魔][enchantment]值
- `#ATTRIBUTE_MODIFIERS` 用于攻击伤害和攻击速度
- `#TOOL` 用于挖掘信息
- `#WEAPON` 用于物品承受的伤害和禁用盾牌

通常，每个工具都使用 `Item.Properties#tool`、`#sword` 或其中一个工具的委托（`pickaxe`、`axe`、`hoe`、`shovel`）设置。这些通常通过传入实用记录 `ToolMaterial` 来处理。请注意，其他通常被认为是工具的物品，如剪刀，没有通过数据组件实现其常见的挖掘逻辑。相反，它们直接继承 `Item` 并通过覆盖相关方法来处理挖掘。交互行为（默认右键单击）也没有数据组件，这意味着锹、斧和锄有它们自己的工具类 `ShovelItem`、`AxeItem` 和 `HoeItem`。

要创建一套标准的工具，你必须首先定义一个 `ToolMaterial`。参考值可以在 `ToolMaterial` 中的常量中找到。此示例使用铜工具，你可以在此处使用自己的材料并根据需要调整值。

``` java
// 我们将铜放在石头和铁之间的某个位置。
public static final ToolMaterial COPPER_MATERIAL = new ToolMaterial(
        // 决定此材料无法破坏哪些方块的标签。更多信息见下文。
        MyBlockTags.INCORRECT_FOR_COPPER_TOOL,
        // 决定材料的耐久度。
        // 石头是131，铁是250。
        200,
        // 决定材料的挖掘速度。剑不使用。
        // 石头使用4，铁使用6。
        5f,
        // 决定攻击伤害加成。不同的工具对此使用不同。例如，剑造成 (getAttackDamageBonus() + 4) 伤害。
        // 石头使用1，铁使用2，分别对应剑的5和6攻击伤害；我们的剑现在造成5.5伤害。
        1.5f,
        // 决定材料的可附魔性。这表示此工具上附魔的效果将有多好。
        // 金使用22，我们将铜设置得略低于此。
        20,
        // 决定哪些物品可以修复此材料的标签。
        Tags.Items.INGOTS_COPPER
);
```

现在我们有了 `ToolMaterial`，我们可以用它来[注册][registering]工具。所有 `tool` 委托都有相同的三个参数：

``` java
// ITEMS 是一个 DeferredRegister.Items
public static final DeferredItem<Item> COPPER_SWORD = ITEMS.registerItem(
    "copper_sword",
    props -> new Item(
        // 物品属性。
        props.sword(
            // 使用的材料。
            COPPER_MATERIAL,
            // 类型特定的攻击伤害加成。剑为3，锹为1.5，镐为1，斧和锄各不相同。
            3,
            // 类型特定的攻击速度修改器。玩家的默认攻击速度为4，因此要达到所需的1.6f值，我们使用 -2.4f。剑为 -2.4f，锹为 -3f，镐为 -2.8f，斧和锄各不相同。
            -2.4f,
        )
    )
);

public static final DeferredItem<Item> COPPER_AXE = ITEMS.registerItem("copper_axe", props -> new Item(props.axe(...)));
public static final DeferredItem<Item> COPPER_PICKAXE = ITEMS.registerItem("copper_pickaxe", props -> new Item(props.pickaxe(...)));
public static final DeferredItem<Item> COPPER_SHOVEL = ITEMS.registerItem("copper_shovel", props -> new Item(props.shovel(...)));
public static final DeferredItem<Item> COPPER_HOE = ITEMS.registerItem("copper_hoe", props -> new Item(props.hoe(...)));
```

:::note
`tool` 接受两个额外参数：代表可以挖掘哪些方块的 `TagKey`，以及击中时阻挡器（例如盾牌）被禁用的秒数。
:::

### 标签(Tags)

当创建 `ToolMaterial` 时，会为其分配一个方块[标签][tags]，其中包含如果使用此工具挖掘则不会掉落任何东西的方块。例如，`minecraft:incorrect_for_stone_tool` 标签包含像钻石矿石这样的方块，而 `minecraft:incorrect_for_iron_tool` 标签包含像黑曜石和远古残骸这样的方块。为了更容易将方块分配给它们不正确的挖掘等级，还存在一个用于需要此工具挖掘的方块的标签。例如，`minecraft:needs_iron_tool` 标签包含像钻石矿石这样的方块，而 `minecraft:needs_diamond_tool` 标签包含像黑曜石和远古残骸这样的方块。

如果你对此没问题，可以重用其中一个不正确标签用于你的工具。例如，如果我们希望我们的铜工具只是更耐用的石质工具，我们会传入 `BlockTags#INCORRECT_FOR_STONE_TOOL`。

或者，我们可以创建自己的标签，如下所示：

``` java
// 此标签将允许我们将这些方块添加到无法挖掘它们的不正确标签中
public static final TagKey<Block> NEEDS_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "needs_copper_tool"));

// 此标签将传入我们的材料
public static final TagKey<Block> INCORRECT_FOR_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "incorrect_for_cooper_tool"));
```

然后，我们填充我们的标签。例如，让铜能够挖掘金矿石、金块和红石矿石，但不能挖掘钻石或绿宝石。（红石块已经可以被石质工具挖掘。）标签文件位于 `src/main/resources/data/mod_id/tags/block/needs_copper_tool.json`（其中 `mod_id` 是你的模组 ID）：

``` json5
{
    "values": [
        "minecraft:gold_block",
        "minecraft:raw_gold_block",
        "minecraft:gold_ore",
        "minecraft:deepslate_gold_ore",
        "minecraft:redstone_ore",
        "minecraft:deepslate_redstone_ore"
    ]
}
```

然后，对于传入材料的标签，我们可以为任何对石质工具不正确但在我们的铜工具标签内的方块提供负面约束。标签文件位于 `src/main/resources/data/mod_id/tags/block/incorrect_for_cooper_tool.json`：

``` json5
{
    "values": [
        "#minecraft:incorrect_for_stone_tool"
    ],
    "remove": [
        "#mod_id:needs_copper_tool"
    ]
}
```

最后，我们可以将我们的标签传入我们的材料实例，如上所示。

如果你想检查工具是否可以使方块状态掉落其方块，请调用 `Tool#isCorrectForDrops`。可以通过调用 `ItemStack#get` 并传入 `DataComponents#TOOL` 来获取 `Tool`。

## 自定义工具

可以通过将 `Tool` [数据组件][datacomponents]（通过 `DataComponents#TOOL`）通过 `Item.Properties#component` 添加到物品的默认组件列表中来创建自定义工具。

`Tool` 包含一个 `Tool.Rule` 列表、持有工具时的默认挖掘速度（默认为 `1`）以及挖掘方块时工具应承受的伤害量（默认为 `1`）。`Tool.Rule` 包含三条信息：要应用规则的 `HolderSet` 方块、可选的用于挖掘该集合中方块的速率，以及一个可选的布尔值，用于确定这些方块是否可以由此工具掉落。如果未设置可选值，则将检查其他规则。如果所有规则都失败，则默认行为是默认挖掘速度和方块不能被掉落。

:::note
可以通过 `Registry#getOrThrow` 从 `TagKey` 创建 `HolderSet`。
:::

创建任何工具或多功能工具类物品（即，将两个或多个工具组合成一个物品，例如斧和镐作为一个物品）是可能的，而无需使用任何现有的 `ToolMaterial` 引用。它可以使用以下部分的组合来实现：

- 通过 `Item.Properties#component` 设置 `DataComponents#TOOL` 来添加具有你自己的规则的 `Tool`。
- 通过 `Item.Properties#attributes` 为物品添加[属性修饰符][attributemodifier]（例如攻击伤害、攻击速度）。
- 通过 `Item.Properties#durability` 为物品添加耐久度。
- 通过 `Item.Properties#repairable` 允许物品被修复。
- 通过 `Item.Properties#enchantable` 允许物品被附魔。
- 通过 `Item.Properties#component` 设置 `DataComponents#WEAPON`，允许物品用作武器并可能禁用阻挡器。
- 覆盖 `IItemExtension#canPerformAction` 以确定物品可以执行哪些 [`ItemAbility`s][itemability]。
- 如果你想基于 `ItemAbility`s 在右键单击时让你的物品修改方块状态，则调用 `IBlockExtension#getToolModifiedState`。
- 将你的工具添加到一些 `minecraft:enchantable/*` `ItemTags` 中，以便你的物品可以应用某些附魔。
- 将你的工具添加到一些 `minecraft:*_preferred_weapons` 标签中，以允许生物偏爱拾取和使用你的武器。

对于盾牌，你可以为主手应用 [`DataComponents#EQUIPPABLE`][equippable] 数据组件，并为活动的持盾实体应用 `DataComponents#BLOCKS_ATTACKS` 以减少伤害。

## `ItemAbility`s

`ItemAbility`s 是对物品能做什么和不能做什么的抽象。这包括左键和右键行为。NeoForge 在 `ItemAbilities` 类中提供默认的 `ItemAbility`s：

- 斧右键能力用于去皮（原木）、刮削（氧化的铜）和去蜡（涂蜡的铜）。
- 锹右键能力用于平整（土径）和熄灭（营火）。
- 剪刀能力用于挖掘（破坏方块）、收获（蜜脾）、移除盔甲（装甲狼）、雕刻（南瓜）、拆除（绊线）和修剪（停止植物生长）。
- 剑横扫、锄耕地、钓鱼竿抛掷、三叉戟投掷、刷子刷拭、打火石点燃和望远镜观察的能力。

要创建你自己的 `ItemAbility`s，请使用 `ItemAbility#get` - 如果需要，它将创建一个新的 `ItemAbility`。然后，在自定义工具类型中，根据需要覆盖 `IItemExtension#canPerformAction`。

要查询 `ItemStack` 是否可以执行某个 `ItemAbility`，请调用 `IItemStackExtension#canPerformAction`。请注意，这适用于任何 `Item`，而不仅仅是工具。

[block]: ../blocks/index.md
[datacomponents]: datacomponents.md
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[equippable]: armor.md#equippable
[item]: index.md
[itemability]: #itemabilitys
[registering]: ../concepts/registries.md#methods-for-registering
[tags]: ../resources/server/tags.md