# 自定义配方

要添加自定义配方，我们至少需要三样东西：一个 `Recipe`、一个 `RecipeType` 和一个 `RecipeSerializer`。根据你要实现的内容，如果重用现有子类不可行，你可能还需要自定义的 `RecipeInput`、`RecipeDisplay`、`SlotDisplay`、`RecipeBookCategory` 和 `RecipePropertySet`。

为了示例，并突出许多不同的特性，我们将实现一个配方驱动的机制，要求你在世界中用特定物品右键点击一个 `BlockState`，破坏该 `BlockState` 并掉落结果物品。

## 配方输入

让我们从定义我们想要放入配方的内容开始。重要的是要理解，配方输入代表玩家当前正在使用的实际输入。因此，我们不在这里使用标签或配料，而是使用我们拥有的实际物品堆叠和方块状态。

```java
// 我们的输入是一个 BlockState 和一个 ItemStack。
public record RightClickBlockInput(BlockState state, ItemStack stack) implements RecipeInput {
    // 从特定槽位获取物品的方法。我们只有一个堆叠，没有槽位的概念，所以我们假设
    // 槽位 0 持有我们的物品，并对任何其他槽位抛出异常。（取自 SingleRecipeInput#getItem。）
    @Override
    public ItemStack getItem(int slot) {
        if (slot != 0) throw new IllegalArgumentException("No item for index " + slot);
        return this.stack();
    }

    // 我们的输入所需的槽位大小。同样，我们没有真正的槽位概念，所以我们只返回 1，
    // 因为我们涉及一个物品堆叠。具有多个物品的输入应该在此处返回实际数量。
    @Override
    public int size() {
        return 1;
    }
}
```

配方输入不需要以任何方式注册或序列化，因为它们是按需创建的。并非总是需要创建你自己的，原版提供的那些（`CraftingInput`、`SingleRecipeInput` 和 `SmithingRecipeInput`）对于许多用例来说已经足够。

## 配方类

现在我们有了输入，让我们进入配方本身。这是保存我们配方数据的东西，并且也处理匹配和返回配方结果。因此，它通常是你自定义配方中最长的类。

```java
// Recipe<T> 的泛型参数是我们上面的 RightClickBlockInput。
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 我们配方数据的代码内表示。这基本上可以是任何你想要的东西。
    // 这里常见的东西是某种处理时间整数，或经验奖励。
    // 注意，我们现在对输入使用配料而不是物品堆叠。
    private final BlockState inputState;
    private final Ingredient inputItem;
    private final ItemStack result;

    // 添加一个设置所有属性的构造函数。
    public RightClickBlockRecipe(BlockState inputState, Ingredient inputItem, ItemStack result) {
        this.inputState = inputState;
        this.inputItem = inputItem;
        this.result = result;
    }

    // 检查给定的输入是否与此配方匹配。第一个参数匹配泛型。
    // 我们检查方块状态和物品堆叠，只有两者都匹配时才返回 true。
    // 如果需要检查输入的维度，我们也会在这里进行。
    @Override
    public boolean matches(RightClickBlockInput input, Level level) {
        return this.inputState == input.state() && this.inputItem.test(input.stack());
    }

    // 根据给定的输入返回配方结果。第一个参数匹配泛型。
    // 重要：如果使用现有的结果，请务必调用 .copy()！否则，可能会出现问题，而且确实会出现，
    // 因为每个配方只有一个结果，但每次制作配方时都会创建组装好的堆叠。
    @Override
    public ItemStack assemble(RightClickBlockInput input, HolderLookup.Provider registries) {
        return this.result.copy();
    }

    // 当为 true 时，将阻止配方在配方书中同步或在首次使用/解锁时获得进度。
    // 这应该仅在配方不应出现在配方书中时为 true，例如地图扩展。
    // 尽管这个配方接受输入状态，但它仍然可以使用下面的方法在自定义配方书中使用。
    @Override
    public boolean isSpecial() {
        return true;
    }

    // 这个例子概述了最重要的方法。有许多其他方法需要重写。
    // 有些方法将在下面的部分中解释，因为它们不容易在这里压缩和理解。
    // 查看 Recipe 的类定义以查看所有方法。
}
```

## 配方书类别

`RecipeBookCategory` 简单定义了在配方书中显示此配方时所属的组。例如，一个铁镐合成配方将出现在 `RecipeBookCategories#CRAFTING_EQUIPMENT` 中，而一个熟鳕鱼配方将出现在 `#FURNANCE_FOOD` 或 `#SMOKER_FOOD` 中。每个配方都有一个关联的 `RecipeBookCategory`。原版类别可以在 `RecipeBookCategories` 中找到。

:::note
有两个熟鳕鱼配方，一个用于熔炉，一个用于烟熏炉。熔炉和烟熏炉配方有不同的配方书类别。
:::

如果你的配方不适合任何现有类别（通常是因为配方不使用现有的制作站（例如合成台、熔炉）），那么可以创建一个新的 `RecipeBookCategory`。每个 `RecipeBookCategory` 必须 [注册][registry] 到 `BuiltInRegistries#RECIPE_BOOK_CATEGORY`：

