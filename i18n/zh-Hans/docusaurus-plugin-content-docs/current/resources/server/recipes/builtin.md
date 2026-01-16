# 内置配方类型

Minecraft 提供了多种开箱即用的配方类型和序列化器供你使用。本文将解释每种配方类型，以及如何生成它们。

## 合成

合成配方通常在合成台、合成器，或在模组添加的合成台或机器中制作。它们的配方类型是 `minecraft:crafting`。

### 有序合成

一些最重要的配方——例如合成台、木棍或大多数工具——都是通过有序配方创建的。这些配方由一个必须按特定形状放入物品的合成图案或形状定义（因此称为“有序”）。让我们看一个示例：

```json5
{
    "type": "minecraft:crafting_shaped",
    "category": "equipment",
    "key": {
        "#": "minecraft:stick",
        "X": "minecraft:iron_ingot"
    },
    "pattern": [
        "XXX",
        " # ",
        " # "
    ],
    "result": {
        "count": 1,
        "id": "minecraft:iron_pickaxe"
    }
}
```

让我们逐行理解：

- `type`：这是有序配方序列化器的 ID，`minecraft:crafting_shaped`。
- `category`：此可选字段定义配方书中的 `CraftingBookCategory`。
- `key` 和 `pattern`：它们共同定义了物品必须如何放入合成网格。
    - 图案定义了最多三行、每行最多三个字符的字符串，以确定形状。所有行必须长度相同，即图案必须形成矩形。空格可用于表示应保持为空的槽位。
    - 键将图案中使用的字符与 [配料][ingredient] 关联起来。在上面的示例中，图案中的所有 `X` 必须是铁锭，所有 `#` 必须是木棍。
- `result`：配方的结果。这是 [物品堆叠的 JSON 表示][itemjson]。
- 示例中未显示的是 `group` 键。此可选字符串属性在配方书中创建一个组。同一组中的配方在配方书中将显示为一个。
- 示例中未显示的是 `show_notification`。此可选布尔值，当为 false 时，禁用首次使用或解锁时在右上角显示的提示。

然后，让我们看看如何在 `RecipeProvider#buildRecipes` 中生成这个配方：

```java
// 我们使用构建器模式，因此不创建变量。通过调用
// ShapedRecipeBuilder#shaped 并传入配方类别（在 RecipeCategory 枚举中找到）
// 和结果物品、结果物品与数量，或结果物品堆叠来创建新的构建器。
ShapedRecipeBuilder.shaped(this.registries.lookupOrThrow(Registries.ITEM), RecipeCategory.TOOLS, Items.IRON_PICKAXE)
        // 创建图案的行。每次调用 #pattern 会添加新的一行。
        // 图案将被验证，即检查其形状。
        .pattern("XXX")
        .pattern(" # ")
        .pattern(" # ")
        // 为图案创建键。图案中使用的所有非空格字符必须被定义。
        // 这可以接受 Ingredients、TagKey<Item>s 或 ItemLikes，即物品或方块。
        .define('X', Items.IRON_INGOT)
        .define('#', Items.STICK)
        // 创建配方进度。虽然消费后台系统不强制要求，
        // 但如果你省略它，配方构建器将会崩溃。第一个参数是进度名称，
        // 第二个是条件。通常，你希望使用 has() 快捷方式作为条件。
        // 可以通过多次调用 #unlockedBy 来添加多个进度要求。
        .unlockedBy("has_iron_ingot", this.has(Items.IRON_INGOT))
        // 将配方存储到传递的 RecipeOutput 中，以便写入磁盘。
        // 如果你想为配方添加条件，可以在输出上设置。
        .save(this.output);
```

此外，你可以调用 `#group` 和 `#showNotification` 来分别设置配方书组和切换提示弹出窗口。

### 无序合成

与有序合成配方不同，无序合成配方不关心配料放入的顺序。因此，没有图案和键，只有一个配料列表：

