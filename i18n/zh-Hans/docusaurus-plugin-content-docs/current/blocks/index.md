# 方块(Blocks)

方块是Minecraft世界的基础。它们构成了所有地形、结构和机器。如果您有兴趣制作模组，那么您可能会想添加一些方块。本页将指导您创建方块，并介绍一些可以用它们做的事情。

## 独一无二的方块(One Block to Rule Them All)

在我们开始之前，重要的是要理解游戏中每种方块都只有一个实例。一个世界由对该方块在不同位置的数千个引用组成。换句话说，同一个方块只是被显示了很多次。

因此，一个方块应该只实例化一次，那就是在[注册][registration]期间。一旦方块被注册，您就可以根据需要使用的注册引用。

与大多数其他注册表不同，方块可以使用一个特殊版本的`DeferredRegister`，称为`DeferredRegister.Blocks`。`DeferredRegister.Blocks`基本上像一个`DeferredRegister<Block>`，但有一些细微差别：

- 它们通过`DeferredRegister.createBlocks("yourmodid")`创建，而不是常规的`DeferredRegister.create(...)`方法。
- `#register`返回一个`DeferredBlock<T extends Block>`，它扩展了`DeferredHolder<Block, T>`。`T`是我们正在注册的方块类的类型。
- 有一些用于注册方块的辅助方法。更多细节请参见[下文][below]。

那么现在，让我们注册我们的方块：

``` java
//BLOCKS is a DeferredRegister.Blocks
public static final DeferredBlock<Block> MY_BLOCK = BLOCKS.register("my_block", registryName -> new Block(...));
```

注册方块后，对新`my_block`的所有引用都应使用此常量。例如，如果您想检查给定位置的方块是否为`my_block`，代码将如下所示：

``` java
level.getBlockState(position) // returns the blockstate placed in the given level (world) at the given position
    //highlight-next-line
    .is(MyBlockRegistrationClass.MY_BLOCK);
```

这种方法还有一个方便的效果，即`block1 == block2`可以工作，并且可以替代Java的`equals`方法（当然，使用`equals`仍然有效，但因为没有必要，因为它是通过引用比较的）。

:::danger
不要在注册之外调用`new Block()`！一旦您这样做，事情可能会并且将会出错：

- 必须在注册表未冻结时创建方块。NeoForge会为您解冻注册表并在之后冻结它们，因此注册是您创建方块的窗口期。
- 如果您尝试在注册表再次冻结时创建和/或注册方块，游戏将崩溃并报告一个`null`方块，这可能非常令人困惑。
- 如果您仍然设法拥有一个悬挂的方块实例，游戏在同步和保存时将无法识别它，并将其替换为空气。
:::

## 创建方块

如前所述，我们首先创建我们的`DeferredRegister.Blocks`：

``` java
public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("yourmodid");
```

### 基础方块

对于不需要特殊功能的简单方块（如圆石、木板等），可以直接使用`Block`类。为此，在注册期间，使用`BlockBehaviour.Properties`参数实例化`Block`。这个`BlockBehaviour.Properties`参数可以使用`BlockBehaviour.Properties#of`创建，并且可以通过调用其方法进行自定义。其中最重要的方法有：

- `setId` - 设置方块的资源键(Resource Key)。
    - 这**必须**在每个方块上设置；否则，将抛出异常。
- `destroyTime` - 确定方块需要被破坏的时间。
    - 石头的破坏时间为1.5，泥土为0.5，黑曜石为50，基岩为-1（不可破坏）。
- `explosionResistance` - 确定方块的爆炸抗性。
    - 石头的爆炸抗性为6.0，泥土为0.5，黑曜石为1200，基岩为3,600,000。
- `sound` - 设置方块被敲击、破坏或放置时发出的声音。
    - 默认值为`SoundType.STONE`。更多细节请参阅[声音页面][sounds]。
- `lightLevel` - 设置方块的光照发射。接受一个带有`BlockState`参数的函数，返回一个介于0到15之间的值。
    - 例如，荧石使用`state -> 15`，火把使用`state -> 14`。
- `friction` - 设置方块的摩擦系数（光滑度）。
    - 默认值为0.6。冰使用0.98。

因此，一个简单的实现将如下所示：

``` java
//BLOCKS is a DeferredRegister.Blocks
public static final DeferredBlock<Block> MY_BETTER_BLOCK = BLOCKS.register(
    "my_better_block", 
    registryName -> new Block(BlockBehaviour.Properties.of()
        //highlight-start
        .setId(ResourceKey.create(Registries.BLOCK, registryName))
        .destroyTime(2.0f)
        .explosionResistance(10.0f)
        .sound(SoundType.GRAVEL)
        .lightLevel(state -> 7)
        //highlight-end
    ));
```