```java
/// 对于某个 DeferredRegister<RecipeBookCategory> RECIPE_BOOK_CATEGORIES
public static final Supplier<RecipeBookCategory> RIGHT_CLICK_BLOCK_CATEGORY = RECIPE_BOOK_CATEGORIES.register(
    "right_click_block", RecipeBookCategory::new
);
```

然后，要设置类别，我们必须像这样重写 `#recipeBookCategory`：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 其他内容在这里

    @Override
    public RecipeBookCategory recipeBookCategory() {
        return RIGHT_CLICK_BLOCK_CATEGORY.get();
    }
}
```

### 搜索类别

所有 `RecipeBookCategory` 在技术上都是 `ExtendedRecipeBookCategory`。还有另一种类型的 `ExtendedRecipeBookCategory` 叫做 `SearchRecipeBookCategory`，用于在配方书中查看所有配方时聚合 `RecipeBookCategory`。

NeoForge 允许用户通过模组事件总线上的 `RegisterRecipeBookSearchCategoriesEvent#register` 指定自己的 `ExtendedRecipeBookCategory` 作为搜索类别。`register` 接受代表搜索类别的 `ExtendedRecipeBookCategory` 和组成该搜索类别的 `RecipeBookCategory`。`ExtendedRecipeBookCategory` 搜索类别不需要注册到某些静态的原版注册表。

```java
// 在某个位置
public static final ExtendedRecipeBookCategory RIGHT_CLICK_BLOCK_SEARCH_CATEGORY = new ExtendedRecipeBookCategory() {};

@SubscribeEvent // 在模组事件总线上
public static void registerSearchCategories(RegisterRecipeBookSearchCategoriesEvent event) {
    event.register(
        // 搜索类别
        RIGHT_CLICK_BLOCK_SEARCH_CATEGORY,
        // 作为可变参数，搜索类别内的所有配方类别
        RIGHT_CLICK_BLOCK_CATEGORY.get()
    )
}
```

## 放置信息

`PlacementInfo` 旨在定义配方消费者使用的制作要求，以及它是否可以/如何放入其关联的制作站（例如合成台、熔炉）。`PlacementInfo` 仅用于物品配料，所以如果希望其他类型的配料（例如流体、方块），则需要从头开始实现周围的逻辑。在这些情况下，配方可以标记为不可放置，并通过 `PlacementInfo#NOT_PLACEABLE` 说明。但是，如果你的配方中至少有一个物品类对象，你应该创建一个 `PlacementInfo`。

可以通过 `create` 创建 `PlacementInfo`，它接受一个或一个配料列表；或者通过 `createFromOptionals`，它接受一个可选配料列表。如果你的配方包含某种空槽位的表示，那么应该使用 `createFromOptionals`，为空槽位提供空的可选值：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 其他内容在这里
    private PlacementInfo info;

    @Override
    public PlacementInfo placementInfo() {
        // 如果此时配料未完全填充，则使用此委托。
        // 标签和配方同时加载，这可能是出现这种情况的原因。
        if (this.info == null) {
            // 因为方块状态可能有物品表示，所以使用可选配料
            List<Optional<Ingredient>> ingredients = new ArrayList<>();
            Item stateItem = this.inputState.getBlock().asItem();
            ingredients.add(stateItem != Items.AIR ? Optional.of(Ingredient.of(stateItem)): Optional.empty());
            ingredients.add(Optional.of(this.inputItem));

            // 创建放置信息
            this.info = PlacementInfo.createFromOptionals(ingredients);
        }

        return this.info;
    }
}
```

## 槽位显示

`SlotDisplay` 表示当配方消费者（如配方书）查看时，应该在什么槽位渲染什么信息。`SlotDisplay` 有两个方法。首先是 `resolve`，它接受包含可用注册表和燃料值的 `ContextMap`（如 `SlotDisplayContext` 所示）；以及当前的 `DisplayContentsFactory`，它接受此槽位要显示的内容；并返回转换后的内容列表，以便接受。然后是 `type`，它保存用于编码/解码显示的 [`MapCodec`][codec] 和 [`StreamCodec`][streamcodec]。

`SlotDisplay` 通常通过 [`Ingredient` 的 `#display`，或用于模组配料的 `ICustomIngredient#display`][ingredients] 来实现；但是，在某些情况下，输入可能不是配料，这意味着 `SlotDisplay` 需要使用一个可用的，或者需要创建一个新的。

以下是 Vanilla 和 NeoForge 提供的可用槽位显示：

- `SlotDisplay.Empty`：代表无内容的槽位。
- `SlotDisplay.ItemSlotDisplay`：代表一个物品的槽位。
- `SlotDisplay.ItemStackSlotDisplay`：代表一个物品堆叠的槽位。
- `SlotDisplay.TagSlotDisplay`：代表一个物品标签的槽位。
- `SlotDisplay.WithRemainder`：代表具有某些制作剩余物的输入的槽位。
- `SlotDisplay.AnyFuel`：代表所有燃料物品的槽位。
- `SlotDisplay.Composite`：代表其他槽位显示组合的槽位。
- `SlotDisplay.SmithingTrimDemoSlotDisplay`：代表将随机盔甲纹饰应用到某个具有给定材料的基底的槽位。
- `FluidSlotDisplay`：代表流体的槽位。
- `FluidStackSlotDisplay`：代表流体堆叠的槽位。
- `FluidTagSlotDisplay`：代表流体标签的槽位。