```json5
{
    "type": "minecraft:crafting_shapeless",
    "category": "misc",
    "ingredients": [
        "minecraft:brown_mushroom",
        "minecraft:red_mushroom",
        "minecraft:bowl"
    ],
    "result": {
        "count": 1,
        "id": "minecraft:mushroom_stew"
    }
}
```

和之前一样，让我们逐行理解：

- `type`：这是无序配方序列化器的 ID，`minecraft:crafting_shapeless`。
- `category`：此可选字段定义配方书中的类别。
- `ingredients`：一个 [配料][ingredient] 列表。列表顺序在代码中为查看配方目的而保留，但配方本身接受任何顺序的配料。
- `result`：配方的结果。这是 [物品堆叠的 JSON 表示][itemjson]。
- 示例中未显示的是 `group` 键。此可选字符串属性在配方书中创建一个组。同一组中的配方在配方书中将显示为一个。

然后，让我们看看如何在 `RecipeProvider#buildRecipes` 中生成这个配方：

```java
// 我们使用构建器模式，因此不创建变量。通过调用
// ShapelessRecipeBuilder#shapeless 并传入配方类别（在 RecipeCategory 枚举中找到）
// 和结果物品、结果物品与数量，或结果物品堆叠来创建新的构建器。
ShapelessRecipeBuilder.shapeless(this.registries.lookupOrThrow(Registries.ITEM), RecipeCategory.MISC, Items.MUSHROOM_STEW)
        // 添加配方配料。这可以接受 Ingredients、TagKey<Item>s 或 ItemLikes。
        // 也存在额外接受数量的重载，用于多次添加相同的配料。
        .requires(Blocks.BROWN_MUSHROOM)
        .requires(Blocks.RED_MUSHROOM)
        .requires(Items.BOWL)
        // 创建配方进度。虽然消费后台系统不强制要求，
        // 但如果你省略它，配方构建器将会崩溃。第一个参数是进度名称，
        // 第二个是条件。通常，你希望使用 has() 快捷方式作为条件。
        // 可以通过多次调用 #unlockedBy 来添加多个进度要求。
        .unlockedBy("has_mushroom_stew", this.has(Items.MUSHROOM_STEW))
        .unlockedBy("has_bowl", this.has(Items.BOWL))
        .unlockedBy("has_brown_mushroom", this.has(Blocks.BROWN_MUSHROOM))
        .unlockedBy("has_red_mushroom", this.has(Blocks.RED_MUSHROOM))
        // 将配方存储到传递的 RecipeOutput 中，以便写入磁盘。
        // 如果你想为配方添加条件，可以在输出上设置。
        .save(this.output);
```

此外，你可以调用 `#group` 来设置配方书组。

:::info
单物品配方（例如，存储方块的拆解）应遵循原版标准，使用无序配方。
:::

### 嬗变合成

嬗变配方是一种特殊的单物品合成配方类型，其中输入堆叠的数据组件会完全复制到结果堆叠中。嬗变通常发生在两个不同的物品之间，其中一个是另一个的染色版本。例如：

```json5
{
    "type": "minecraft:crafting_transmute",
    "category": "misc",
    "group": "shulker_box_dye",
    "input": "#minecraft:shulker_boxes",
    "material": "minecraft:blue_dye",
    "result": {
        "id": "minecraft:blue_shulker_box"
    }
}
```

和之前一样，让我们逐行理解：

- `type`：这是无序配方序列化器的 ID，`minecraft:crafting_transmute`。
- `category`：此可选字段定义配方书中的类别。
- `group`：此可选字符串属性在配方书中创建一个组。同一组中的配方在配方书中将显示为一个，这通常对嬗变配方有意义。
- `input`：要嬗变的 [配料]。
- `material`：将堆叠转换为其结果的 [配料]。
- `result`：配方的结果。这是 [物品堆叠的 JSON 表示][itemjson]。

然后，让我们看看如何在 `RecipeProvider#buildRecipes` 中生成这个配方：