更多文档，请参见`BlockBehaviour.Properties`的源代码。更多示例，或查看Minecraft使用的值，请查看`Blocks`类。

:::note
重要的是要理解，世界中的方块与物品栏中的方块不是同一个东西。在物品栏中看起来像方块的实际上是一个`BlockItem`，一种特殊的[物品][item]类型，使用时放置一个方块。这也意味着诸如创造标签页或最大堆叠数之类的事情由相应的`BlockItem`处理。

`BlockItem`必须与方块分开注册。这是因为方块不一定需要物品，例如如果它不打算被收集（例如火就是这种情况）。
:::

### 更多功能

直接使用`Block`只允许非常基础的方块。如果您想添加功能，比如玩家交互或不同的碰撞箱，需要一个扩展`Block`的自定义类。`Block`类有许多可以重写以执行不同操作的方法；有关更多信息，请参见`Block`、`BlockBehaviour`和`IBlockExtension`类。另请参阅下面的[使用方块][usingblocks]部分，了解方块最常见的一些用例。

如果您想制作具有不同变体的方块（想想有底部、顶部和双变体的台阶），您应该使用[方块状态][blockstates]。最后，如果您想要一个存储额外数据的方块（想想存储其物品栏的箱子），应使用[方块实体][blockentities]。这里的经验法则是，如果您有有限且数量合理的状态（最多几百个状态），请使用方块状态；如果您有无限或接近无限的状态数量，请使用方块实体。

#### 方块类型

方块类型是用于序列化和反序列化方块对象的[`MapCodec`s][codec]。这个`MapCodec`通过`BlockBehaviour#codec`设置，并[注册][registration]到方块类型注册表中。目前，它的唯一用途是在生成方块列表报告时使用。应为`Block`的每个子类创建一个方块类型。例如，`FlowerBlock#CODEC`代表大多数花的方块类型，而其子类`WitherRoseBlock`具有单独的方块类型。

如果方块子类只接受`BlockBehaviour.Properties`，则可以使用`BlockBehaviour#simpleCodec`来创建`MapCodec`。

``` java
// For some block subclass
public class SimpleBlock extends Block {
    public SimpleBlock(BlockBehavior.Properties properties) {
        // ...
    }

    @Override
    public MapCodec<SimpleBlock> codec() {
        return SIMPLE_CODEC.get();
    }
}

// In some registration class
public static final DeferredRegister<MapCodec<? extends Block>> REGISTRAR = DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");

public static final Supplier<MapCodec<SimpleBlock>> SIMPLE_CODEC = REGISTRAR.register(
    "simple",
    () -> BlockBehaviour.simpleCodec(SimpleBlock::new)
);
```

如果方块子类包含更多参数，则应使用[`RecordCodecBuilder#mapCodec`][codec]来创建`MapCodec`，为`BlockBehaviour.Properties`参数传递`BlockBehaviour#propertiesCodec`。

``` java
// For some block subclass
public class ComplexBlock extends Block {
    public ComplexBlock(int value, BlockBehavior.Properties properties) {
        // ...
    }

    @Override
    public MapCodec<ComplexBlock> codec() {
        return COMPLEX_CODEC.get();
    }

    public int getValue() {
        return this.value;
    }
}

// In some registration class
public static final DeferredRegister<MapCodec<? extends Block>> REGISTRAR = DeferredRegister.create(BuiltInRegistries.BLOCK_TYPE, "yourmodid");

public static final Supplier<MapCodec<ComplexBlock>> COMPLEX_CODEC = REGISTRAR.register(
    "simple",
    () -> RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            Codec.INT.fieldOf("value").forGetter(ComplexBlock::getValue),
            BlockBehaviour.propertiesCodec() // represents the BlockBehavior.Properties parameter
        ).apply(instance, ComplexBlock::new)
    )
);
```

:::note
尽管方块类型目前基本上未被使用，但随着Mojang继续朝着以编解码器为中心的结构发展，预计它将来会变得更加重要。
:::

### `DeferredRegister.Blocks` 辅助方法

我们已经讨论了如何创建`DeferredRegister.Blocks`[上文][above]，以及它返回`DeferredBlock`。现在，让我们看看这个专门的`DeferredRegister`还提供了哪些其他实用工具。让我们从`#registerBlock`开始：

