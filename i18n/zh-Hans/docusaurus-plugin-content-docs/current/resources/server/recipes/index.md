import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 配方

配方是在 Minecraft 世界中将一组对象转换为其他对象的一种方式。尽管 Minecraft 纯粹将此系统用于物品转换，但该系统的构建方式允许任何类型的对象——方块、实体等——被转换。除非另有明确说明，本文中几乎所有的配方都使用配方数据文件；一个“配方”指的是数据驱动的配方。

配方数据文件位于 `data/<命名空间>/recipe/<路径>.json`。例如，配方 `minecraft:diamond_block` 位于 `data/minecraft/recipe/diamond_block.json`。

## 术语

- **配方 JSON** 或 **配方文件** 是由 `RecipeManager` 加载和存储的 JSON 文件。它包含配方类型、输入和输出等信息，以及附加信息（例如处理时间）。
- **`Recipe`** 保存所有 JSON 字段的代码内表示，以及匹配逻辑（“此输入是否与配方匹配？”）和一些其他属性。
- **`RecipeInput`** 是向配方提供输入的类型。有多个子类，例如 `CraftingInput` 或 `SingleRecipeInput`（用于熔炉等）。
- **配方配料** 或简称 **配料** 是配方的单个输入（而 `RecipeInput` 通常代表要检查配方配料的输入集合）。配料是一个非常强大的系统，因此 [在它们自己的文章中][ingredients] 进行了概述。
- **`PlacementInfo`** 是配方包含的物品以及它们应填充的索引的定义。如果配方在某种程度上无法根据提供的物品捕获（例如，仅更改数据组件），则使用 `PlacementInfo#NOT_PLACEABLE`。
- **`SlotDisplay`** 定义单个槽位在配方查看器（如配方书）中应如何显示。
- **`RecipeDisplay`** 定义配方的 `SlotDisplay`，供配方查看器（如配方书）使用。虽然该接口仅包含配方结果和进行配方的工作站的方法，但子类型可以捕获配料或网格大小等信息。
- **`RecipeManager`** 是服务器上的一个单例字段，保存所有加载的配方。
- **`RecipeSerializer`** 基本上是围绕 [`MapCodec`][codec] 和 [`StreamCodec`][streamcodec] 的包装器，两者都用于序列化。
- **`RecipeType`** 是 `Recipe` 的注册类型等价物。主要在按类型查找配方时使用。作为经验法则，不同的制作容器应使用不同的 `RecipeType`。例如，`minecraft:crafting` 配方类型涵盖 `minecraft:crafting_shaped` 和 `minecraft:crafting_shapeless` 配方序列化器，以及特殊合成序列化器。
- **`RecipeBookCategory`** 是在配方书中查看时代表某些配方的组。
- **配方 [进度]** 是负责在配方书中解锁配方的进度。它们不是必需的，玩家通常更青睐配方查看器模组而忽视它们，但是 [配方数据提供器][datagen] 会为你生成它们，因此建议顺其自然。
- **`RecipePropertySet`** 定义了菜单中定义输入槽位可接受的可用配料列表。
- **`RecipeBuilder`** 在数据生成期间用于创建 JSON 配方。
- **配方工厂** 是从 `RecipeBuilder` 创建 `Recipe` 时使用的方法引用。它可以是对构造函数的引用，也可以是静态构建器方法，或者是专门为此目的创建的函数式接口（通常称为 `Factory`）。

## JSON 规范

配方文件的内容根据所选类型差异很大。所有配方文件共有的属性是 `type` 和 [`neoforge:conditions`][conditions] 属性：

```json5
{
    // 配方类型。这映射到配方序列化器注册表中的条目。
    "type": "minecraft:crafting_shaped",
    // 数据加载条件列表。可选，NeoForge 添加。更多信息请参阅上面的文章链接。
    "neoforge:conditions": [ /*...*/ ]
}
```

Minecraft 提供的完整类型列表可以在 [内置配方类型文章][builtin] 中找到。模组也可以 [定义自己的配方类型][customrecipes]。

## 使用配方

配方通过 `RecipeManager` 类加载、存储和获取，而该类又通过 `ServerLevel#recipeAccess` 或 - 如果你没有可用的 `ServerLevel` - `ServerLifecycleHooks.getCurrentServer()#getRecipeManager` 获取。服务器默认不同步配方到客户端，而是仅发送用于限制菜单槽位输入的 `RecipePropertySet`。此外，每当配方在配方书中解锁时，其 `RecipeDisplay` 和相应的 `RecipeDisplayEntry` 会发送到客户端（不包括所有 `Recipe#isSpecial` 返回 true 的配方）。因此，配方逻辑应始终在服务器上运行。