我们的配方中有三个“槽位”：`BlockState` 输入、`Ingredient` 输入和 `ItemStack` 结果。`Ingredient` 输入已经有一个关联的 `SlotDisplay`，`ItemStack` 可以用 `SlotDisplay.ItemStackSlotDisplay` 表示。另一方面，`BlockState` 将需要其自己的自定义 `SlotDisplay` 和 `DisplayContentsFactory`，因为现有的只接受物品堆叠，而在这个例子中，方块状态的处理方式不同。

从 `DisplayContentsFactory` 开始，它旨在作为某种类型到所需内容显示类型的转换器。可用的工厂有：

- `DisplayContentsFactory.ForStacks`：接受 `ItemStack` 的转换器。
- `DisplayContentsFactory.ForRemainders`：接受输入对象和剩余对象列表的转换器。
- `DisplayContentsFactory.ForFluidStacks`：接受 `FluidStack` 的转换器。

有了这个，就可以实现 `DisplayContentsFactory` 将提供的对象转换为所需的输出。例如，`SlotDisplay.ItemStackContentsFactory` 接受 `ForStacks` 转换器，并将堆叠转换为 `ItemStack`。

对于我们的 `BlockState`，我们将创建一个接受状态以及一个输出状态本身的基本实现的工厂。

```java
// 方块状态的基本转换器
public interface ForBlockStates<T> extends DisplayContentsFactory<T> {

    // 委托方法
    default forState(Holder<Block> block) {
        return this.forState(block.value());
    }

    default forState(Block block) {
        return this.forState(block.defaultBlockState());
    }

    // 接受方块状态并转换为所需输出
    T forState(BlockState state);
}

// 方块状态输出的实现
public class BlockStateContentsFactory implements ForBlockStates<BlockState> {
    // 单例实例
    public static final BlockStateContentsFactory INSTANCE = new BlockStateContentsFactory();

    private BlockStateContentsFactory() {}

    @Override
    public BlockState forState(BlockState state) {
        return state;
    }
}

// 物品堆叠输出的实现
public class BlockStateStackContentsFactory implements ForBlockStates<ItemStack> {
    // 单例实例
    public static final BlockStateStackContentsFactory INSTANCE = new BlockStateStackContentsFactory();

    private BlockStateStackContentsFactory() {}

    @Override
    public ItemStack forState(BlockState state) {
        return new ItemStack(state.getBlock());
    }
}
```

然后，有了这个，我们可以创建一个新的 `SlotDisplay`。`SlotDisplay.Type` 必须被 [注册][registry]：

```java
// 一个简单的槽位显示
public record BlockStateSlotDisplay(BlockState state) implements SlotDisplay {
    public static final MapCodec<BlockStateSlotDisplay> CODEC = BlockState.CODEC.fieldOf("state")
        .xmap(BlockStateSlotDisplay::new, BlockStateSlotDisplay::state);
    public static final StreamCodec<RegistryFriendlyByteBuf, BlockStateSlotDisplay> STREAM_CODEC =
        StreamCodec.composite(
            ByteBufCodecs.idMapper(Block.BLOCK_STATE_REGISTRY), BlockStateSlotDisplay::state,
            BlockStateSlotDisplay::new
        );
    
    @Override
    public <T> Stream<T> resolve(ContextMap context, DisplayContentsFactory<T> factory) {
        return switch (factory) {
            // 检查我们的内容工厂并在必要时进行转换
            case ForBlockStates<T> states -> Stream.of(states.forState(this.state));
            // 如果你希望内容根据内容显示以不同方式处理，
            // 那么你可以像这样对其他显示进行 case 匹配
            case ForStacks<T> stacks -> Stream.of(stacks.forStack(state.getBlock().asItem()));
            // 如果没有工厂匹配，则在转换后的流中不返回任何内容
            default -> Stream.empty();
        }
    }

    @Override
    public SlotDisplay.Type<? extends SlotDisplay> type() {
        // 返回下面注册的类型
        return BLOCK_STATE_SLOT_DISPLAY.get();
    }
}

// 在某个注册器类中
/// 对于某个 DeferredRegister<SlotDisplay.Type<?>> SLOT_DISPLAY_TYPES
public static final Supplier<SlotDisplay.Type<BlockStateSlotDisplay>> BLOCK_STATE_SLOT_DISPLAY = SLOT_DISPLAY_TYPES.register(
    "block_state",
    () -> new SlotDisplay.Type<>(BlockStateSlotDisplay.CODEC, BlockStateSlotDisplay.STREAM_CODEC)
);
```

## 配方显示

