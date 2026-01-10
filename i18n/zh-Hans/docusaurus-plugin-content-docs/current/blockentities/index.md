# 方块实体(Block Entities)

方块实体允许在[方块状态][blockstate]不适用的情况下在[方块][block]上存储数据。这对于具有非有限数量选项的数据尤其如此，例如物品栏。方块实体是固定的并绑定到方块，但在其他方面与[实体][entities]有许多相似之处，因此得名。

:::note
如果您的方块具有有限且数量合理（最多几百个）的可能状态，您可能需要考虑使用[方块状态][blockstate]。
:::

## 创建和注册方块实体

与实体类似而与方块不同，`BlockEntity`类表示方块实体实例，而不是[已注册][registration]的单例对象。单例通过`BlockEntityType<?>`类表示。我们需要两者来创建一个新的方块实体。

让我们从创建方块实体类开始：

```
public class MyBlockEntity extends BlockEntity {
    public MyBlockEntity(BlockPos pos, BlockState state) {
        super(type, pos, state);
    }
}
```

您可能已经注意到，我们向父构造函数传递了一个未定义的变量`type`。让我们暂时保留这个未定义的变量，转而进行注册。

[注册][registration]的过程与实体类似。我们创建关联的单例类`BlockEntityType<?>`的实例，并将其注册到方块实体类型注册表中，如下所示：

```
public static final DeferredRegister<BlockEntityType<?>> BLOCK_ENTITY_TYPES =
        DeferredRegister.create(Registries.BLOCK_ENTITY_TYPE, ExampleMod.MOD_ID);

public static final Supplier<BlockEntityType<MyBlockEntity>> MY_BLOCK_ENTITY = BLOCK_ENTITY_TYPES.register(
        "my_block_entity",
        // The block entity type.
        () -> new BlockEntityType<>(
                // The supplier to use for constructing the block entity instances.
                MyBlockEntity::new,
                // An optional value that, when true, only allows players with OP permissions
                // to load NBT data (e.g. placing a block item)
                false,
                // A vararg of blocks that can have this block entity.
                // This assumes the existence of the referenced blocks as DeferredBlock<Block>s.
                MyBlocks.MY_BLOCK_1.get(), MyBlocks.MY_BLOCK_2.get()
        )
);
```

:::note
请记住，`DeferredRegister`必须注册到[模组事件总线][modbus]上！
:::

现在我们有了方块实体类型，我们可以用它来替换我们之前留下的`type`变量：

```
public class MyBlockEntity extends BlockEntity {
    public MyBlockEntity(BlockPos pos, BlockState state) {
        super(MY_BLOCK_ENTITY.get(), pos, state);
    }
}
```

:::info
这种令人困惑的设置过程的原因是，`BlockEntityType`期望一个`BlockEntityType.BlockEntitySupplier<T extends BlockEntity>`，这基本上是一个`BiFunction<BlockPos, BlockState, T extends BlockEntity>`。因此，拥有一个我们可以直接使用`::new`引用的构造函数是非常有益的。但是，我们还需要将构造的方块实体类型提供给`BlockEntity`的默认且唯一的构造函数，所以我们需要稍微传递一下引用。
:::

最后，我们需要修改与方块实体关联的方块类。这意味着我们将无法将方块实体附加到简单的`Block`实例，而是需要一个子类：

```
// The important part is implementing the EntityBlock interface and overriding the #newBlockEntity method.
public class MyEntityBlock extends Block implements EntityBlock {
    // Constructor deferring to super.
    public MyEntityBlock(BlockBehaviour.Properties properties) {
        super(properties);
    }

    // Return a new instance of our block entity here.
    @Override
    public BlockEntity newBlockEntity(BlockPos pos, BlockState state) {
        return new MyBlockEntity(pos, state);
    }
}
```

然后，您当然需要在[方块注册][blockreg]中使用此类作为类型：

```
public static final DeferredBlock<MyEntityBlock> MY_BLOCK_1 =
        BLOCKS.register("my_block_1", () -> new MyEntityBlock( /* ... */ ));
public static final DeferredBlock<MyEntityBlock> MY_BLOCK_2 =
        BLOCKS.register("my_block_2", () -> new MyEntityBlock( /* ... */ ));
```

## 存储数据

`BlockEntity`的主要目的之一是存储数据。方块实体上的数据存储可以通过两种方式发生：读取和写入[值I/O][valueio]，或使用[数据附件][dataattachments]。本节将介绍读写值I/O；有关数据附件，请参阅链接文章。