获取配方的最简单方法是通过其资源键：

```java
RecipeManager recipes = serverLevel.recipeAccess();
// RecipeHolder<?> 是资源键和配方本身的记录。
Optional<RecipeHolder<?>> optional = recipes.byKey(
    ResourceKey.create(Registries.RECIPE, ResourceLocation.withDefaultNamespace("diamond_block"))
);
optional.map(RecipeHolder::value).ifPresent(recipe -> {
    // 在这里对配方做任何你想做的事情。请注意，配方可以是任何类型。
});
```

一个更实用的方法是构造一个 `RecipeInput` 并尝试获取匹配的配方。在这个例子中，我们将使用 `CraftingInput#of` 创建一个包含一个钻石方块的 `CraftingInput`。这将创建一个无序输入，有序输入将使用 `CraftingInput#ofPositioned`，其他输入将使用其他 `RecipeInput`（例如，熔炉配方通常使用 `new SingleRecipeInput`）。

```java
RecipeManager recipes = serverLevel.recipeAccess();
// 构造一个 RecipeInput，根据配方要求。例如，为合成配方构造一个 CraftingInput。
// 参数分别是宽度、高度和物品。
CraftingInput input = CraftingInput.of(1, 1, List.of(new ItemStack(Items.DIAMOND_BLOCK)));
// RecipeHolder 上的泛型通配符应扩展 CraftingRecipe。
// 这允许稍后更多的类型安全。
Optional<RecipeHolder<? extends CraftingRecipe>> optional = recipes.getRecipeFor(
        // 获取配方的配方类型。在我们的例子中，我们使用合成类型。
        RecipeType.CRAFTING,
        // 我们的配方输入。
        input,
        // 我们的世界上下文。
        serverLevel
);
// 这将返回钻石方块 -> 9 个钻石的配方（除非数据包更改了该配方）。
optional.map(RecipeHolder::value).ifPresent(recipe -> {
    // 在这里做任何你想做的事情。注意，配方现在是 CraftingRecipe 而不是 Recipe<?>。
});
```

或者，你也可以获取一个可能为空的匹配你的输入的配方列表，这在可以合理假设有多个配方匹配的情况下特别有用：

```java
RecipeManager recipes = serverLevel.recipeAccess();
CraftingInput input = CraftingInput.of(1, 1, List.of(new ItemStack(Items.DIAMOND_BLOCK)));
// 这些不是 Optionals，可以直接使用。但是，列表可能为空，表示没有匹配的配方。
Stream<RecipeHolder<? extends Recipe<CraftingInput>>> list = recipes.recipeMap().getRecipesFor(
    // 与上述相同的参数。
    RecipeType.CRAFTING, input, serverLevel
);
```

一旦我们有了正确的配方输入，我们也想获取配方输出。这是通过调用 `Recipe#assemble` 完成的：

```java
RecipeManager recipes = serverLevel.recipeAccess();
CraftingInput input = CraftingInput.of(...);
Optional<RecipeHolder<? extends CraftingRecipe>> optional = recipes.getRecipeFor(...);
// 使用 ItemStack.EMPTY 作为后备。
ItemStack result = optional
        .map(RecipeHolder::value)
        .map(recipe -> recipe.assemble(input, serverLevel.registryAccess()))
        .orElse(ItemStack.EMPTY);
```

如果需要，也可以遍历某种类型的所有配方。操作如下：

```java
RecipeManager recipes = serverLevel.recipeAccess();
// 像之前一样，传递所需的配方类型。
Collection<RecipeHolder<?>> list = recipes.recipeMap().byType(RecipeType.CRAFTING);
```

## 配方优先级

有时，配方可能与其他配方重叠，通常是因为一个图案使用特定物品，而另一个相同图案使用包含该物品的标签。在这些情况下，原版使用它找到的第一个配方，这由首先读取和加载的配方决定。这可能是一个问题，因为如果特定物品配方在基于标签的配方之后加载，那么特定物品配方将永远无法获得。

为了解决这个问题，NeoForge 引入了配方优先级来排序应该首先显示哪些配方。条目表示为配方注册表键到整数优先级值的映射。优先级值根据最高值排序，未指定的配方默认为 `0`。这意味着优先级大于 `0` 的配方首先排序，而优先级小于 `0` 的配方最后排序。优先级映射位于 `data/<命名空间>/recipe_priorities.json`，其中所有配方优先级合并在一起，除非 `replace` 为 true，这将清除所有先前加载的条目。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
{
    // 为 true 时，清除所有先前加载的条目。
    "replace": false,
    // 配方条目到其优先级值的映射。
    // 如果配方没有优先级，则默认为 0。
    "entries": {
        // 指向 'data/examplemod/recipe/higher_priority.json'
        // 此配方将在任何默认值之前检查。
        "examplemod:higher_priority": 1,
        // 指向 'data/examplemod/recipe/lower_priority.json'
        // 此配方将在任何默认值之后检查。
        "examplemod:lower_priority": -1,
        // 指向 'data/examplemod/recipe/even_lower_priority.json'
        // 此配方将在任何默认值以及 'lower_priority' 配方之后检查。
        "examplemod:even_lower_priority": -2
    }
}
```

</TabItem>
<TabItem value="datagen" label="Datagen">

```java
// 生成配方优先级
public class ExamplePrioritiesProvider extends RecipePrioritiesProvider {

