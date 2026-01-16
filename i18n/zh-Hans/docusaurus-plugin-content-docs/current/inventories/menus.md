---
sidebar_position: 2
---
# 菜单（Menus）

菜单是图形用户界面（GUI）的一种后端类型；它们处理与某些表示的数据持有者交互所涉及的逻辑。菜单本身不是数据持有者。它们是允许用户间接修改内部数据持有者状态的视图。因此，数据持有者不应直接耦合到任何菜单，而是传入要调用和修改的数据引用。

## `MenuType`

菜单是动态创建和移除的，因此不是注册表对象。因此，另一个工厂对象被注册，以便轻松创建和引用菜单的*类型*。对于菜单，这些是 `MenuType`。

`MenuType` 必须被[注册][registered]。

### `MenuSupplier`

通过将 `MenuSupplier` 和 `FeatureFlagSet` 传递给其构造函数来创建 `MenuType`。`MenuSupplier` 表示一个函数，该函数接受容器的 ID 和查看菜单的玩家的物品栏，并返回一个新创建的 [`AbstractContainerMenu`][acm]。

``` java
// 对于某个 DeferredRegister<MenuType<?>> REGISTER
public static final Supplier<MenuType<MyMenu>> MY_MENU = REGISTER.register("my_menu", () -> new MenuType<>(MyMenu::new, FeatureFlags.DEFAULT_FLAGS));

// 在 MyMenu 中，一个 AbstractContainerMenu 子类
public MyMenu(int containerId, Inventory playerInv) {
    super(MY_MENU.get(), containerId);
    // ...
}
```

:::note
容器标识符对于单个玩家是唯一的。这意味着，两个不同玩家上的相同容器 ID 将代表两个不同的菜单，即使它们查看的是相同的数据持有者。
:::

`MenuSupplier` 通常负责在客户端上创建一个菜单，其中使用虚拟数据引用来存储和与来自服务器数据持有者的同步信息进行交互。

### `IContainerFactory`

如果客户端需要额外信息（例如世界中数据持有者的位置），则可以使用子类 `IContainerFactory`。除了容器 ID 和玩家物品栏外，它还提供一个 `RegistryFriendlyByteBuf`，可以存储从服务器发送的额外信息。可以使用 `IContainerFactory` 通过 `IMenuTypeExtension#create` 创建 `MenuType`。

``` java
// 对于某个 DeferredRegister<MenuType<?>> REGISTER
public static final Supplier<MenuType<MyMenuExtra>> MY_MENU_EXTRA = REGISTER.register("my_menu_extra", () -> IMenuTypeExtension.create(MyMenu::new));

// 在 MyMenuExtra 中，一个 AbstractContainerMenu 子类
public MyMenuExtra(int containerId, Inventory playerInv, FriendlyByteBuf extraData) {
    super(MY_MENU_EXTRA.get(), containerId);
    // 从缓冲区存储额外数据
    // ...
}
```

## `AbstractContainerMenu`

所有菜单都从 `AbstractContainerMenu` 扩展而来。菜单接受两个参数：[`MenuType`][mt]（表示菜单本身的类型）和容器 ID（表示当前访问者的菜单的唯一标识符）。

:::note
菜单标识符在 0-99 之间循环，每当玩家打开一个菜单时递增。
:::

每个菜单应包含两个构造函数：一个用于在服务器上初始化菜单，另一个用于在客户端上初始化菜单。用于在客户端上初始化菜单的构造函数是提供给 `MenuType` 的那个。服务器菜单构造函数包含的任何字段都应在客户端菜单构造函数中具有一些默认值。

``` java
// 客户端菜单构造函数
public MyMenu(int containerId, Inventory playerInventory) { // 如果需要从服务器读取数据，则可选 FriendlyByteBuf 参数
    this(containerId, playerInventory, /* 此处为任何默认参数 */);
}

// 服务器菜单构造函数
public MyMenu(int containerId, Inventory playerInventory, /* 此处为任何额外参数。 */) {
    // ...
}
```

