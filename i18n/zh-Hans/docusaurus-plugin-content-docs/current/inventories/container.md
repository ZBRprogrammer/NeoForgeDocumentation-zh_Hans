---
sidebar_position: 0
---
# 容器（Containers）

[方块实体][blockentity] 的一个常见用途是存储某种物品。Minecraft 中一些最基本的[方块][block]，如熔炉或箱子，就为此使用了方块实体。为了在某个东西上存储物品，Minecraft 使用 `Container`（容器）。

`Container` 接口定义了诸如 `#getItem`、`#setItem` 和 `#removeItem` 之类的方法，可用于查询和更新容器。由于它是一个接口，它实际上不包含后备列表或其他数据结构，这取决于实现系统。

因此，`Container` 不仅可以在方块实体上实现，也可以在任何其他类上实现。值得注意的例子包括实体物品栏，以及常见的模组化[物品][item]，如背包。

:::warning
NeoForge 提供了 `ItemStacksResourceHandler` 类，作为在许多地方对 `Container` 的替代。应尽可能使用它来代替 `Container`，因为它允许与其他 `Container`/`ItemStacksResourceHandler` 进行更清晰的交互。

本文存在的主要原因是为了在原生代码中作为参考，或者如果你在多个加载器上开发模组。如果可能，请始终在你自己的代码中使用 `ItemStacksResourceHandler`！相关文档正在编写中。
:::

## 基本容器实现

容器可以以任何你喜欢的方式实现，只要满足规定的方法（就像 Java 中的任何其他接口一样）。然而，通常使用具有固定长度的 `NonNullList<ItemStack>` 作为后备结构。单槽容器也可能只使用一个 `ItemStack` 字段。

例如，一个大小为 27 个槽位（一个箱子）的 `Container` 的基本实现可能如下所示：

``` java
public class MyContainer implements Container {
    private final NonNullList<ItemStack> items = NonNullList.withSize(
            // 列表的大小，即我们容器中的槽位数量。
            27,
            // 默认值，用于替代普通列表中你会使用 null 的地方。
            ItemStack.EMPTY
    );

    // 我们容器中的槽位数量。
    @Override
    public int getContainerSize() {
        return 27;
    }

    // 容器是否被视为空。
    @Override
    public boolean isEmpty() {
        return this.items.stream().allMatch(ItemStack::isEmpty);
    }

    // 返回指定槽位中的物品堆。
    @Override
    public ItemStack getItem(int slot) {
        return this.items.get(slot);
    }

    // 当容器发生变化时调用此方法，即添加、修改或移除物品堆时。
    // 例如，你可以在这里调用 BlockEntity#setChanged。
    @Override
    public void setChanged() {

    }

    // 从给定槽位移除指定数量的物品，返回刚刚移除的物品堆。
    // 我们在这里委托给 ContainerHelper，它按预期为我们执行此操作。
    // 但是，我们必须手动调用 #setChanged。
    @Override
    public ItemStack removeItem(int slot, int amount) {
        ItemStack stack = ContainerHelper.removeItem(this.items, slot, amount);
        this.setChanged();
        return stack;
    }

    // 从指定槽位移除所有物品，返回刚刚移除的物品堆。
    // 我们再次委托给 ContainerHelper，并且我们必须再次手动调用 #setChanged。
    @Override
    public ItemStack removeItemNoUpdate(int slot) {
        ItemStack stack = ContainerHelper.takeItem(this.items, slot);
        this.setChanged();
        return stack;
    }

    // 在给定槽位设置给定的物品堆。首先限制到容器的最大堆叠大小。
    @Override
    public void setItem(int slot, ItemStack stack) {
        stack.limitSize(this.getMaxStackSize(stack));
        this.items.set(slot, stack);
        this.setChanged();
    }

    // 容器是否对给定玩家仍然“有效”。例如，箱子和
    // 类似方块在这里检查玩家是否仍在方块的一定距离内。
    @Override
    public boolean stillValid(Player player) {
        return true;
    }

    // 清除内部存储，将所有槽位再次设置为空。
    @Override
    public void clearContent() {
        items.clear();
        this.setChanged();
    }
}
```