`RecipeDisplay` 与 `SlotDisplay` 相同，只是它代表整个配方。默认接口只跟踪配方的 `result` 和代表应用配方的工作台的 `craftingStation`。`RecipeDisplay` 也有一个 `type`，它保存用于编码/解码显示的 [`MapCodec`][codec] 和 [`StreamCodec`][streamcodec]。然而，没有任何可用的 `RecipeDisplay` 子类型包含在客户端正确渲染我们配方所需的所有信息。因此，我们需要创建我们自己的 `RecipeDisplay`。

所有槽位和配料都应表示为 `SlotDisplay`。任何限制，例如网格大小，都可以以用户决定的任何方式提供。

```java
// 一个简单的配方显示
public record RightClickBlockRecipeDisplay(
    SlotDisplay inputState,
    SlotDisplay inputItem,
    SlotDisplay result, // 实现 RecipeDisplay#result
    SlotDisplay craftingStation // 实现 RecipeDisplay#craftingStation
) implements RecipeDisplay {
    public static final MapCodec<RightClickBlockRecipeDisplay> MAP_CODEC = RecordCodecBuilder.mapCodec(
        instance -> instance.group(
                    SlotDisplay.CODEC.fieldOf("inputState").forGetter(RightClickBlockRecipeDisplay::inputState),
                    SlotDisplay.CODEC.fieldOf("inputState").forGetter(RightClickBlockRecipeDisplay::inputItem),
                    SlotDisplay.CODEC.fieldOf("result").forGetter(RightClickBlockRecipeDisplay::result),
                    SlotDisplay.CODEC.fieldOf("crafting_station").forGetter(RightClickBlockRecipeDisplay::craftingStation)
                )
                .apply(instance, RightClickBlockRecipeDisplay::new)
    );
    public static final StreamCodec<RegistryFriendlyByteBuf, RightClickBlockRecipeDisplay> STREAM_CODEC = StreamCodec.composite(
        SlotDisplay.STREAM_CODEC,
        RightClickBlockRecipeDisplay::inputState,
        SlotDisplay.STREAM_CODEC,
        RightClickBlockRecipeDisplay::inputItem,
        SlotDisplay.STREAM_CODEC,
        RightClickBlockRecipeDisplay::result,
        SlotDisplay.STREAM_CODEC,
        RightClickBlockRecipeDisplay::craftingStation,
        RightClickBlockRecipeDisplay::new
    );

    @Override
    public RecipeDisplay.Type<? extends RecipeDisplay> type() {
        // 返回下面注册的类型
        return RIGHT_CLICK_BLOCK_RECIPE_DISPLAY.get();
    }
}

// 在某个注册器类中
/// 对于某个 DeferredRegister<RecipeDisplay.Type<?>> RECIPE_DISPLAY_TYPES
public static final Supplier<RecipeDisplay.Type<RightClickBlockRecipeDisplay>> RIGHT_CLICK_BLOCK_RECIPE_DISPLAY = RECIPE_DISPLAY_TYPES.register(
    "right_click_block",
    () -> new RecipeDisplay.Type<>(RightClickBlockRecipeDisplay.CODEC, RightClickBlockRecipeDisplay.STREAM_CODEC)
);
```

然后，我们可以通过像这样重写 `#display` 来为配方创建配方显示：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 其他内容在这里

    @Override
    public List<RecipeDisplay> display() {
        // 同一个配方可以有许多不同的显示
        // 但是这个例子将像其他配方一样只使用一个。
        return List.of(
            // 用指定的槽位添加我们的配方显示
            new RightClickBlockRecipeDisplay(
                new BlockStateSlotDisplay(this.inputState),
                this.inputItem.display(),
                new SlotDisplay.ItemStackSlotDisplay(this.result),
                new SlotDisplay.ItemSlotDisplay(Items.GRASS_BLOCK)
            )
        )
    }
}
```

## 配方类型

接下来，是我们的配方类型。这相当简单，因为除了与配方类型关联的名称之外，没有其他数据。它们是配方系统中两个 [已注册][registry] 的部分之一，因此与所有其他注册表一样，我们创建一个 `DeferredRegister` 并向其注册：

```java
public static final DeferredRegister<RecipeType<?>> RECIPE_TYPES =
        DeferredRegister.create(Registries.RECIPE_TYPE, ExampleMod.MOD_ID);

public static final Supplier<RecipeType<RightClickBlockRecipe>> RIGHT_CLICK_BLOCK_TYPE =
        RECIPE_TYPES.register(
                "right_click_block",
                // 创建配方类型，将 `toString` 设置为类型的注册表名称
                RecipeType::simple
        );