:::note
如果菜单中不需要显示额外数据，则只需要一个构造函数。
:::

每个菜单实现必须实现两个方法：`#stillValid` 和 [`#quickMoveStack`][qms]。

### `#stillValid` 和 `ContainerLevelAccess`

`#stillValid` 确定菜单是否应保持对给定玩家打开。这通常指向静态的 `#stillValid`，它接受一个 `ContainerLevelAccess`、玩家以及此菜单附加到的 `Block`。客户端菜单必须始终为此方法返回 `true`，静态的 `#stillValid` 默认会这样做。此实现检查玩家是否在数据存储对象所在位置的八个方块范围内。

`ContainerLevelAccess` 在封闭范围内提供当前世界和方块位置。在服务器上构造菜单时，可以通过调用 `ContainerLevelAccess#create` 创建新的访问。客户端菜单构造函数可以传入 `ContainerLevelAccess#NULL`，这将不执行任何操作。

``` java
// 客户端菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory) {
    this(containerId, playerInventory, ContainerLevelAccess.NULL);
}

// 服务器菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory, ContainerLevelAccess access) {
    // ...
}

// 假设此菜单附加到 Supplier<Block> MY_BLOCK
@Override
public boolean stillValid(Player player) {
    return AbstractContainerMenu.stillValid(this.access, player, MY_BLOCK.get());
}
```

### 数据同步

某些数据需要同时存在于服务器和客户端上以显示给玩家。为此，菜单实现了基本的数据同步层，以便每当当前数据与上次同步到客户端的数据不匹配时进行同步。对于玩家，这是每个游戏刻检查的。

Minecraft 默认支持两种形式的数据同步：通过 `Slot` 的 [`ItemStack`][itemstack] 和通过 `DataSlot` 的整数。`Slot` 和 `DataSlot` 是视图，它们持有对数据存储的引用，这些存储可以由玩家在屏幕中修改，假设操作有效。每种数据同步方法都会向服务器菜单构造函数添加一个参数，而客户端将创建一个虚拟实例，用于写入从服务器发送的数据。

#### `DataSlot`

`DataSlot` 是一个抽象类，应实现一个 getter 和 setter 以引用存储在数据存储对象中的数据。客户端菜单构造函数应始终通过 `DataSlot#standalone` 提供新实例。然后可以使用 `#addDataSlot` 将数据槽添加到菜单中。

这些，连同槽位，应在每次初始化新菜单时重新创建。

:::note
尽管 `DataSlot` 存储一个整数，但由于它通过网络发送值的方式，它实际上被限制为**短整型**（-32768 到 32767）。整数的高 16 位被忽略。

NeoForge 修补了数据包以向客户端提供完整的整数。
:::

``` java
// 假设我们在每次初始化服务器菜单时构造了一个 DataSlot

// 客户端菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory) {
    this(
        containerId, playerInventory,
        // 传入一个虚拟槽位以保存服务器同步的值
        DataSlot.standalone()
    );
}

// 服务器菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory, DataSlot dataSingle) {
    // 为处理的整数添加数据槽
    this.addDataSlot(dataSingle);

    // ...
}
```

#### `ContainerData`

如果需要将多个整数同步到客户端，则可以使用 `ContainerData` 来引用整数。此接口充当索引查找，使得每个索引代表一个不同的整数。`ContainerData` 也可以在数据对象本身中构造，如果通过 `#addDataSlots` 将 `ContainerData` 添加到菜单中。该方法为接口指定的数据量创建一个新的 `DataSlot`。客户端菜单构造函数应始终通过 `SimpleContainerData` 提供新实例。

``` java
// 假设我们有一个大小为 3 的 ContainerData

// 客户端菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory) {
    this(containerId, playerInventory, new SimpleContainerData(3));
}

// 服务器菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory, ContainerData dataMultiple) {
    // 检查 ContainerData 大小是否为某个固定值
    checkContainerDataCount(dataMultiple, 3);

    // 为处理的整数添加数据槽
    this.addDataSlots(dataMultiple);

    // ...
}
```