### `SimpleContainer`

`SimpleContainer` 类是一个带有一些额外功能的基本容器实现，例如添加 `ContainerListener` 的能力。如果你需要一个没有特殊要求的容器实现，可以使用它。

### `BaseContainerBlockEntity`

`BaseContainerBlockEntity` 类是 Minecraft 中许多重要方块实体的基类，例如箱子和类似箱子的方块、各种熔炉类型、漏斗、发射器、投掷器和酿造台等。

除了 `Container` 之外，它还实现了 `MenuProvider` 和 `Nameable` 接口：

- `Nameable` 定义了与设置（自定义）名称相关的几个方法，并且除了许多方块实体外，还被诸如 `Entity` 之类的类实现。这使用了 [`Component` 系统][component]。
- 另一方面，`MenuProvider` 定义了 `#createMenu` 方法，该方法允许从容器构造一个 [`AbstractContainerMenu`][menu]。这意味着，如果你想要一个没有关联 GUI 的容器（例如点唱机），则不希望使用此类。

`BaseContainerBlockEntity` 将通过两个方法 `#getItems` 和 `#setItems` 捆绑我们通常对 `NonNullList<ItemStack>` 进行的所有调用，从而大大减少了我们需要编写的样板代码。`BaseContainerBlockEntity` 的示例实现可能如下所示：

``` java
public class MyBlockEntity extends BaseContainerBlockEntity {
    // 容器大小。这当然可以是任何你想要的值。
    public static final int SIZE = 9;
    // 我们的物品堆列表。由于存在 #setItems，这不是 final 的。
    private NonNullList<ItemStack> items = NonNullList.withSize(SIZE, ItemStack.EMPTY);

    // 构造函数，和之前一样。
    public MyBlockEntity(BlockPos pos, BlockState blockState) {
        super(MY_BLOCK_ENTITY.get(), pos, blockState);
    }

    // 容器大小，和之前一样。
    @Override
    public int getContainerSize() {
        return SIZE;
    }

    // 我们物品堆列表的 getter。
    @Override
    protected NonNullList<ItemStack> getItems() {
        return items;
    }

    // 我们物品堆列表的 setter。
    @Override
    protected void setItems(NonNullList<ItemStack> items) {
        this.items = items;
    }

    // 菜单的显示名称。别忘了添加翻译！
    @Override
    protected Component getDefaultName() {
        return Component.translatable("container.examplemod.myblockentity");
    }

    // 从此容器创建的菜单。有关在此处返回什么，请参见下文。
    @Override
    protected AbstractContainerMenu createMenu(int containerId, Inventory inventory) {
        return null;
    }
}
```

请记住，此类同时是 `BlockEntity` 和 `Container`。这意味着你可以使用此类作为你的方块实体的超类，以获得一个具有预实现容器的功能方块实体。

:::note
实现 `Container` 的 `BlockEntity` 默认处理掉落其内容。如果你选择不实现 `Container`，那么你将需要处理[移除逻辑][beremove]。
:::

### `WorldlyContainer`

`WorldlyContainer` 是 `Container` 的子接口，允许按 `Direction` 访问给定 `Container` 的槽位。它主要用于仅将部分容器暴露给特定侧面的方块实体。例如，这可以用于一个从一侧输出并从所有其他侧面输入的机器，反之亦然。该接口的简单实现可能如下所示：