```

注册了我们的配方类型之后，我们必须在我们的配方中重写 `#getType`，像这样：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 其他内容在这里

    @Override
    public RecipeType<? extends Recipe<RightClickBlockInput>> getType() {
        return RIGHT_CLICK_BLOCK_TYPE.get();
    }
}
```

## 配方序列化器

配方序列化器提供两个编解码器，一个 map 编解码器和一个流编解码器，分别用于从/到 JSON 和从/到网络的序列化。本节不会深入探讨编解码器的工作原理，更多信息请参阅 [Map 编解码器][codec] 和 [Stream 编解码器][streamcodec]。

由于配方序列化器可能变得相当大，原版将它们移到单独的类中。建议但不强制遵循此做法——较小的序列化器通常在配方类的字段中以匿名类定义。为了遵循良好实践，我们将创建一个保存我们编解码器的单独类：

```java
// 泛型参数是我们的配方类。
// 注意：这假设简单的 RightClickBlockRecipe#getInputState、#getInputItem 和 #getResult getters 是
// 可用的，这些在以上代码中被省略了。
public class RightClickBlockRecipeSerializer implements RecipeSerializer<RightClickBlockRecipe> {
    public static final MapCodec<RightClickBlockRecipe> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            BlockState.CODEC.fieldOf("state").forGetter(RightClickBlockRecipe::getInputState),
            Ingredient.CODEC.fieldOf("ingredient").forGetter(RightClickBlockRecipe::getInputItem),
            ItemStack.CODEC.fieldOf("result").forGetter(RightClickBlockRecipe::getResult)
    ).apply(inst, RightClickBlockRecipe::new));
    public static final StreamCodec<RegistryFriendlyByteBuf, RightClickBlockRecipe> STREAM_CODEC =
            StreamCodec.composite(
                    ByteBufCodecs.idMapper(Block.BLOCK_STATE_REGISTRY), RightClickBlockRecipe::getInputState,
                    Ingredient.CONTENTS_STREAM_CODEC, RightClickBlockRecipe::getInputItem,
                    ItemStack.STREAM_CODEC, RightClickBlockRecipe::getResult,
                    RightClickBlockRecipe::new
            );

    // 返回我们的 map 编解码器。
    @Override
    public MapCodec<RightClickBlockRecipe> codec() {
        return CODEC;
    }

    // 返回我们的流编解码器。
    @Override
    public StreamCodec<RegistryFriendlyByteBuf, RightClickBlockRecipe> streamCodec() {
        return STREAM_CODEC;
    }
}
```

与类型类似，我们注册我们的序列化器：

```java
public static final DeferredRegister<RecipeType<?>> RECIPE_SERIALIZERS =
        DeferredRegister.create(Registries.RECIPE_SERIALIZER, ExampleMod.MOD_ID);

public static final Supplier<RecipeSerializer<RightClickBlockRecipe>> RIGHT_CLICK_BLOCK =
        RECIPE_SERIALIZERS.register("right_click_block", RightClickBlockRecipeSerializer::new);
```

同样，我们也必须在我们的配方中重写 `#getSerializer`，像这样：

```java
public class RightClickBlockRecipe implements Recipe<RightClickBlockInput> {
    // 其他内容在这里

    @Override
    public RecipeSerializer<? extends Recipe<RightClickBlockInput>> getSerializer() {
        return RIGHT_CLICK_BLOCK.get();
    }
}
```

## 制作机制

现在你的配方的所有部分都完成了，你可以制作一些配方 JSON（请参阅 [数据生成] 部分），然后像上面一样查询配方管理器以获取你的配方。然后你用配方做什么取决于你。一个常见的用例是能够处理你的配方的机器，将活动配方存储为字段。

然而，在我们的案例中，我们希望在右键点击方块时应用配方。我们将使用一个 [事件处理器][event] 来实现这一点。请记住，这是一个示例实现，你可以以任何你喜欢的方式更改它（只要你在服务器上运行它）。由于我们希望交互状态在客户端和服务器上匹配，我们还需要 [通过网络同步任何相关的输入状态][networking]。

我们可以像这样设置一个简单的网络实现来同步配方输入：