```java
// 我们使用构建器模式，因此不创建变量。通过调用
// TransmuteRecipeBuilder#transmute 并传入配方类别（在 RecipeCategory 枚举中找到）、
// 配料输入、配料材料和结果物品来创建新的构建器。
TransmuteRecipeBuilder.transmute(RecipeCategory.MISC, this.tag(ItemTags.SHULKER_BOXES),
    Ingredient.of(DyeItem.byColor(DyeColor.BLUE)), ShulkerBoxBlock.getBlockByColor(DyeColor.BLUE).asItem())
        // 设置在配方书中显示的配方组。
        .group("shulker_box_dye")
        // 创建配方进度。虽然消费后台系统不强制要求，
        // 但如果你省略它，配方构建器将会崩溃。第一个参数是进度名称，
        // 第二个是条件。通常，你希望使用 has() 快捷方式作为条件。
        // 可以通过多次调用 #unlockedBy 来添加多个进度要求。
        .unlockedBy("has_shulker_box", this.has(ItemTags.SHULKER_BOXES))
        // 将配方存储到传递的 RecipeOutput 中，以便写入磁盘。
        // 如果你想为配方添加条件，可以在输出上设置。
        .save(this.output);
```

### 特殊合成

在某些情况下，输出必须根据输入动态创建。大多数时候，这是为了通过从输入堆叠计算其值来设置输出上的数据组件。这些配方通常只指定类型并硬编码其他所有内容。例如：

```json5
{
    "type": "minecraft:crafting_special_armordye"
}
```

这个用于染色皮革盔甲的配方，只指定了类型并硬编码了其他所有内容——最引人注目的是颜色计算，这在 JSON 中很难表达。Minecraft 为大多数特殊合成配方添加了 `crafting_special_` 前缀，但无需遵循此做法。

在 `RecipeProvider#buildRecipes` 中生成这个配方的操作如下：

```java
// #special 的参数是一个 Function<CraftingBookCategory, Recipe<?>>。
// 所有原版特殊配方都为此使用一个带有 CraftingBookCategory 参数的构造函数。
SpecialRecipeBuilder.special(ArmorDyeRecipe::new)
        // 这个 #save 的重载允许我们指定一个名称。它也可以在有序或无序构建器上使用。
        .save(this.output, "armor_dye");
```

原版提供以下特殊合成序列化器（模组可能会添加更多）：

- `minecraft:crafting_special_armordye`：用于染色皮革盔甲和其他可染色物品。
- `minecraft:crafting_special_bannerduplicate`：用于复制旗帜。
- `minecraft:crafting_special_bookcloning`：用于复制已写好的书。这会使结果书的 generation 属性增加一。
- `minecraft:crafting_special_firework_rocket`：用于制作烟花火箭。
- `minecraft:crafting_special_firework_star`：用于制作烟花之星。
- `minecraft:crafting_special_firework_star_fade`：用于为烟花之星添加淡化效果。
- `minecraft:crafting_special_mapcloning`：用于复制已填充的地图。也适用于藏宝图。
- `minecraft:crafting_special_mapextending`：用于扩展已填充的地图。
- `minecraft:crafting_special_repairitem`：用于将两个损坏的物品修复为一个。
- `minecraft:crafting_special_shielddecoration`：用于将旗帜应用到盾牌上。
- `minecraft:crafting_special_tippedarrow`：用于根据输入药水制作药箭。
- `minecraft:crafting_decorated_pot`：用于用碎片制作饰纹陶罐。

## 熔炉类配方

第二组最重要的配方是通过烧炼或类似过程制作的配方。所有在高炉（类型 `minecraft:smelting`）、烟熏炉（`minecraft:smoking`）、高炉（`minecraft:blasting`）和营火（`minecraft:campfire_cooking`）中制作的配方都使用相同的格式：