``` java
// 参见上面的 BaseContainerBlockEntity 方法。如果需要，你当然可以直接扩展 BlockEntity
// 并自己实现 Container。
public class MyBlockEntity extends BaseContainerBlockEntity implements WorldlyContainer {
    // 其他内容在此处
    
    // 假设槽位 0 是我们的输出，槽位 1-8 是我们的输入。
    // 进一步假设我们向顶部输出并从所有其他侧面输入。
    private static final int[] OUTPUTS = new int[]{0};
    private static final int[] INPUTS = new int[]{1, 2, 3, 4, 5, 6, 7, 8};

    // 根据传入的 Direction 返回暴露的槽位索引数组。
    @Override
    public int[] getSlotsForFace(Direction side) {
        return side == Direction.UP ? OUTPUTS : INPUTS;
    }

    // 是否可以通过给定侧面在给定槽位放置物品。
    // 对于我们的示例，仅当我们不从上方输入且索引在范围 [1, 8] 内时才返回 true。
    @Override
    public boolean canPlaceItemThroughFace(int index, ItemStack itemStack, @Nullable Direction direction) {
        return direction != Direction.UP && index > 0 && index < 9;
    }

    // 是否可以从给定侧面和给定槽位取出物品。
    // 对于我们的示例，仅当我们从上方拉取且来自槽位索引 0 时才返回 true。
    @Override
    public boolean canTakeItemThroughFace(int index, ItemStack stack, Direction direction) {
        return direction == Direction.UP && index == 0;
    }
}
```

## 使用容器

现在我们已经创建了容器，让我们来使用它们！

由于 `Container` 和 `BlockEntity` 之间有相当大的重叠，如果可能，最好通过将方块实体强制转换为 `Container` 来检索容器：

``` java
if (blockEntity instanceof Container container) {
    // 对容器执行某些操作
}
```

然后，容器可以使用我们之前提到的方法，例如：

``` java
// 获取容器中的第一个物品。
ItemStack stack = container.getItem(0);

// 将容器中的第一个物品设置为泥土。
container.setItem(0, new ItemStack(Items.DIRT));

// 从第三个槽位移除最多 16 个物品。
container.removeItem(2, 16);
```

:::warning
如果尝试访问超出其容器大小的槽位，容器可能会抛出异常。或者，它们可能返回 `ItemStack.EMPTY`，就像（例如）`SimpleContainer` 的情况一样。
:::

### `ContainerUser`

能够访问容器的活体实体实现 `ContainerUser`。每个用户定义其是否打开了一个容器，以及实体可以与容器交互的最大方块距离。当容器对象被交互时（例如，右键单击箱子），`Container` 会使用 `ContainerUser` 调用 `startOpen`，并在容器对象关闭时（例如，离开箱子菜单）调用 `stopOpen`。

这些方法通常用于通过 `ContainerOpenersCounter` 跟踪打开容器的活体实体数量，该计数器用于一些实体 AI 和渲染。

## `ItemStack` 上的 `Container`

到目前为止，我们主要讨论了 `BlockEntity` 上的 `Container`。但是，它们也可以使用 `minecraft:container` [数据组件][datacomponent] 应用于 [`ItemStack`][itemstack]：

``` java
// 我们在这里使用 SimpleContainer 作为超类，这样我们就不必自己重新实现物品处理逻辑。
// 由于 SimpleContainer 的实现细节，如果多方可以同时访问容器，这可能会导致竞争条件，
// 所以我们假设我们的模组不允许这样做。
// 如果需要，你当然可以使用不同的 Container 实现（或自己实现 Container）。
public class MyBackpackContainer extends SimpleContainer {
    // 此容器所属的物品堆。在构造函数中传入并设置。
    private final ItemStack stack;
    
    public MyBackpackContainer(ItemStack stack) {
        // 我们使用期望的容器大小调用超类。
        super(27);
        // 设置 stack 字段。
        this.stack = stack;
        // 我们从数据组件（如果存在）加载容器内容，该组件由
        // ItemContainerContents 类表示。如果不存在，我们使用 ItemContainerContents.EMPTY。
        ItemContainerContents contents = stack.getOrDefault(DataComponents.CONTAINER, ItemContainerContents.EMPTY);
        // 将数据组件内容复制到我们的物品堆列表中。
        contents.copyInto(this.getItems());
    }

    // 当内容更改时，我们将数据组件保存在物品堆上。
    @Override
    public void setChanged() {
        super.setChanged();
        this.stack.set(DataComponents.CONTAINER, ItemContainerContents.fromItems(this.getItems()));
    }
}
```