```java
// 一个基本的数据包类，必须被注册。
public record ClientboundRightClickBlockRecipesPayload(
    Set<BlockState> inputStates, Set<Holder<Item>> inputItems
) implements CustomPacketPayload {

    // ...
}

// 数据包将数据存储在实例类中。
// 存在于服务器和客户端上以进行初始匹配。
public interface RightClickBlockRecipeInputs {

    Set<BlockState> inputStates();
    Set<Holder<Item>> inputItems();

    default boolean test(BlockState state, ItemStack stack) {
        return this.inputStates().contains(state) && this.inputItems().contains(stack.getItemHolder());
    }
}

// 服务器资源监听器，以便在配方重载时重新加载。
public class ServerRightClickBlockRecipeInputs implements ResourceManagerReloadListener, RightClickBlockRecipeInputs {

    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "block_recipe_inputs");

    private final RecipeManager recipeManager;

    private Set<BlockState> inputStates;
    private Set<Holder<Item>> inputItems;

    public RightClickBlockRecipeInputs(RecipeManager recipeManager) {
        this.recipeManager = recipeManager;
    }

    // 由于 #apply 是根据监听器注册顺序同步触发的，所以在此处设置输入。
    // 配方总是最先应用。
    @Override
    public void onResourceManagerReload(ResourceManager manager) {
        MinecraftServer server = ServerLifecycleHooks.getCurrentServer();
        if (server == null) return; // 永远不应为 null

        // 填充输入
        Set<BlockState> inputStates = new HashSet<>();
        Set<Holder<Item>> inputItems = new HashSet<>();

        this.recipeManager.recipeMap().byType(RIGHT_CLICK_BLOCK_TYPE.get())
            .forEach(holder -> {
                var recipe = holder.value();
                inputStates.add(recipe.getInputState());
                inputItems.addAll(recipe.getInputItem().items());
            });
        
        this.inputStates = Set.copyOf(inputStates);
        this.inputItems = Set.copyOf(inputItems);
    }

    public void syncToClient(Stream<ServerPlayer> players) {
        ClientboundRightClickBlockRecipesPayload payload =
            new ClientboundRightClickBlockRecipesPayload(this.inputStates, this.inputItems);
        players.forEach(player -> PacketDistributor.sendToPlayer(player, payload));
    }

    @Override
    public Set<BlockState> inputStates() {
        return this.inputStates;
    }

    @Override
    public Set<Holder<Item>> inputItems() {
        return this.inputItems;
    }
}

// 客户端实现来保存输入。
public record ClientRightClickBlockRecipeInputs(
    Set<BlockState> inputStates, Set<Holder<Item>> inputItems
) implements RightClickBlockRecipeInputs {

    public ClientRightClickBlockRecipeInputs(Set<BlockState> inputStates, Set<Holder<Item>> inputItems) {
        this.inputStates = Set.copyOf(inputStates);
        this.inputItems = Set.copyOf(inputItems);
    }
}

// 根据端处理配方实例。
public class ServerRightClickBlockRecipes {

    private static ServerRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ServerRightClickBlockRecipes.inputs;
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void addListener(AddServerReloadListenersEvent event) {
        // 注册服务器重载监听器
        ServerRightClickBlockRecipes.inputs = new ServerRightClickBlockRecipeInputs(
            event.getServerResources().getRecipeManager()
        );
        event.addListener(ServerRightClickBlockRecipeInputs.ID, ServerRightClickBlockRecipes.inputs);
        // 确保它在配方之后运行
        event.addDependency(VanillaServerListeners.RECIPES, ServerRightClickBlockRecipeInputs.ID);
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void datapackSync(OnDatapackSyncEvent event) {
        // 发送到客户端
        ServerRightClickBlockRecipes.inputs.syncToClient(event.getRelevantPlayers());
    }
}

public class ClientRightClickBlockRecipes {

    private static ClientRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ClientRightClickBlockRecipes.inputs;
    }

    // 处理发送的数据包
    public static void handle(final ClientboundRightClickBlockRecipesPayload data, final IPayloadContext context) {
        // 在主线程上用数据做一些事情
        ClientRightClickBlockRecipes.inputs = new ClientRightClickBlockRecipeInputs(
            data.inputStates(), data.inputItems()
        );
    }

    @SubscribeEvent // 仅在物理客户端的游戏事件总线上
    public static void clientLogOut(ClientPlayerNetworkEvent.LoggingOut event) {
        // 在世界登出时清除存储的输入
        ClientRightClickBlockRecipes.inputs = null;
    }
}

public class RightClickBlockRecipes {
    // 创建代理方法以正确访问
    public static RightClickBlockRecipeInputs inputs(Level level) {
        return level.isClientSide()
            ? ClientRightClickBlockRecipes.inputs()
            : ServerRightClickBlockRecipes.inputs();
    }
}
```

或者，你可以改为 [将完整配方同步到客户端][clientrecipes]：