``` java
public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks("yourmodid");

public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.register(
    "example_block", registryName -> new Block(
        BlockBehaviour.Properties.of()
            // The ID must be set on the block
            .setId(ResourceKey.create(Registries.BLOCK, registryName))
    )
);

// Same as above, except that the block properties are constructed eagerly.
// setId is also called internally on the properties object.
public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerBlock(
    "example_block",
    Block::new, // The factory that the properties will be passed into.
    BlockBehaviour.Properties.of() // The properties to use.
);
```

如果你想使用`Block::new`，你可以完全省略工厂：

``` java
public static final DeferredBlock<Block> EXAMPLE_BLOCK = BLOCKS.registerSimpleBlock(
    "example_block",
    BlockBehaviour.Properties.of() // The properties to use.
);
```

这与前面的示例完全相同，但稍微短一些。当然，如果你想使用`Block`的子类而不是`Block`本身，你将不得不使用前面的方法。

### 资源(Resources)

如果你注册了你的方块并将其放置在世界中，你会发现它缺少一些东西，比如纹理。这是因为[纹理][textures]等是由Minecraft的资源系统处理的。在Minecraft中添加新方块时，你应该编写或[生成][datagen]以下文件：

- 一个[方块状态文件][bsfile]
- 一个[方块模型][model]
- 一个[翻译][i18n]
- 一个[战利品表][loottable]
- 一些方块[标签][tags]，例如用于挖掘

对于以上所有内容，请参考类似的原版方块的文件和数据生成器。

## 使用方块

方块很少直接用于做事情。事实上，可能Minecraft中最常见的两个操作——获取位置处的方块和在位置处设置方块——使用的是方块状态，而不是方块。通常的设计方法是让方块定义行为，但让行为实际上通过方块状态运行。因此，`BlockState`通常作为参数传递给`Block`的方法。有关如何使用方块状态以及如何从方块获取方块状态的更多信息，请参阅[使用方块状态][usingblockstates]。

在多种情况下，`Block`的多个方法在不同时间被使用。以下小节列出了最常见的方块相关流程。除非另有说明，否则所有方法都在逻辑两端调用，并且应在两端返回相同的结果。

### 放置方块

方块放置逻辑从`BlockItem#useOn`（或其某个子类的实现，例如用于睡莲的`PlaceOnWaterBlockItem`）调用。有关游戏如何到达那里的更多信息，请参阅[右键点击物品][rightclick]。实际上，这意味着一旦`BlockItem`被右键点击（例如一个圆石物品），就会调用此行为。

- 检查几个先决条件，例如您是否处于旁观模式，方块的所有必需功能标志是否已启用，或者目标位置是否不在世界边界之外。如果其中至少一个检查失败，流程结束。
- 为当前尝试放置方块的位置处的方块调用`BlockBehaviour#canBeReplaced`。如果返回`false`，流程结束。这里返回`true`的突出情况是高草丛或雪层。
- 调用`Block#getStateForPlacement`。在这里，根据上下文（包括位置、旋转和方块放置的面等信息），可以返回不同的方块状态。这对于可以放置在不同方向的方块非常有用。
- 使用上一步获得的方块状态调用`BlockBehaviour#canSurvive`。如果返回`false`，流程结束。
- 通过`Level#setBlock`调用将方块状态设置到世界中。
    - 在该`Level#setBlock`调用中，调用`BlockBehaviour#onPlace`。
- 调用`Block#setPlacedBy`。

### 破坏方块

破坏方块稍微复杂一些，因为它需要时间。这个过程大致可以分为三个阶段：“开始”、“挖掘”和“实际破坏”。

- 当点击左键时，进入“开始”阶段。
- 现在，需要按住左键，进入“挖掘”阶段。**此阶段的方法每个刻都会被调用。**
- 如果“继续”阶段没有被中断（通过释放左键）并且方块被破坏，则进入“实际破坏”阶段。

或者，对于那些喜欢伪代码的人：

``` java
leftClick();
initiatingStage();
while (leftClickIsBeingHeld()) {
    miningStage();
    if (blockIsBroken()) {
        actuallyBreakingStage();
        break;
    }
}
```

以下小节将这些阶段进一步细分为实际的方法调用。有关游戏如何从左键点击到达此流程的信息，请参阅[左键点击物品][leftclick]。

#### “开始”阶段