:::info
数据附件的主要目的，正如其名称所示，是将数据附加到现有的方块实体，例如原版或其他模组提供的方块实体。对于您自己的模组的方块实体，直接保存和加载到值I/O是首选。
:::

可以使用`#loadAdditional`和`#saveAdditional`方法分别从[值I/O][valueio]读取数据并写入数据。当方块实体同步到磁盘或通过网络同步时，会调用这些方法。

```
public class MyBlockEntity extends BlockEntity {
    // This can be any value of any type you want, so long as you can somehow serialize it to the value I/O.
    // We will use an int for the sake of example.
    private int value;

    public MyBlockEntity(BlockPos pos, BlockState state) {
        super(MY_BLOCK_ENTITY.get(), pos, state);
    }

    // Read values from the passed ValueInput here.
    @Override
    public void loadAdditional(ValueInput input) {
        super.loadAdditional(input);
        // Will default to 0 if absent. See the ValueIO article for more information.
        this.value = input.getIntOr("value", 0);
    }

    // Save values into the passed ValueOutput here.
    @Override
    public void saveAdditional(ValueOutput output) {
        super.saveAdditional(output);
        output.putInt("value", this.value);
    }
}
```

在这两个方法中，调用父类方法很重要，因为它添加了基本信息，如位置。标签名称`id`、`x`、`y`、`z`、`NeoForgeData`和`neoforge:attachments`由父类方法保留，因此您不应自己使用它们。

当然，您会想要设置其他值，而不仅仅是使用默认值。您可以自由地这样做，就像任何其他字段一样。但是，如果您希望游戏保存这些更改，则必须随后调用`#setChanged()`，该方法将方块实体的区块标记为脏（=需要保存）。如果您不调用该方法，方块实体可能在保存期间被跳过，因为Minecraft的保存系统只保存被标记为脏的区块。

### 移除方块实体

有时，您可能希望方块实体在移除时导出其存储的数据（例如，当被玩家破坏时掉落其物品栏）。在这些情况下，逻辑应在`BlockEntity#preRemoveSideEffects`内处理。默认情况下，如果您的方块实体掉落实现了[`Container`][container]，则方块实体将掉落其存储的内容。

```
public class MyBlockEntity extends BlockEntity {

    @Override
    public void preRemoveSideEffects(BlockPos pos, BlockState state) {
        super.preRemoveSideEffects(pos, state);
        // Perform any remaining export logic on removal here.
    }
}
```

:::warning
使用设置了`Block.UPDATE_SKIP_BLOCK_ENTITY_SIDEEFFECTS`标志移除的方块不会调用此方法。这通常发生在使用克隆命令或以严格模式放置结构时。
:::

如果相邻方块需要知道方块实体被破坏（例如，通过比较器输出红石信号的物品栏），那么您的方块应该重写`BlockBehaviour#affectNeighborsAfterRemoval`。输出红石信号的方块实体通常在这里调用`Containers#updateNeighboursAfterDestroy`。

```
public class MyEntityBlock extends Block implements EntityBlock {

    @Override
    protected void affectNeighborsAfterRemoval(BlockState state, ServerLevel level, BlockPos pos, boolean movedByPiston) {
        // Handle whatever logic you want to execute on the surrounding neighbors
        Containers.updateNeighboursAfterDestroy(state, level, pos);
    }
}
```

## 刻处理器(Tickers)

方块实体的另一个非常常见的用途，通常与一些存储的数据结合，是刻处理。刻处理意味着每个游戏刻执行一些代码。这是通过重写`EntityBlock#getTicker`并返回一个`BlockEntityTicker`来完成的，`BlockEntityTicker`基本上是一个具有四个参数（level、position、blockstate和block entity）的消费者，如下所示：