```json5
{
    "type": "minecraft:smelting",
    "category": "food",
    "cookingtime": 200,
    "experience": 0.1,
    "ingredient": {
        "item": "minecraft:kelp"
    },
    "result": {
        "id": "minecraft:dried_kelp"
    }
}
```

让我们逐行理解：

- `type`：这是配方序列化器的 ID，`minecraft:smelting`。根据你制作的熔炉类配方的种类，这个值可能不同。
- `category`：此可选字段定义配方书中的类别。
- `cookingtime`：此字段决定配方需要处理的时间，以游戏刻为单位。所有原版熔炉配方使用 200，烟熏炉和高炉使用 100，营火使用 600。但是，这可以是任何你想要的数值。
- `experience`：决定制作此配方时奖励的经验值。此字段是可选的，如果省略则不会获得经验。
- `ingredient`：配方的输入 [配料]。
- `result`：配方的结果。这是 [物品堆叠的 JSON 表示][itemjson]。

这些配方在 `RecipeProvider#buildRecipes` 中的数据生成如下所示：

```java
// 对烟熏配方使用 #smoking，对高炉配方使用 #blasting，对营火配方使用 #campfireCooking。
// 所有这些构建器在其他方面的工作方式相同。
SimpleCookingRecipeBuilder.smelting(
        // 我们的输入配料。
        Ingredient.of(Items.KELP),
        // 我们的配方类别。
        RecipeCategory.FOOD,
        // 我们的结果物品。也可以是 ItemStack。
        Items.DRIED_KELP,
        // 我们的经验奖励。
        0.1f,
        // 我们的烧炼时间。
        200
)
        // 配方进度，与上面的合成配方类似。
        .unlockedBy("has_kelp", this.has(Blocks.KELP))
        // 这个 #save 的重载允许我们指定一个名称。
        .save(this.output, "dried_kelp_smelting");
```

:::info
这些配方的配方类型与它们的配方序列化器相同，即熔炉使用 `minecraft:smelting`，烟熏炉使用 `minecraft:smoking`，依此类推。
:::

## 切石

切石机配方使用 `minecraft:stonecutting` 配方类型。它们简单到不能再简单了，只有类型、输入和输出：

```json5
{
    "type": "minecraft:stonecutting",
    "ingredient": "minecraft:andesite",
    "result": {
        "count": 2,
        "id": "minecraft:andesite_slab"
    }
}
```

`type` 定义了配方序列化器（`minecraft:stonecutting`）。配料是一个 [配料]，结果是一个基本的 [物品堆叠 JSON][itemjson]。与合成配方类似，它们也可以选择性地为配方书中的分组指定一个 `group`。

在 `RecipeProvider#buildRecipes` 中的数据生成也很简单：

```java
SingleItemRecipeBuilder.stonecutting(Ingredient.of(Items.ANDESITE), RecipeCategory.BUILDING_BLOCKS, Items.ANDESITE_SLAB, 2)
        .unlockedBy("has_andesite", this.has(Items.ANDESITE))
        .save(this.output, "andesite_slab_from_andesite_stonecutting");
```

请注意，单物品配方构建器不支持实际的 ItemStack 结果，因此不支持带有数据组件的结果。然而，配方编解码器确实支持它们，所以如果需要此功能，则需要实现自定义构建器。

## 锻造

锻造台支持两种不同的配方序列化器。一种用于将输入转换为输出，同时复制输入组件（如附魔），另一种用于将组件应用到输入上。两者都使用 `minecraft:smithing` 配方类型，并且需要三个输入，分别称为基底、模板和附加物品。

### 转换锻造

此配方序列化器用于将两个输入物品转换为一个，保留第一个输入的数据组件。原版主要将其用于下界合金装备，但是任何物品都可以在这里使用：

```json5
{
    "type": "minecraft:smithing_transform",
    "addition": "#minecraft:netherite_tool_materials",
    "base": "minecraft:diamond_axe",
    "result": {
        "id": "minecraft:netherite_axe"
    },
    "template": "minecraft:netherite_upgrade_smithing_template"
}
```