#### `Slot`

`Slot` 表示对物品栏中某个 [`ItemStack`][itemstack] 的引用。每个 `Slot` 至少具有四个参数：物品堆所在的物品栏、此槽位专门表示的堆的索引，以及槽位在屏幕上渲染的左上角位置相对于 `AbstractContainerScreen#leftPos` 和 `#topPos` 的 x 和 y 位置。任何额外参数通常为槽位提供上下文以处理独特行为，例如仅接受被视为燃料的物品或防止物品被取出。

服务器菜单构造函数应接受物品栏实例或视图。同时，客户端菜单构造函数应始终提供一个相同大小的空物品栏实例以写入服务器数据。然后可以使用 `#addSlot` 将所需的槽位或其子类型之一添加到菜单中。

对于 [`Container`][container]，客户端菜单通常会传入一个 `SimpleContainer` 并使用常规 `Slot` 添加。对于 [`ResourceHandler<ItemResource>` 能力][cap]，客户端菜单通常会传入一个 `ItemStacksResourceHandler` 并使用 `ResourceHandlerSlot` 添加。

在大多数情况下，菜单包含的任何槽位首先被添加，然后是玩家的物品栏，最后是玩家的快捷栏。要从菜单访问任何单个 `Slot`，必须根据添加槽位的顺序计算索引。

``` java
// 假设我们有一个来自数据对象的物品栏，大小为 10

// 客户端菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory) {
    this(containerId, playerInventory, new ItemStacksResourceHandler(10));
}

// 服务器菜单构造函数
public MyMenuAccess(int containerId, Inventory playerInventory, StacksResourceHandler<ItemStack, ItemResource> dataInventory) {
    // 检查数据物品栏大小是否为某个固定值
    int dataSize = dataInventory.getSlots();
    if (dataSize < 10) {
        throw new IllegalArgumentException("Container size " + dataSize + " is smaller than expected " + 5);
    }

    // 然后，为数据物品栏添加槽位
    // 如果你使用的是槽位的子类型，请确保服务器上使用的任何数据
    // 在客户端上也可用。

    // 通过循环遍历位置并添加槽位来创建物品栏。

    // 两行
    for (int j = 0; j < 2; j++) {
        // 五列
        for (int i = 0; i < 5; i++) {
            // 为数据物品栏中的每个槽位添加
            this.addSlot(new ResourceHandlerSlot(
                // 物品栏
                dataInventory,
                // 用于修改存储资源的索引修改器
                dataInventory::set,
                // 此槽位表示的数据物品栏索引：
                // 行索引 * 列数 + 列索引
                j * 5 + i,
                // 相对于 leftPos 的 x 位置
                // 原版槽位默认大小为 18 单位
                // startX + 列索引 * 槽位渲染宽度
                44 + i * 18,
                // 相对于 topPos 的 y 位置
                // 原版槽位默认大小为 18 单位
                // startY + 行索引 * 槽位渲染高度
                20 + j * 18
            ))
        }
    }

    // 为玩家物品栏添加槽位（所有 27 + 9 个快捷栏槽位）
    // 如果你想自定义 9x3 + 9 网格为其他形式，
    // 像上面那样循环遍历
    this.addStandardInventorySlots(
        playerInventory,
        // 相对于 leftPos 的起始 x 位置
        8,
        // 相对于 topPos 的起始 y 位置
        84
    );

    // ...
}
```

#### `#quickMoveStack`

`#quickMoveStack` 是任何菜单必须实现的第二个方法。每当物品堆被 Shift-单击或快速移出其当前槽位时，就会调用此方法，直到物品堆完全移出其先前槽位或没有其他地方可去。该方法返回正在被快速移动的槽位中的物品堆副本。

通常使用 `#moveItemStackTo` 在槽位之间移动物品堆，该方法将物品堆移动到第一个可用的槽位。它接受要移动的物品堆、要尝试将物品堆移动到的第一个槽位索引（包括）、最后一个槽位索引（不包括），以及是否从头到尾检查槽位（当为 `false` 时）或从尾到头检查（当为 `true` 时）。