- 检查几个先决条件，例如您是否处于旁观模式，主手中`ItemStack`的所有必需功能标志是否已启用，或者相关方块是否不在世界边界之外。如果其中至少一个检查失败，流程结束。
- 触发`PlayerInteractEvent.LeftClickBlock`。如果事件被取消，流程结束。
    - 注意：当事件在客户端被取消时，不会向服务器发送数据包，因此服务器上不会运行任何逻辑。
    - 但是，在服务器上取消此事件仍会导致客户端代码运行，这可能导致不同步！
- 调用`BlockBehaviour#attack`。

#### “挖掘”阶段

- 触发`PlayerInteractEvent.LeftClickBlock`。如果事件被取消，流程移动到“完成”阶段。
    - 注意：当事件在客户端被取消时，不会向服务器发送数据包，因此服务器上不会运行任何逻辑。
    - 但是，在服务器上取消此事件仍会导致客户端代码运行，这可能导致不同步！
- 调用`BlockBehaviour#getDestroyProgress`并添加到内部破坏进度计数器。
    - `BlockBehaviour#getDestroyProgress`返回一个介于0和1之间的浮点值，表示每个刻应增加多少破坏进度计数器。
- 相应地更新进度覆盖（裂痕纹理）。
- 如果破坏进度大于1.0（即完成，即方块应被破坏），则退出“挖掘”阶段并进入“实际破坏”阶段。

#### “实际破坏”阶段

- 调用`Item#canDestroyBlock`。如果返回`false`（确定方块不应被破坏），流程移动到“完成”阶段。
- 如果方块是`GameMasterBlock`的实例，则调用`Player#canUseGameMasterBlocks`。这确定玩家是否有能力破坏仅限创造模式的方块。如果为`false`，流程移动到“完成”阶段。
- 仅服务器：调用`Player#blockActionRestricted`。这确定当前玩家是否可以破坏方块。如果为`true`，流程移动到“完成”阶段。
- 仅服务器：触发`BlockEvent.BreakEvent`。如果被取消，流程移动到“完成”阶段。初始取消状态由上述三个方法确定。
- 调用`Block#playerWillDestroy`。
- 仅服务器：调用`IBlockExtension#canHarvestBlock`。这确定方块是否可以被收获，即破坏时掉落物品。如果`Player#preventsBlockDrops`返回true，则忽略此方法。
    - 仅服务器：如果`IBlockExtension#canHarvestBlock`没有被重写且没有调用其父方法，则触发`PlayerEvent.HarvestCheck`。如果`HarvestCheck#canHarvest`返回`false`，那么`Block#playerDestroy`将不会被调用，阻止任何资源或经验掉落。
- 仅服务器：调用`Item#mineBlock`。
- 调用`IBlockExtension#onDestroyedByPlayer`。如果返回`false`，流程移动到“完成”阶段。
    - 通过以`Blocks.AIR.defaultBlockState()`或当前记录的流体作为方块状态参数的`Level#setBlock`调用，从世界中移除方块状态。
        - 在该`Level#setBlock`调用中，调用`Block#onRemove`。
    - 如果`IBlockExtension#onDestroyedByPlayer`返回`true`，则调用`Block#destroy`。
- 仅服务器：如果之前的`IBlockExtension#canHarvestBlock`和`IBlockExtension#onDestroyedByPlayer`调用返回`true`，则调用`Block#playerDestroy`。
    - 仅服务器：调用`Block#dropResources`。这确定方块被挖掘时掉落什么，包括经验。
        - 仅服务器：触发`BlockDropsEvent`。如果事件被取消，则方块破坏时不会掉落任何东西。否则，`BlockDropsEvent#getDrops`中的每个`ItemEntity`都会被添加到当前世界中。此外，如果`getDroppedExperience`大于0，则调用`Block#popExperience`。
            - 仅服务器：调用`IBlockExtension#getExpDrop`，并由`EnchantmentHelper#processBlockExperience`增强。这是在可能被修改之前为`BlockDropsEvent#getDroppedExperience`设置的初始值。
- 仅服务器：如果用于挖掘方块的物品在上述过程中的任何时候损坏，则触发`PlayerDestroyItemEvent`。

#### 挖掘速度

挖掘速度根据方块的硬度、使用的[工具][tool]的速度以及几个实体[属性][attributes]按照以下规则计算：

