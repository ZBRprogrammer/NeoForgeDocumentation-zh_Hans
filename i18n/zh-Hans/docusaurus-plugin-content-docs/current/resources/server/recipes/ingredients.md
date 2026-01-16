# 配料

`Ingredient` 用于 [配方] 中，以检查给定的 [`ItemStack`][itemstack] 是否是配方的有效输入。为此，`Ingredient` 实现了 `Predicate<ItemStack>`，并且可以调用 `#test` 来确认给定的 `ItemStack` 是否匹配配料。

不幸的是，`Ingredient` 的许多内部结构很混乱。NeoForge 通过尽可能忽略 `Ingredient` 类来解决这个问题，而是为自定义配料引入了 `ICustomIngredient` 接口。这不是对常规 `Ingredient` 的直接替代，但我们可以分别使用 `ICustomIngredient#toVanilla` 和 `Ingredient#getCustomIngredient` 与 `Ingredient` 相互转换。

## 内置配料类型

获取配料的最简单方法是使用 `Ingredient#of` 辅助方法。存在几种变体：

- `Ingredient.of()` 返回一个空的配料。
- `Ingredient.of(Blocks.IRON_BLOCK, Items.GOLD_BLOCK)` 返回一个接受铁块或金块的配料。参数是 [`ItemLike`s][itemlike] 的可变参数，这意味着可以使用任意数量的方块和物品。
- `Ingredient.of(Stream.of(Items.DIAMOND_SWORD))` 返回一个接受物品的配料。与上一个方法类似，但是使用 `Stream<ItemLike>`，如果你恰好有一个的话。
- `Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.WOODEN_SLABS))` 返回一个接受指定 [标签] 中任何物品的配料，例如任何木质台阶。

此外，NeoForge 添加了一些额外的配料：

- `BlockTagIngredient.of(BlockTags.CONVERTABLE_TO_MUD)` 返回一个类似于 `Ingredient.of()` 的标签变体的配料，但是使用方块标签。这应该用于你会使用物品标签，但只有方块标签可用的情况（例如 `minecraft:convertable_to_mud`）。
- `CustomDisplayIngredient.of(Ingredient.of(Items.DIRT), SlotDisplay.Empty.INSTANCE)` 返回一个具有自定义 [`SlotDisplay`][slotdisplay] 的配料，你提供该显示以确定槽位如何在客户端渲染时被消耗。
- `CompoundIngredient.of(Ingredient.of(Items.DIRT))` 返回一个具有子配料的配料，在构造函数（可变参数）中传入。如果其任何子配料匹配，则该配料匹配。
- `DataComponentIngredient.of(true, new ItemStack(Items.DIAMOND_SWORD))` 返回一个除了物品之外，还匹配数据组件的配料。布尔参数表示严格匹配（true）或部分匹配（false）。严格匹配意味着数据组件必须完全匹配，而部分匹配意味着数据组件必须匹配，但其他数据组件也可能存在。存在 `#of` 的其他重载，允许指定多个 `Item`，或提供其他选项。
- `DifferenceIngredient.of(Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.PLANKS)), Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.NON_FLAMMABLE_WOOD)))` 返回一个匹配第一个配料中所有内容但不匹配第二个配料的配料。给定的例子只匹配可以燃烧的木板（即除了绯红木板、诡异木板和模组下界木木板之外的所有木板）。
- `IntersectionIngredient.of(Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.PLANKS)), Ingredient.of(BuiltInRegistries.ITEM.getOrThrow(ItemTags.NON_FLAMMABLE_WOOD)))` 返回一个匹配同时匹配两个子配料的所有内容的配料。给定的例子只匹配不能燃烧的木板（即绯红木板、诡异木板和模组下界木木板）。

:::note
如果你使用数据生成，并接受用于标签实例的 `HolderSet` 的配料（那些调用 `Registry#getOrThrow` 的），那么该 `HolderSet` 应该通过 `HolderLookup.Provider` 获取，使用 `HolderLookup.Provider#lookupOrThrow` 获取物品注册表，并使用 `HolderGetter#getOrThrow` 和 `TagKey` 获取持有器集合。
:::

请记住，NeoForge 提供的配料类型是 `ICustomIngredient`，必须在原版上下文中使用它们之前调用 `#toVanilla`，如本文开头所述。

## 自定义配料类型

模组开发者可以通过 `ICustomIngredient` 系统添加他们的自定义配料类型。为了示例，让我们制作一个附魔物品配料，它接受一个物品标签和一个附魔到最小等级的映射：