    public ExamplePrioritiesProvider(PackOutput output, CompletableFuture<HolderLookup.Provider> registries) {
        // 将 'examplemod' 替换为你的模组 ID。
        super(output, registries, "examplemod");
    }

    @Override
    protected void start() {
        // 将配方条目注册到优先级值。

        this.add(
            // 指向 'data/examplemod/recipe/higher_priority.json'
            ResourceKey.create(Registries.RECIPE, ResourceLocation.fromNamespaceAndPath("examplemod", "higher_priority")),
            // 此配方将在任何默认值之前检查。
            1
        );

        this.add(
            // 指向 'data/examplemod/recipe/lower_priority.json'
            ResourceLocation.fromNamespaceAndPath("examplemod", "lower_priority"),
            // 此配方将在任何默认值之后检查。
            -1
        );

        this.add(
            // 指向 'data/examplemod/recipe/even_lower_priority.json'
            // 命名空间从传递给提供器的模组 ID 推断。
            "even_lower_priority",
            // 此配方将在任何默认值以及 'lower_priority' 配方之后检查。
            -2
        );
    }
}
```

</TabItem>
</Tabs>

## 其他配方机制

原版中的一些机制通常被认为是配方，但在代码中的实现方式不同。这通常是由于历史遗留原因，或者因为“配方”是从其他数据（例如 [标签]）构建的。

:::warning
配方查看器模组通常不会拾取这些配方。对这些模组的支持必须手动添加，更多信息请参阅相应模组的文档。
:::

### 铁砧配方

铁砧有两个输入槽和一个输出槽。唯一的原版用例是工具修复、合并和重命名，由于每个用例都需要特殊处理，因此不提供配方文件。但是，可以使用 `AnvilUpdateEvent` 构建系统。这个 [事件] 允许获取输入（左侧输入槽）和材料（右侧输入槽），并允许设置输出物品堆叠，以及经验成本和要消耗的材料数量。也可以通过 [取消][cancel] 事件来完全阻止该过程。

```java
// 这个例子允许用一整组泥土修复石镐，消耗半组，需要 3 级经验。
@SubscribeEvent // 在游戏事件总线上
public static void onAnvilUpdate(AnvilUpdateEvent event) {
    ItemStack left = event.getLeft();
    ItemStack right = event.getRight();
    if (left.is(Items.STONE_PICKAXE) && right.is(Items.DIRT) && right.getCount() >= 32) {
        event.setOutput(new ItemStack(Items.STONE_PICKAXE));
        event.setMaterialCost(32);
        event.setXpCost(3);
    }
}
```

### 酿造

请参阅 [药水效果与药水文章中的酿造章节][brewing]。

### 扩展合成网格大小

负责保存有序合成配方内存表示的 `ShapedRecipePattern` 类硬编码了 3x3 槽位的限制，阻碍了想要添加更大的合成表同时重用原版有序合成配方类型的模组。为了解决这个问题，NeoForge 打补丁添加了一个名为 `ShapedRecipePattern#setCraftingSize(int width, int height)` 的静态方法，允许增加限制。应该在 `FMLCommonSetupEvent` 期间调用它。最大的值胜出，例如，如果一个模组添加了一个 4x6 的合成表，另一个添加了一个 6x5 的合成表，那么结果值将是 6x6。

:::danger
`ShapedRecipePattern#setCraftingSize` 不是线程安全的。必须将其包装在 `event#enqueueWork` 调用中。
:::

### 客户端配方

默认情况下，原版不会向 [逻辑客户端][logicalside] 发送任何配方。相反，会同步 `RecipePropertySet` / `SelectableRecipe.SingleInputSet` 以处理用户交互期间适当的客户端行为。此外，当配方在配方书中解锁时，其 `RecipeDisplay` 会同步。但是，这两种情况的范围有限，特别是当需要从配方本身获取更多数据时。在这些情况下，NeoForge 提供了一种将给定 `RecipeType` 的完整配方发送到客户端的方法。