``` java
// This will return the tool's mining speed, or 1 if the held item is either empty, not a tool,
// or not applicable for the block being broken.
float destroySpeed = item.getDestroySpeed(blockState);
// If we have an applicable tool, add the minecraft:mining_efficiency attribute as an additive modifier.
if (destroySpeed > 1) {
    destroySpeed += player.getAttributeValue(Attributes.MINING_EFFICIENCY);
}
// Apply effects from haste or conduit power.
if (player.hasEffect(MobEffects.HASTE) || player.hasEffect(MobEffects.CONDUIT_POWER)) {
    int haste = player.hasEffect(MobEffects.HASTE)
        ? player.getEffect(MobEffects.HASTE).getAmplifier()
        : 0;
    int conduitPower = player.hasEffect(MobEffects.CONDUIT_POWER)
        ? player.getEffect(MobEffects.CONDUIT_POWER).getAmplifier()
        : 0;
    int amplifier = Math.max(haste, conduitPower);
    destroySpeed *= 1 + (amplifier + 1) * 0.2f;
}
// Apply slowness effect.
if (player.hasEffect(MobEffects.MINING_FATIGUE)) {
    destroySpeed *= switch (player.getEffect(MobEffects.MINING_FATIGUE).getAmplifier()) {
        case 0 -> 0.3F;
        case 1 -> 0.09F;
        case 2 -> 0.0027F;
        default -> 8.1E-4F;
    };
}
// Add the minecraft:block_break_speed attribute as a multiplicative modifier.
destroySpeed *= player.getAttributeValue(Attributes.BLOCK_BREAK_SPEED);
// If the player is underwater, apply the underwater mining speed penalty multiplicatively.
if (player.isEyeInFluid(FluidTags.WATER)) {
    destroySpeed *= player.getAttributeValue(Attributes.SUBMERGED_MINING_SPEED);
}
// If the player is trying to break a block in mid-air, make the player mine 5 times slower.
if (!player.onGround()) {
    destroySpeed /= 5;
}
destroySpeed = /* The PlayerEvent.BreakSpeed event is fired here, allowing modders to further modify this value. */;
return destroySpeed;
```

具体的代码可以参考`Player#getDestroySpeed`。

### 刻处理

刻处理是一种每1/20秒或50毫秒（“一刻”）更新（刻处理）游戏部分的机制。方块提供了在不同的方式下调用的不同刻处理方法。

#### 服务器刻处理和刻调度

`BlockBehaviour#tick`在两种情况下被调用：要么通过默认的[随机刻][randomtick]（见下文），要么通过预定的刻。预定的刻可以通过`Level#scheduleTick(BlockPos, Block, int)`创建，其中`int`表示延迟。原版在许多地方使用这个系统，例如，大型垂滴叶的倾斜机制严重依赖这个系统。其他突出的用户是各种红石组件。

#### 客户端刻处理

`Block#animateTick`专门在客户端调用，每帧一次。这是客户端专用行为发生的地方，例如火把粒子生成。

#### 天气刻处理

天气刻处理由`Block#handlePrecipitation`处理，并且独立于常规刻处理运行。它仅在服务器上调用，仅当以某种形式下雨时，有1/16的几率调用。例如，这用于在雨或雪中填满的炼药锅。

#### 随机刻

随机刻系统独立于常规刻处理运行。随机刻必须通过方块的`BlockBehaviour.Properties`调用`BlockBehaviour.Properties#randomTicks()`方法来启用。这使得方块可以成为随机刻机制的一部分。

每个刻，区块中一定数量的方块会发生随机刻。该数量由`randomTickSpeed`游戏规则定义。默认值为3时，每个刻选择区块中的3个随机方块。如果这些方块启用了随机刻，那么它们各自的`BlockBehaviour#randomTick`方法将被调用。

随机刻被Minecraft中广泛使用的机制使用，例如植物生长、冰和雪融化，或铜氧化。

[above]: #one-block-to-rule-them-all
[attributes]: ../entities/attributes.md
[below]: #deferredregisterblocks-helpers
[blockentities]: ../blockentities/index.md
[blockstates]: states.md
[bsfile]: ../resources/client/models/index.md#blockstate-files
[codec]: ../datastorage/codecs.md#records
[datagen]: ../resources/index.md#data-generation
[i18n]: ../resources/client/i18n.md
[item]: ../items/index.md
[leftclick]: ../items/interactions.md#left-clicking-an-item
[loottable]: ../resources/server/loottables/index.md
[model]: ../resources/client/models/index.md
[randomtick]: #random-ticking
[registration]: ../concepts/registries.md#methods-for-registering
[resources]: ../resources/index.md#assets
[rightclick]: ../items/interactions.md#right-clicking-an-item
[sounds]: ../resources/client/sounds.md
[tags]: ../resources/server/tags.md
[textures]: ../resources/client/textures.md
[tool]: ../items/tools.md
[usingblocks]: #using-blocks
[usingblockstates]: states.md#using-blockstates