```java
// 存在于服务器和客户端上以进行初始匹配。
public interface RightClickBlockRecipeInputs {

    Set<BlockState> inputStates();
    Set<Holder<Item>> inputItems();

    default boolean test(BlockState state, ItemStack stack) {
        return this.inputStates().contains(state) && this.inputItems().contains(stack.getItemHolder());
    }
}

// 服务器资源监听器，以便在配方重载时重新加载。
public class ServerRightClickBlockRecipeInputs implements ResourceManagerReloadListener, RightClickBlockRecipeInputs {

    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "block_recipe_inputs");

    private final RecipeManager recipeManager;

    private Set<BlockState> inputStates;
    private Set<Holder<Item>> inputItems;

    public RightClickBlockRecipeInputs(RecipeManager recipeManager) {
        this.recipeManager = recipeManager;
    }

    // 由于 #apply 是根据监听器注册顺序同步触发的，所以在此处设置输入。
    // 配方总是最先应用。
    @Override
    public void onResourceManagerReload(ResourceManager manager) {
        MinecraftServer server = ServerLifecycleHooks.getCurrentServer();
        if (server != null) { // 永远不应为 null
            // 填充输入
            Set<BlockState> inputStates = new HashSet<>();
            Set<Holder<Item>> inputItems = new HashSet<>();

            this.recipeManager.recipeMap().byType(RIGHT_CLICK_BLOCK_TYPE.get())
                .forEach(holder -> {
                    var recipe = holder.value();
                    inputStates.add(recipe.getInputState());
                    inputItems.addAll(recipe.getInputItem().items());
                });
            
            this.inputStates = Set.copyOf(inputStates);
            this.inputItems = Set.copyOf(inputItems);
        }
    }

    @Override
    public Set<BlockState> inputStates() {
        return this.inputStates;
    }

    @Override
    public Set<Holder<Item>> inputItems() {
        return this.inputItems;
    }
}

// 客户端实现来保存输入。
public record ClientRightClickBlockRecipeInputs(
    Set<BlockState> inputStates, Set<Holder<Item>> inputItems
) implements RightClickBlockRecipeInputs {

    public ClientRightClickBlockRecipeInputs(Set<BlockState> inputStates, Set<Holder<Item>> inputItems) {
        this.inputStates = Set.copyOf(inputStates);
        this.inputItems = Set.copyOf(inputItems);
    }
}

// 根据端处理配方实例。
public class ServerRightClickBlockRecipes {

    private static ServerRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ServerRightClickBlockRecipes.inputs;
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void addListener(AddServerReloadListenersEvent event) {
        // 注册服务器重载监听器
        ServerRightClickBlockRecipes.inputs = new ServerRightClickBlockRecipeInputs(
            event.getServerResources().getRecipeManager()
        );
        event.addListener(ServerRightClickBlockRecipeInputs.ID, ServerRightClickBlockRecipes.inputs);
        // 确保它在配方之后运行
        event.addDependency(VanillaServerListeners.RECIPES, ServerRightClickBlockRecipeInputs.ID);
    }

    @SubscribeEvent // 在游戏事件总线上
    public static void datapackSync(OnDatapackSyncEvent event) {
        // 指定要同步到客户端的配方类型
        event.sendRecipes(RIGHT_CLICK_BLOCK_TYPE.get());
    }
}

public class ClientRightClickBlockRecipes {

    private static ClientRightClickBlockRecipeInputs inputs;

    public static RightClickBlockRecipeInputs inputs() {
        return ClientRightClickBlockRecipes.inputs;
    }

    @SubscribeEvent // 仅在物理客户端的游戏事件总线上
    public static void recipesReceived(RecipesReceivedEvent event) {
        // 存储配方
        Set<BlockState> inputStates = new HashSet<>();
        Set<Holder<Item>> inputItems = new HashSet<>();

        event.getRecipeMap().byType(RIGHT_CLICK_BLOCK_TYPE.get())
            .forEach(holder -> {
                var recipe = holder.value();
                inputStates.add(recipe.getInputState());
                inputItems.addAll(recipe.getInputItem().items());
            });
        
        ClientRightClickBlockRecipes.inputs = new ClientRightClickBlockRecipeInputs(
            inputStates, inputItems
        );
    }

    @SubscribeEvent // 仅在物理客户端的游戏事件总线上
    public static void clientLogOut(ClientPlayerNetworkEvent.LoggingOut event) {
        // 在世界登出时清除存储的输入
        ClientRightClickBlockRecipes.inputs = null;
    }
}

public class RightClickBlockRecipes {
    // 创建代理方法以正确访问
    public static RightClickBlockRecipeInputs inputs(Level level) {
        return level.isClientSide()
            ? ClientRightClickBlockRecipes.inputs()
            : ServerRightClickBlockRecipes.inputs();
    }
}
```

然后，使用同步的输入，我们可以检查游戏中使用的输入：

```java
@SubscribeEvent // 在游戏事件总线上
public static void useItemOnBlock(UseItemOnBlockEvent event) {
    // 如果我们不在事件由方块决定的阶段，则跳过。有关详细信息，请参阅事件的 javadocs。
    if (event.getUsePhase() != UseItemOnBlockEvent.UsePhase.BLOCK) return;
    // 获取参数以首先检查输入
    Level level = event.getLevel();
    BlockPos pos = event.getPos();
    BlockState blockState = level.getBlockState(pos);
    ItemStack itemStack = event.getItemStack();

    // 检查输入是否可能在两端产生配方
    if (!RightClickBlockRecipes.inputs(level).test(blockState, itemStack)) return;

    // 如果是这样，在检查配方之前确保在服务器端
    if (!level.isClientSide() && level instanceof ServerLevel serverLevel) {
        // 创建一个输入并查询配方。
        RightClickBlockInput input = new RightClickBlockInput(blockState, itemStack);
        Optional<RecipeHolder<? extends Recipe<CraftingInput>>> optional = serverLevel.recipeAccess().getRecipeFor(
            // 配方类型。
            RIGHT_CLICK_BLOCK_TYPE.get(),
            input,
            level
        );
        ItemStack result = optional
            .map(RecipeHolder::value)
            .map(e -> e.assemble(input, level.registryAccess()))
            .orElse(ItemStack.EMPTY);
        
        // 如果有结果，则破坏方块并在世界中掉落结果。
        if (!result.isEmpty()) {
            level.removeBlock(pos, false);
            ItemEntity entity = new ItemEntity(level,
                    // pos 的中心。
                    pos.getX() + 0.5, pos.getY() + 0.5, pos.getZ() + 0.5,
                    result);
            level.addFreshEntity(entity);
        }
    }

    // 取消事件以停止交互管道，无论在哪一端。
    // 已经确保可能会有结果。
    event.cancelWithResult(InteractionResult.SUCCESS_SERVER);
}
```