在 Minecraft 的实现中，此方法的逻辑相当一致：

``` java
// 假设我们有一个大小为 5 的数据物品栏
// 该物品栏有 4 个输入（索引 1 - 4）输出到一个结果槽位（索引 0）
// 我们还有 27 个玩家物品栏槽位和 9 个快捷栏槽位
// 因此，实际的槽位索引如下：
//   - 数据物品栏：结果（0），输入（1 - 4）
//   - 玩家物品栏（5 - 31）
//   - 玩家快捷栏（32 - 40）
@Override
public ItemStack quickMoveStack(Player player, int quickMovedSlotIndex) {
    // 被快速移动的槽位堆
    ItemStack quickMovedStack = ItemStack.EMPTY;
    // 被快速移动的槽位
    Slot quickMovedSlot = this.slots.get(quickMovedSlotIndex);
  
    // 如果槽位在有效范围内且槽位不为空
    if (quickMovedSlot != null && quickMovedSlot.hasItem()) {
        // 获取要移动的原始堆
        ItemStack rawStack = quickMovedSlot.getItem(); 
        // 将槽位堆设置为原始堆的副本
        quickMovedStack = rawStack.copy();

        /*
        以下快速移动逻辑可以简化为：如果在数据物品栏中，
        尝试移动到玩家物品栏/快捷栏，反之亦然，适用于
        不能转换数据的容器（例如箱子）。
        */

        // 如果在数据物品栏结果槽位上执行了快速移动
        if (quickMovedSlotIndex == 0) {
            // 尝试将结果槽位移动到玩家物品栏/快捷栏
            if (!this.moveItemStackTo(rawStack, 5, 41, true)) {
                // 如果无法移动，则不再快速移动
                return ItemStack.EMPTY;
            }

            // 在结果槽位快速移动时执行逻辑
            quickMovedSlot.onQuickCraft(rawStack, quickMovedStack);
        }
        // 否则，如果在玩家物品栏或快捷栏槽位上执行了快速移动
        else if (quickMovedSlotIndex >= 5 && quickMovedSlotIndex < 41) {
            // 尝试将物品栏/快捷栏槽位移动到数据物品栏输入槽位
            if (!this.moveItemStackTo(rawStack, 1, 5, false)) {
                // 如果无法移动且在玩家物品栏槽位中，则尝试移动到快捷栏
                if (quickMovedSlotIndex < 32) {
                    if (!this.moveItemStackTo(rawStack, 32, 41, false)) {
                        // 如果无法移动，则不再快速移动
                        return ItemStack.EMPTY;
                    }
                }
                // 否则尝试将快捷栏移动到玩家物品栏槽位
                else if (!this.moveItemStackTo(rawStack, 5, 32, false)) {
                    // 如果无法移动，则不再快速移动
                    return ItemStack.EMPTY;
                }
            }
        }
        // 否则，如果在数据物品栏输入槽位上执行了快速移动，尝试移动到玩家物品栏/快捷栏
        else if (!this.moveItemStackTo(rawStack, 5, 41, false)) {
            // 如果无法移动，则不再快速移动
            return ItemStack.EMPTY;
        }

        if (rawStack.isEmpty()) {
            // 如果原始堆已完全移出槽位，则将槽位设置为空堆
            quickMovedSlot.setByPlayer(ItemStack.EMPTY);
        } else {
            // 否则，通知槽位堆计数已更改
            quickMovedSlot.setChanged();
        }

        /*
        如果菜单不表示可以转换堆的容器（例如箱子），
        则可以删除以下 if 语句和 Slot#onTake 调用。
        */
        if (rawStack.getCount() == quickMovedStack.getCount()) {
            // 如果原始堆无法移动到另一个槽位，则不再快速移动
            return ItemStack.EMPTY;
        }
        // 对移动后剩余的堆执行逻辑
        quickMovedSlot.onTake(player, rawStack);
    }

    return quickMovedStack; // 返回槽位堆
}
```