```java
public class MinEnchantedIngredient implements ICustomIngredient {
    private final TagKey<Item> tag;
    private final Map<Holder<Enchantment>, Integer> enchantments;
    // 用于序列化配料的编解码器。
    public static final MapCodec<MinEnchantedIngredient> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            TagKey.codec(Registries.ITEM).fieldOf("tag").forGetter(e -> e.tag),
            Codec.unboundedMap(Enchantment.CODEC, Codec.INT)
                    .optionalFieldOf("enchantments", Map.of())
                    .forGetter(e -> e.enchantments)
    ).apply(inst, MinEnchantedIngredient::new));
    // 从常规编解码器创建流编解码器。在某些情况下，从头定义新的流编解码器可能更有意义。
    public static final StreamCodec<RegistryFriendlyByteBuf, MinEnchantedIngredient> STREAM_CODEC =
            ByteBufCodecs.fromCodecWithRegistries(CODEC.codec());

    // 允许传入预存在的附魔到等级的映射。
    public MinEnchantedIngredient(TagKey<Item> tag, Map<Holder<Enchantment>, Integer> enchantments) {
        this.tag = tag;
        this.enchantments = enchantments;
    }

    // 通过验证物品是否在标签中以及检查所有所需附魔的存在且至少达到所需等级，检查传递的 ItemStack 是否匹配我们的配料。
    @Override
    public boolean test(ItemStack stack) {
        return stack.is(tag) && enchantments.keySet()
                .stream()
                .allMatch(ench -> EnchantmentHelper.getEnchantmentsForCrafting(stack).getLevel(ench) >= enchantments.get(ench));
    }

    // 确定此配料是否执行 NBT 或数据组件匹配（false）或不执行（true）。
    // 也确定是否使用流编解码器进行同步，稍后详述。
    // 我们查询堆叠上的附魔，因此我们的配料不是简单的。
    @Override
    public boolean isSimple() {
        return false;
    }

    // 返回匹配此配料的物品流。主要用于显示目的。
    // 这里有一些好的做法需要遵循：
    // - 始终至少包含一个物品，以防止意外识别为空。
    // - 每个接受的物品至少包含一次。
    // - 如果 #isSimple 为 true，这应该是精确的并包含每个匹配的物品。
    //   如果不为 true，这应尽可能精确，但不需要超级准确。
    // 在我们的例子中，我们使用标签中的所有物品。
    @Override
    public Stream<Holder<Item>> items() {
        return BuiltInRegistries.ITEM.getOrThrow(tag).stream();
    }
}
```

自定义配料是一个 [注册表]，因此我们必须注册我们的配料。我们使用 NeoForge 提供的 `IngredientType` 类来注册，它基本上是围绕一个 [`MapCodec`][codec] 和可选的 [`StreamCodec`][streamcodec] 的包装器。

```java
public static final DeferredRegister<IngredientType<?>> INGREDIENT_TYPES =
        DeferredRegister.create(NeoForgeRegistries.Keys.INGREDIENT_TYPE, ExampleMod.MOD_ID);

public static final Supplier<IngredientType<MinEnchantedIngredient>> MIN_ENCHANTED =
        INGREDIENT_TYPES.register("min_enchanted",
                // 流编解码器参数是可选的，如果未指定流编解码器，
                // 将使用 ByteBufCodecs#fromCodec 或 #fromCodecWithRegistries 从编解码器创建。
                () -> new IngredientType<>(MinEnchantedIngredient.CODEC, MinEnchantedIngredient.STREAM_CODEC));
```

当我们完成这些后，还需要在我们的配料类中重写 `#getType`：

```java
public class MinEnchantedIngredient implements ICustomIngredient {
    // 其他内容在这里

    @Override    
    public IngredientType<?> getType() {
        return MIN_ENCHANTED.get();
    }
}
```

搞定了！我们的配料类型已经可以使用了。

## JSON 表示

由于原版配料非常有限，而 NeoForge 为它们引入了一个全新的注册表，因此也值得看看内置和我们自己的配料在 JSON 中的样子。

作为对象并指定 `neoforge:ingredient_type` 的配料通常被认为是非原版的。例如：

```json5
{
    "neoforge:ingredient_type": "neoforge:block_tag",
    "tag": "minecraft:convertable_to_mud"
}
```

或另一个使用我们自己配料的例子：

```json5
{
    "neoforge:ingredient_type": "examplemod:min_enchanted",
    "tag": "c:swords",
    "enchantments": {
        "minecraft:sharpness": 4
    }
}
```

如果配料是一个字符串，意味着未指定 `neoforge:ingredient_type`，那么我们有一个原版配料。原版配料是代表物品的字符串，或者当以 `#` 为前缀时代表标签。

一个原版物品配料的例子：

```json5
"minecraft:dirt"
```

一个原版标签配料的例子：

```json5
"#c:ingots"
```

[codec]: ../../../datastorage/codecs.md
[itemlike]: ../../../items/index.md#itemlike
[itemstack]: ../../../items/index.md#itemstacks
[recipes]: index.md
[registry]: ../../../concepts/registries.md
[slotdisplay]: index.md#slot-displays
[streamcodec]: ../../../networking/streamcodecs.md
[tag]: ../tags.md