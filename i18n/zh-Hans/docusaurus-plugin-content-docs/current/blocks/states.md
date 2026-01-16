# 方块状态(Blockstates)

通常，你会遇到想要方块有不同状态的情况。例如，小麦作物有八个生长阶段，为每个阶段制作一个单独的方块感觉不对。或者你有一个台阶或类似台阶的方块——一个底部状态，一个顶部状态，以及一个两者都有的状态。

这就是方块状态发挥作用的地方。方块状态是一种表示方块可以具有的不同状态的简单方法，例如生长阶段或台阶放置类型。

## 方块状态属性

方块状态使用一个属性系统。一个方块可以有多个不同类型的属性。例如，末地传送门框架有两个属性：是否有一只眼睛（`eye`，2个选项）以及它被放置的方向（`facing`，4个选项）。因此，末地传送门框架总共有8（2 * 4）种不同的方块状态：
``` java
minecraft:end_portal_frame[facing=north,eye=false]
minecraft:end_portal_frame[facing=east,eye=false]
minecraft:end_portal_frame[facing=south,eye=false]
minecraft:end_portal_frame[facing=west,eye=false]
minecraft:end_portal_frame[facing=north,eye=true]
minecraft:end_portal_frame[facing=east,eye=true]
minecraft:end_portal_frame[facing=south,eye=true]
minecraft:end_portal_frame[facing=west,eye=true]
```

符号`blockid[property1=value1,property2=value,...]`是表示方块状态的标准化文本形式，并在原版的某些位置使用，例如在命令中。

如果你的方块没有定义任何方块状态属性，它仍然有一个方块状态——那就是没有任何属性的那个，因为没有属性可指定。这可以表示为`minecraft:oak_planks[]`或简单的`minecraft:oak_planks`。

与方块一样，每个`BlockState`在内存中只存在一次。这意味着`==`可以并且应该用于比较`BlockState`。`BlockState`也是一个最终类，意味着它不能被扩展。**任何功能都放在相应的[方块][block]类中！**

## 何时使用方块状态

### 方块状态 vs. 独立方块

一个好的经验法则是：**如果它有不同的名称，它应该是一个独立的方块。** 一个例子是制作椅子方块：椅子的方向应该是一个属性，而不同类型的木头应该分成不同的方块。所以每种木头类型都有一个椅子方块，每个椅子方块有四种方块状态（每个方向一种）。

### 方块状态 vs. [方块实体][blockentity]

这里的经验法则是：**如果你有有限数量的状态，使用方块状态；如果你有无限或接近无限数量的状态，使用方块实体。** 方块实体可以存储任意数量的数据，但比方块状态慢。

方块状态和方块实体可以结合使用。例如，箱子使用方块状态属性来处理方向、是否被水logged或成为双箱子等事情，而存储物品栏、当前是否打开或与漏斗交互则由方块实体处理。

对于“多少个状态对于方块状态来说太多了？”这个问题没有确定的答案，但我们建议如果你需要超过8-9位的数据（即超过几百个状态），你应该使用方块实体。

## 实现方块状态

要实现方块状态属性，在你的方块类中，创建或引用一个`public static final Property<?>`常量。虽然你可以自由创建自己的`Property<?>`实现，但原版代码提供了几个方便的覆盖大多数用例的实现：

- `IntegerProperty`
    - 实现`Property<Integer>`。定义一个持有整数值的属性。注意不支持负值。
    - 通过调用`IntegerProperty#create(String propertyName, int minimum, int maximum)`创建。
- `BooleanProperty`
    - 实现`Property<Boolean>`。定义一个持有`true`或`false`值的属性。
    - 通过调用`BooleanProperty#create(String propertyName)`创建。
- `EnumProperty<E extends Enum<E>>`
    - 实现`Property<E>`。定义一个可以取枚举类值的属性。
    - 通过调用`EnumProperty#create(String propertyName, Class<E> enumClass)`创建。
    - 也可以只使用枚举值的一个子集（例如16种`DyeColor`中的4种），参见`EnumProperty#create`的重载。

`BlockStateProperties`类包含共享的原版属性，只要可能就应该使用或引用它们，而不是创建自己的属性。

一旦你有了属性常量，在你的方块类中重写`Block#createBlockStateDefinition(StateDefinition.Builder)`。在该方法中，调用`StateDefinition.Builder#add(YOUR_PROPERTY);`。`StateDefinition.Builder#add`有一个可变参数，所以如果你有多个属性，你可以一次性添加它们。

每个方块也会有一个默认状态。如果没有指定其他内容，默认状态使用每个属性的默认值。你可以通过从构造函数中调用`Block#registerDefaultState(BlockState)`方法来更改默认状态。

如果你希望更改放置方块时使用的`BlockState`，重写`Block#getStateForPlacement(BlockPlaceContext)`。这可以用于，例如，根据玩家放置方块时的站立位置或视线方向来设置方块的方向。

为了进一步说明这一点，以下是`EndPortalFrameBlock`类的相关部分：