让我们逐行分解：

- `type`：这是配方序列化器的 ID，`minecraft:smithing_transform`。
- `base`：配方的基底 [配料]。通常，这是某种装备。
- `template`：配方的模板 [配料]。通常，这是一个锻造模板。
- `addition`：配方的附加 [配料]。通常，这是某种材料，例如下界合金锭。
- `result`：配方的结果。这是 [物品堆叠的 JSON 表示][itemjson]。

在数据生成期间，调用 `SmithingTransformRecipeBuilder#smithing` 在 `RecipeProvider#buildRecipes` 中添加你的配方：

```java
SmithingTransformRecipeBuilder.smithing(
        // 模板配料。
        Ingredient.of(Items.NETHERITE_UPGRADE_SMITHING_TEMPLATE),
        // 基底配料。
        Ingredient.of(Items.DIAMOND_AXE),
        // 附加配料。
        this.tag(ItemTags.NETHERITE_TOOL_MATERIALS),
        // 配方书类别。
        RecipeCategory.TOOLS,
        // 结果物品。请注意，虽然配方编解码器在这里接受物品堆叠，但构建器不接受。
        // 如果需要物品堆叠输出，你需要使用自己的构建器。
        Items.NETHERITE_AXE
)
        // 配方进度，与上面的其他配方类似。
        .unlocks("has_netherite_ingot", this.has(ItemTags.NETHERITE_TOOL_MATERIALS))
        // 这个 #save 的重载允许我们指定一个名称。
        .save(this.output, "netherite_axe_smithing");
```

### 盔甲纹饰锻造

盔甲纹饰锻造是将盔甲纹饰应用到盔甲上的过程：

```json5
{
    "type": "minecraft:smithing_trim",
    "addition": "#minecraft:trim_materials",
    "base": "#minecraft:trimmable_armor",
    "pattern": "minecraft:spire",
    "template": "minecraft:bolt_armor_trim_smithing_template"
}
```

再次，让我们将其分解成几个部分：

- `type`：这是配方序列化器的 ID，`minecraft:smithing_trim`。
- `base`：配方的基底 [配料]。所有原版用例在这里都使用 `minecraft:trimmable_armor` 标签。
- `template`：配方的模板 [配料]。所有原版用例在这里都使用盔甲纹饰锻造模板。
- `addition`：配方的附加 [配料]。所有原版用例在这里都使用 `minecraft:trim_materials` 标签。
- `pattern`：应用到基底配料上的纹饰图案。

此配方序列化器明显缺少结果字段。这是因为它使用基底输入并将模板和附加物品“应用”到其上，即它根据其他输入设置基底的组件，并使用该操作的结果作为配方的结果。

在数据生成期间，调用 `SmithingTrimRecipeBuilder#smithingTrim` 在 `RecipeProvider#buildRecipes` 中添加你的配方：

```java
SmithingTrimRecipeBuilder.smithingTrim(
        // 模板配料。
        Ingredient.of(Items.BOLT_ARMOR_TRIM_SMITHING_TEMPLATE),
        // 基底配料。
        this.tag(ItemTags.TRIMMABLE_ARMOR),
        // 附加配料。
        this.tag(ItemTags.TRIM_MATERIALS),
        // 应用到基底的纹饰图案。
        this.registries.lookupOrThrow(Registries.TRIM_PATTERN).getOrThrow(TrimPatterns.SPIRE),
        // 配方书类别。
        RecipeCategory.MISC
)
        // 配方进度，与上面的其他配方类似。
        .unlocks("has_smithing_trim_template", this.has(Items.BOLT_ARMOR_TRIM_SMITHING_TEMPLATE))
        // 这个 #save 的重载允许我们指定一个名称。是的，这个名称是从原版复制来的。
        .save(this.output, "bolt_armor_trim_smithing_template_smithing_trim");
```

[ingredient]: ingredients.md
[itemjson]: ../../../items/index.md#json-representation