看，你已经创建了一个基于物品的容器！调用 `new MyBackpackContainer(stack)` 来为菜单或其他用例创建容器。

:::warning
请注意，直接与 `Container` 交互的菜单在修改其 `ItemStack` 时必须 `#copy()` 它们，否则会破坏数据组件上的不可变性契约。为此，NeoForge 为你提供了 `StackCopySlot` 类。
:::

## `Entity` 上的 `Container`

[`Entity`][entity] 上的 `Container` 很棘手：实体是否具有容器无法普遍确定。这完全取决于你正在处理什么实体，因此可能需要大量特殊情况处理。

如果你正在自己创建一个实体，没有什么能阻止你直接在其上实现 `Container`，但请注意，你将无法使用诸如 `SimpleContainer` 之类的超类（因为 `Entity` 是超类）。

### `Mob` 上的 `Container`

`Mob` 不实现 `Container`，但它们实现了 `EquipmentUser` 接口（以及其他）。该接口定义了方法 `#setItemSlot(EquipmentSlot, ItemStack)`、`#getItemBySlot(EquipmentSlot)` 和 `#setDropChance(EquipmentSlot, float)`。虽然代码方面与 `Container` 无关，但功能相当相似：我们将槽位（在这种情况下是装备槽位）与 `ItemStack` 关联起来。

与 `Container` 最显著的区别是没有类似列表的顺序（尽管 `Mob` 在后台使用 `NonNullList<ItemStack>`）。访问不是通过槽位索引进行的，而是通过七个 `EquipmentSlot` 枚举值：`MAINHAND`、`OFFHAND`、`FEET`、`LEGS`、`CHEST`、`HEAD` 和 `BODY`（其中 `BODY` 用于马和狗盔甲）。

与生物的“槽位”交互的示例如下：

``` java
// 获取 HEAD（头盔）槽位中的物品堆。
ItemStack helmet = mob.getItemBySlot(EquipmentSlot.HEAD);

// 将基岩放入生物的 FEET（靴子）槽位。
mob.setItemSlot(EquipmentSlot.FEET, new ItemStack(Items.BEDROCK));

// 启用该基岩在生物被杀死时总是掉落。
mob.setDropChance(EquipmentSlot.FEET, 1f);
```

### `InventoryCarrier`

`InventoryCarrier` 是一些活体实体（如村民）实现的接口。它声明了一个方法 `#getInventory`，返回一个 `SimpleContainer`。此接口用于需要实际物品栏而不仅仅是 `EquipmentUser` 提供的装备槽位的非玩家实体。

### `Player` 上的 `Container`（玩家物品栏）

玩家的物品栏是通过 `Inventory` 类实现的，该类实现了 `Container` 以及前面提到的 `Nameable` 接口。该 `Inventory` 的实例然后作为名为 `inventory` 的字段存储在 `Player` 上，可通过 `Player#getInventory` 访问。可以像任何其他容器一样与物品栏交互。

物品栏内容存储在两个位置：

- `NonNullList<ItemStack> items` 列表覆盖 36 个主物品栏槽位，包括九个快捷栏槽位（索引 0-8）。
- `EntityEquipment equipment` 映射存储 `EquipmentSlot` 堆：盔甲槽位（`FEET`、`LEGS`、`CHEST`、`HEAD`）、`OFFHAND`、`BODY` 和 `SADDLE`，按此顺序。

当遍历物品栏内容时，建议先遍历 `items`，然后使用 `Inventory#EQUIPMENT_SLOT_MAPPING` 遍历 `equipment` 的索引。

[beremove]: ../blockentities/index.md#removing-block-entities
[block]: ../blocks/index.md
[blockentity]: ../blockentities/index.md
[component]: ../resources/client/i18n.md#components
[datacomponent]: ../items/datacomponents.md
[entity]: ../entities/index.md
[item]: ../items/index.md
[itemstack]: ../items/index.md#itemstacks
[menu]: menus.md