## 打开菜单

一旦注册了菜单类型，菜单本身已经完成，并且附加了[屏幕][screen]，玩家就可以打开菜单。可以通过在逻辑服务器上调用 `IPlayerExtension#openMenu` 来打开菜单。该方法接受服务器端菜单的 `MenuProvider`，如果需要将额外数据同步到客户端，则可选地接受 `Consumer<RegistryFriendlyByteBuf>`。

:::note
带有 `Consumer<RegistryFriendlyByteBuf>` 参数的 `IPlayerExtension#openMenu` 应仅在使用 [`IContainerFactory`][icf] 创建菜单类型时使用。
:::

#### `MenuProvider`

`MenuProvider` 是一个接口，包含两个方法：`#createMenu`（创建菜单的服务器实例）和 `#getDisplayName`（返回包含菜单标题的组件以传递给[屏幕][screen]）。`#createMenu` 方法包含三个参数：菜单的容器 ID、打开菜单的玩家的物品栏，以及打开菜单的玩家。

可以使用 `SimpleMenuProvider` 轻松创建 `MenuProvider`，它接受创建服务器菜单的方法引用和菜单的标题。

``` java
// 在可以访问逻辑服务器上 Player 的某个实现中（例如 ServerPlayer 实例）
// 假设我们有 ServerPlayer serverPlayer
serverPlayer.openMenu(new SimpleMenuProvider(
    (containerId, playerInventory, player) -> new MyMenu(containerId, playerInventory, /* 服务器参数 */),
    Component.translatable("menu.title.examplemod.mymenu")
));
```

### 常见实现

菜单通常在某种玩家交互时打开（例如，当右键单击方块或实体时）。

#### 方块实现

方块通常通过重写 `BlockBehaviour#useWithoutItem` 来实现菜单，为[交互][interaction]返回 `InteractionResult#SUCCESS`。

`MenuProvider` 应通过重写 `BlockBehaviour#getMenuProvider` 来实现。原生方法使用此方法在旁观者模式下查看菜单。

``` java
// 在某个 Block 子类中
@Override
public MenuProvider getMenuProvider(BlockState state, Level level, BlockPos pos) {
    return new SimpleMenuProvider(/* ... */);
}

@Override
public InteractionResult useWithoutItem(BlockState state, Level level, BlockPos pos, Player player, BlockHitResult result) {
    if (!level.isClientSide() && player instanceof ServerPlayer serverPlayer) {
        serverPlayer.openMenu(state.getMenuProvider(level, pos));
    }

    return InteractionResult.SUCCESS;
}
```

:::note
这是实现逻辑的最简单方式，不是唯一方式。如果你希望方块仅在特定条件下打开菜单，那么需要事先将一些数据同步到客户端，以便在条件不满足时返回 `InteractionResult#PASS` 或 `#FAIL`。
:::

#### 生物实现

生物通常通过重写 `Mob#mobInteract` 来实现菜单。这与方块实现类似，唯一的区别是 `Mob` 本身应实现 `MenuProvider` 以支持旁观者模式查看。

``` java
public class MyMob extends Mob implements MenuProvider {
    // ...

    @Override
    public InteractionResult mobInteract(Player player, InteractionHand hand) {
        if (!this.level.isClientSide() && player instanceof ServerPlayer serverPlayer) {
            serverPlayer.openMenu(this);
        }

        return InteractionResult.SUCCESS;
    }
}
```

:::note
再次强调，这是实现逻辑的最简单方式，不是唯一方式。
:::

[registered]: ../concepts/registries.md#methods-for-registering
[acm]: #abstractcontainermenu
[mt]: #menutype
[qms]: #quickmovestack
[cap]: capabilities.md#neoforge-provided-capabilities
[screen]: ../rendering/screens.md
[icf]: #icontainerfactory
[side]: ../concepts/sides.md#the-logical-side
[interaction]: ../items/interactions.md#right-clicking-an-item
[itemstack]: ../items/index.md#itemstacks