必须在 [游戏事件总线][events] 上监听两个事件：`OnDatapackSyncEvent` 和 `RecipesReceivedEvent`。首先，通过调用 `OnDatapackSyncEvent#sendRecipes` 指定要同步到客户端的 `RecipeType`。然后，可以通过 `RecipesReceivedEvent#getRecipeMap` 从提供的 `RecipeMap` 访问配方。此外，一旦玩家退出世界，应通过 `ClientPlayerNetworkEvent.LoggingOut` 清除客户端上存储的任何配方。

```java
// 假设我们有一些自定义的 RecipeType<ExampleRecipe> EXAMPLE_RECIPE_TYPE

@SubscribeEvent // 在游戏事件总线上
public static void datapackSync(OnDatapackSyncEvent event) {
    // 指定要同步到客户端的配方类型
    event.sendRecipes(EXAMPLE_RECIPE_TYPE);
}

// 仅在物理客户端的某个类中

private static final List<RecipeHolder<ExampleRecipe>> EXAMPLE_RECIPES = new ArrayList<>();

@SubscribeEvent // 仅在物理客户端的游戏事件总线上
public static void recipesReceived(RecipesReceivedEvent event) {
    // 首先移除先前的配方
    EXAMPLE_RECIPES.clear();

    // 然后存储你想要的配方
    EXAMPLE_RECIPES.addAll(event.getRecipeMap().byType(EXAMPLE_RECIPE_TYPE));
}

@SubscribeEvent // 仅在物理客户端的游戏事件总线上
public static void clientLogOut(ClientPlayerNetworkEvent.LoggingOut event) {
    // 在世界登出时清除存储的配方
    EXAMPLE_RECIPES.clear();
}
```

:::warning
如果你计划为你的配方类型同步配方，应在物理两端调用 `OnDatapackSyncEvent`。所有世界，包括单人游戏，在服务器和客户端之间都有划分，这意味着从服务器引用数据包注册表条目到客户端可能会导致游戏崩溃。
:::

## 数据生成

像大多数其他 JSON 文件一样，配方可以数据生成。对于配方，我们希望扩展 `RecipeProvider` 类并重写 `#buildRecipes`，并扩展 `RecipeProvider.Runner` 类以传递给数据生成器：

```java
public class MyRecipeProvider extends RecipeProvider {

    // 构造要运行的提供器
    protected MyRecipeProvider(HolderLookup.Provider provider, RecipeOutput output) {
        super(provider, output);
    }
 
    @Override
    protected void buildRecipes() {
        // 在此处添加你的配方。
    }

    // 要添加到数据生成器的运行器
    public static class Runner extends RecipeProvider.Runner {
        // 从 `GatherDataEvent`s 获取参数。
        public Runner(PackOutput output, CompletableFuture<HolderLookup.Provider> lookupProvider) {
            super(output, lookupProvider);
        }

        @Override
        protected RecipeProvider createRecipeProvider(HolderLookup.Provider provider, RecipeOutput output) {
            return new MyRecipeProvider(provider, output);
        }
    }
}
```

值得注意的是 `RecipeOutput` 参数。Minecraft 使用此对象自动为你生成配方进度。除此之外，NeoForge 向 `RecipeOutput` 注入了 [条件] 支持，可以通过 `#withConditions` 调用。

配方本身通常通过 `RecipeBuilder` 的子类添加。列出所有原版配方构建器超出了本文的范围（它们在 [内置配方类型文章][builtin] 中解释），但是创建你自己的构建器在 [自定义配方页面][customdatagen] 中有解释。

与所有其他数据提供器一样，必须像这样将配方提供器注册到 `GatherDataEvent`s：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    // 如果添加数据包对象，首先调用 event.createDatapackRegistryObjects(...)

    event.createProvider(MyRecipeProvider.Runner::new);
}
```

配方提供器还添加了常见场景的帮助器，例如 `twoByTwoPacker`（用于 2x2 方块配方）、`threeByThreePacker`（用于 3x3 方块配方）或 `nineBlockStorageRecipes`（用于 3x3 方块配方和 1 个方块到 9 个物品的配方）。

[advancement]: ../advancements.md
[brewing]: ../../../items/mobeffects.md#brewing
[builtin]: builtin.md
[cancel]: ../../../concepts/events.md#cancellable-events
[codec]: ../../../datastorage/codecs.md
[conditions]: ../conditions.md
[customdatagen]: custom.md#data-generation
[customrecipes]: custom.md
[datagen]: #data-generation
[event]: ../../../concepts/events.md
[ingredients]: ingredients.md
[logicalside]: ../../../concepts/sides.md#the-logical-side
[streamcodec]: ../../../networking/streamcodecs.md
[tags]: ../tags.md