```
// Note: The ticker is defined in the block, not the block entity. However, it is good practice to
// keep the ticking logic in the block entity in some way, for example by defining a static #tick method.
public class MyEntityBlock extends Block implements EntityBlock {
    // other stuff here

    @SuppressWarnings("unchecked") // Due to generics, an unchecked cast is necessary here.
    @Override
    public <T extends BlockEntity> BlockEntityTicker<T> getTicker(Level level, BlockState state, BlockEntityType<T> type) {
        // You can return different tickers here, depending on whatever factors you want. A common use case would be
        // to return different tickers on the client or server, only tick one side to begin with,
        // or only return a ticker for some blockstates (e.g. when using a "my machine is working" blockstate property).
        return type == MY_BLOCK_ENTITY.get() ? (BlockEntityTicker<T>) MyBlockEntity::tick : null;
    }
}

public class MyBlockEntity extends BlockEntity {
    // other stuff here

    // The signature of this method matches the signature of the BlockEntityTicker functional interface.
    public static void tick(Level level, BlockPos pos, BlockState state, MyBlockEntity blockEntity) {
        // Whatever you want to do during ticking.
        // For example, you could change a crafting progress value or consume power here.
    }
}
```

请注意，`#tick`方法实际上是每个刻调用的。因此，如果可能，您应该避免在这里进行大量复杂的计算，例如每X刻计算一次，或缓存结果。

## 同步

方块实体逻辑通常在服务器上运行。因此，我们需要告诉客户端我们在做什么。有三种方法可以做到这一点：区块加载时、方块更新时或使用自定义数据包。通常，您应该仅在必要时同步信息，以免不必要地堵塞网络。

### 在区块加载时同步

每次从网络或磁盘读取区块时，都会加载区块（并因此使用此方法）。要在此处发送数据，您需要重写以下方法：

```
public class MyBlockEntity extends BlockEntity {
    // ...

    // Create an update tag here. For block entities with only a few fields, this can just call #saveWithoutMetadata.
    @Override
    public CompoundTag getUpdateTag(HolderLookup.Provider registries) {
        return this.saveWithoutMetadata(registries);
    }

    // Handle a received update tag here. The default implementation calls #loadWithComponents here,
    // so you do not need to override this method if you don't plan to do anything beyond that.
    @Override
    public void handleUpdateTag(ValueInput input) {
        super.handleUpdateTag(input);
    }
}
```

### 在方块更新时同步

每当方块更新发生时，就会使用此方法。方块更新必须手动触发，但通常比区块同步处理得更快。

```
public class MyBlockEntity extends BlockEntity {
    // ...

    // Create an update tag here, like above.
    @Override
    public CompoundTag getUpdateTag(HolderLookup.Provider registries) {
        return this.saveWithoutMetadata(registries);
    }

    // Return our packet here. This method returning a non-null result tells the game to use this packet for syncing.
    @Override
    public Packet<ClientGamePacketListener> getUpdatePacket() {
        // The packet uses the CompoundTag returned by #getUpdateTag. An alternative overload of #create exists
        // that allows you to specify a custom update tag, including the ability to omit data the client might not need.
        return ClientboundBlockEntityDataPacket.create(this);
    }

    // Optionally: Run some custom logic when the packet is received.
    // The super/default implementation forwards to #loadWithComponents.
    @Override
    public void onDataPacket(Connection connection, ValueInput input) {
        super.onDataPacket(connection, input);
        // Do whatever you need to do here.
    }
}
```

要实际发送数据包，必须在服务器上通过调用`Level#sendBlockUpdated(BlockPos pos, BlockState oldState, BlockState newState, int flags)`触发更新通知。位置应该是方块实体的位置，可通过`BlockEntity#getBlockPos`获得。两个方块状态参数都可以是方块实体位置处的方块状态，可通过`BlockEntity#getBlockState`获得。最后，`flags`参数是更新掩码，如在[`Level#setBlock`][setblock]中使用的。

### 使用自定义数据包

通过使用专用的更新数据包，您可以在需要时自行发送数据包。这是最灵活但也是最复杂的变体，因为它需要设置网络处理器。您可以使用`PacketDistrubtor#sendToPlayersTrackingChunk`将数据包发送给跟踪方块实体的所有玩家。有关更多信息，请参阅[网络][networking]部分。

:::caution
进行安全检查很重要，因为当消息到达玩家时，`BlockEntity`可能已经被销毁/替换。您还应该通过`Level#hasChunkAt`检查区块是否已加载。
:::

[block]: ../blocks/index.md
[blockreg]: ../blocks/index.md#basic-blocks
[blockstate]: ../blocks/states.md
[container]: ../inventories/container.md
[dataattachments]: ../datastorage/attachments.md
[entities]: ../entities/index.md
[modbus]: ../concepts/events.md#event-buses
[networking]: ../networking/index.md
[registration]: ../concepts/registries.md#methods-for-registering
[setblock]: ../blocks/states.md#levelsetblock
[valueio]: ../datastorage/valueio.md