## 数据生成

要为你自己的配方序列化器创建配方构建器，你需要实现 `RecipeBuilder` 及其方法。一个常见的实现，部分复制自原版，看起来像这样：

```java
// 这个类是抽象的，因为有很多每个配方序列化器的逻辑。
// 它的目的是展示所有（原版）配方构建器的共同部分。
public abstract class SimpleRecipeBuilder implements RecipeBuilder {
    // 将字段设为 protected，以便我们的子类可以使用它们。
    protected final ItemStack result;
    protected final Map<String, Criterion<?>> criteria = new LinkedHashMap<>();
    @Nullable
    protected final String group;

    // 构造函数通常接受结果物品堆叠是常见的。
    // 或者，静态构建器方法也是可能的。
    public SimpleRecipeBuilder(ItemStack result) {
        this.result = result;
    }

    // 此方法为配方进度添加条件。
    @Override
    public SimpleRecipeBuilder unlockedBy(String name, Criterion<?> criterion) {
        this.criteria.put(name, criterion);
        return this;
    }

    // 此方法添加配方书组。如果你不想使用配方书组，
    // 移除 this.group 字段并使此方法无操作（即返回 this）。
    @Override
    public SimpleRecipeBuilder group(@Nullable String group) {
        this.group = group;
        return this;
    }

    // 原版在这里需要一个 Item，而不是一个 ItemStack。你仍然可以而且应该使用 ItemStack
    // 来序列化配方。
    @Override
    public Item getResult() {
        return this.result.getItem();
    }
}
```

所以我们有了配方构建器的基础。现在，在我们继续处理配方序列化器依赖的部分之前，我们应该首先考虑制作我们的配方工厂。在我们的案例中，直接使用构造函数是有意义的。在其他情况下，使用静态辅助方法或小的函数式接口是可行的方法。如果你使用一个构建器来处理多个配方类，这尤其相关。

利用 `RightClickBlockRecipe::new` 作为我们的配方工厂，并重用上面的 `SimpleRecipeBuilder` 类，我们可以为 `RightClickBlockRecipe` 创建以下配方构建器：

```java
public class RightClickBlockRecipeBuilder extends SimpleRecipeBuilder {
    private final BlockState inputState;
    private final Ingredient inputItem;

    // 由于我们每种输入都恰好有一个，我们将它们传递给构造函数。
    // 对于具有某种配料列表的配方序列化器的构建器，通常会
    // 初始化一个空列表，并具有 #addIngredient 或类似的方法。
    public RightClickBlockRecipeBuilder(ItemStack result, BlockState inputState, Ingredient inputItem) {
        super(result);
        this.inputState = inputState;
        this.inputItem = inputItem;
    }

    // 使用给定的 RecipeOutput 和 key 保存配方。此方法在 RecipeBuilder 接口中定义。
    @Override
    public void save(RecipeOutput output, ResourceKey<Recipe<?>> key) {
        // 构建进度。
        Advancement.Builder advancement = output.advancement()
                .addCriterion("has_the_recipe", RecipeUnlockedTrigger.unlocked(key))
                .rewards(AdvancementRewards.Builder.recipe(key))
                .requirements(AdvancementRequirements.Strategy.OR);
        this.criteria.forEach(advancement::addCriterion);
        // 我们的工厂参数是结果、方块状态和配料。
        RightClickBlockRecipe recipe = new RightClickBlockRecipe(this.inputState, this.inputItem, this.result);
        // 将 id、配方和配方进度传递给 RecipeOutput。
        output.accept(key, recipe, advancement.build(key.location().withPrefix("recipes/")));
    }
}
```

现在，在 [数据生成][recipedatagen] 期间，你可以像调用其他构建器一样调用你的配方构建器：

```java
@Override
protected void buildRecipes(RecipeOutput output) {
    new RightClickRecipeBuilder(
            // 我们的构造函数参数。这个例子添加了广受欢迎的泥土 -> 钻石转换。
            new ItemStack(Items.DIAMOND),
            Blocks.DIRT.defaultBlockState(),
            Ingredient.of(Items.APPLE)
    )
            .unlockedBy("has_apple", this.has(Items.APPLE))
            .save(output);
    // 其他配方构建器在这里
}
```

:::note
也可以将 `SimpleRecipeBuilder` 合并到 `RightClickBlockRecipeBuilder`（或你自己的配方构建器）中，特别是如果你只有一个或两个配方构建器。这里的抽象旨在展示构建器的哪些部分是配方依赖的，哪些不是。
:::

[clientrecipes]: index.md#client-side-recipes
[codec]: ../../../datastorage/codecs.md
[datagen]: #data-generation
[event]: ../../../concepts/events.md
[ingredients]: ingredients.md
[networking]: ../../../networking/payload.md
[recipedatagen]: index.md#data-generation
[registry]: ../../../concepts/registries.md#methods-for-registering
[streamcodec]: ../../../networking/streamcodecs.md