``` java
public class EndPortalFrameBlock extends Block {
    // Note: It is possible to directly use the values in BlockStateProperties instead of referencing them here again.
    // However, for the sake of simplicity and readability, it is recommended to add constants like this.
    public static final EnumProperty<Direction> FACING = BlockStateProperties.FACING;
    public static final BooleanProperty EYE = BlockStateProperties.EYE;

    public EndPortalFrameBlock(BlockBehaviour.Properties pProperties) {
        super(pProperties);
        // stateDefinition.any() returns a random BlockState from an internal set,
        // we don't care because we're setting all values ourselves anyway
        this.registerDefaultState(stateDefinition.any()
                .setValue(FACING, Direction.NORTH)
                .setValue(EYE, false)
        );
    }

    @Override
    protected void createBlockStateDefinition(StateDefinition.Builder<Block, BlockState> pBuilder) {
        // this is where the properties are actually added to the state
        pBuilder.add(FACING, EYE);
    }

    @Override
    @Nullable
    public BlockState getStateForPlacement(BlockPlaceContext pContext) {
        // code that determines which state will be used when
        // placing down this block, depending on the BlockPlaceContext
    }
}
```

## 使用方块状态

要从`Block`转到`BlockState`，调用`Block#defaultBlockState()`。默认方块状态可以通过`Block#registerDefaultState`更改，如上所述。

你可以通过调用`BlockState#getValue(Property<?>)`来获取属性的值，传递给它你想要获取值的属性。重用我们的末地传送门框架示例，这将如下所示：

``` java
// EndPortalFrameBlock.FACING is an EnumPropery<Direction> and thus can be used to obtain a Direction from the BlockState
Direction direction = endPortalFrameBlockState.getValue(EndPortalFrameBlock.FACING);
```

如果你想获得具有不同值集的`BlockState`，只需在现有的方块状态上调用`BlockState#setValue(Property<T>, T)`并指定属性及其值。以我们的末地传送门框架为例，如下所示：

``` java
endPortalFrameBlockState = endPortalFrameBlockState.setValue(EndPortalFrameBlock.FACING, Direction.SOUTH);
```

:::note
`BlockState`是不可变的。这意味着当你调用`#setValue(Property<T>, T)`时，你实际上并没有修改方块状态。相反，内部会执行查找，并返回你请求的方块状态对象，这是具有这些确切属性值的唯一对象。这也意味着仅仅调用`state#setValue`而不将其保存到变量中（例如保存回`state`）是没有作用的。
:::

要从世界中获取`BlockState`，使用`Level#getBlockState(BlockPos)`。

### `Level#setBlock`

要在世界中设置`BlockState`，使用`Level#setBlock(BlockPos, BlockState, int)`。

`int`参数值得额外解释，因为它的含义并不立即明显。它表示所谓的更新标志。

为了帮助正确设置更新标志，`Block`中有许多以`UPDATE_`为前缀的`int`常量。这些常量可以按位或在一起（例如`Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS`）如果你想组合它们。

- `Block.UPDATE_NEIGHBORS`向相邻方块发送更新。更具体地说，它调用`Block#neighborChanged`，这调用许多方法，其中大多数以某种方式与红石相关。
- `Block.UPDATE_CLIENTS`将方块更新同步到客户端。
- `Block.UPDATE_INVISIBLE`明确不在客户端更新。这也覆盖了`Block.UPDATE_CLIENTS`，导致更新不同步。方块总是在服务器上更新。
- `Block.UPDATE_IMMEDIATE`强制在客户端的主线程上重新渲染。
- `Block.UPDATE_KNOWN_SHAPE`停止邻居更新递归。
- `Block.UPDATE_SUPPRESS_DROPS`禁用该位置旧方块的方块掉落。
- `Block.UPDATE_MOVE_BY_PISTON`仅被活塞代码用于表示方块被活塞移动。这主要负责延迟光照引擎更新。
- `Block.UPDATE_SKIP_SHAPE_UPDATE_ON_WIRE`被`ExperimentalRedstoneWireEvaluator`用于指示是否应跳过形状更新。仅当信号强度不是放置或原始信号源不是当前电线时才设置。
- `Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS`阻止调用`BlockEntity#preRemoveSideEffects`。这通常阻止方块实体清空其内容。
- `Block.UPDATE_SKIP_ON_PLACE`阻止调用`Block#onPlace`。这通常阻止任何方块处理其初始行为（例如更新铁轨以附着到其他方块，生成铁傀儡）。
- `Block.UPDATE_NONE`是`Block.UPDATE_INVISIBLE | Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS`的别名。
- `Block.UPDATE_ALL`是`Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS`的别名。
- `Block.UPDATE_ALL_IMMEDIATE`是`Block.UPDATE_NEIGHBORS | Block.UPDATE_CLIENTS | Block.UPDATE_IMMEDIATE`的别名。
- `Block.UPDATE_SKIP_ALL_SIDEEFFECTS`是`Block.UPDATE_SKIP_ON_PLACE | Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS | Block.UPDATE_SUPPRESS_DROPS | Block.UPDATE_KNOWN_SHAPE`的别名。

还有一个便捷方法`Level#setBlockAndUpdate(BlockPos pos, BlockState state)`，它内部调用`setBlock(pos, state, Block.UPDATE_ALL)`。

[block]: index.md
[blockentity]: ../blockentities/index.md