---
sidebar_position: 1
---
# 交互(Interactions)

本文旨在使玩家左键单击、右键单击或中键单击事物的相当复杂和混乱的过程更容易理解，并阐明在何处使用什么结果以及为什么。

## `HitResult`（命中结果）

为了确定玩家当前正在看什么，Minecraft 使用 `HitResult`。`HitResult` 在某种程度上等同于其他游戏引擎中的射线投射结果，最值得注意的是包含一个方法 `#getLocation`。

命中结果可以是三种类型之一，通过 `HitResult.Type` 枚举表示：`BLOCK`、`ENTITY` 或 `MISS`。类型为 `BLOCK` 的 `HitResult` 可以强制转换为 `BlockHitResult`，而类型为 `ENTITY` 的 `HitResult` 可以强制转换为 `EntityHitResult`；两种类型都提供了关于击中了哪个[方块(Block)][block]或[实体(Entity)][entity]的额外上下文。如果类型是 `MISS`，则表示既没有击中方块也没有击中实体，不应强制转换为任何一个子类。

在[物理客户端][physicalside]上的每一帧，`Minecraft` 类更新并存储当前查看的 `HitResult` 在 `hitResult` 字段中。然后可以通过 `Minecraft.getInstance().hitResult` 访问此字段。

## 左键单击物品

- 检查主手中 [`ItemStack`][itemstack] 所需的所有[功能标志][featureflag]是否已启用。如果此检查失败，流程结束。
- 使用左鼠标按钮和主手触发 `InputEvent.InteractionKeyMappingTriggered`。如果[事件][event]被[取消][cancel]，流程结束。
- 根据你正在看的内容（使用 `Minecraft` 中的 [`HitResult`][hitresult]），发生不同的事情：
    - 如果你正在看一个在你触及范围内的[实体]：
        - 触发 `AttackEntityEvent`。如果事件被取消，流程结束。
        - 调用 `IItemExtension#onLeftClickEntity`。如果返回 true，流程结束。
        - 在目标上调用 `Entity#isAttackable`。如果返回 false，流程结束。
        - 在目标上调用 `Entity#skipAttackInteraction`。如果返回 true，流程结束。
        - 如果目标在 `minecraft:redirectable_projectile` 标签中（默认情况下这是火球和风弹）并且是 `Projectile` 的实例，则目标被偏转，流程结束。
        - 实体基础伤害（`minecraft:attack_damage` [属性][attribute]的值）和附魔额外伤害作为两个单独的浮点数计算。如果两者都为0，流程结束。
            - 注意，这排除了主手物品的[属性修饰符][attributemodifier]，这些在检查之后添加。
        - 主手物品的 `minecraft:attack_damage` 属性修饰符被添加到基础伤害中。
        - 触发 `CriticalHitEvent`。如果事件的 `#isCriticalHit` 方法返回 true，则基础伤害乘以事件 `#getDamageMultiplier` 方法返回的值，如果[一系列条件][critical]通过，则默认为1.5，否则为1.0，但可能被事件修改。
        - 附魔额外伤害被添加到基础伤害中，产生最终的伤害值。
        - 触发 `SweepAttackEvent`。如果事件的 `isSweeping` 方法返回 true，则玩家将执行横扫攻击。默认情况下，这会检查攻击冷却是否 > 90%，攻击不是暴击，玩家在地面上且移动速度不超过其 `minecraft:movement_speed` 属性值。
        - 调用 [`Entity#hurtOrSimulate`][hurt]。如果返回 false，流程结束。
        - 如果目标是 `LivingEntity` 的实例且攻击强度大于90%，玩家正在疾跑，并且被附魔修改后的 `minecraft:attack_knockback` 属性值大于0，则调用 `LivingEntity#knockback`。
            - 在该方法中，触发 `LivingKnockBackEvent`。如果事件被取消，则不应用击退。
        - 玩家基于 `SweepAttackEvent#isSweeping` 对附近的 `LivingEntity` 执行横扫攻击。
            - 在该方法中，如果实体在玩家触及范围内且 `Entity#hurtServer` 返回 true，则再次调用 `LivingEntity#knockback`，这又触发了一次 `LivingKnockBackEvent`。
        - 调用 `Item#hurtEnemy`。这可以用于攻击后效果。例如，钉锤（mace）在此处将玩家发射回空中（如果适用）。
        - 调用 `Item#postHurtEnemy`。耐久度伤害在此处应用。
            - 如果耐久度达到零，将堆叠变为 `ItemStack#EMPTY`，则触发 `PlayerDestroyItemEvent`。
    - 如果你正在看一个在你触及范围内的[方块]：
        - 启动[方块破坏子流程][blockbreak]。
    - 否则：
        - 触发 `PlayerInteractEvent.LeftClickEmpty`。

## 右键单击物品

在右键单击流程中，会调用许多返回两种结果类型之一的方法（见下文）。这些方法中的大多数如果返回明确的成功或明确的失败，则取消流程。为了可读性，从现在开始，这种“明确的成功或明确的失败”将被称为“明确的结果”。

- 使用右键鼠标按钮和主手触发 `InputEvent.InteractionKeyMappingTriggered`。如果[事件][event]被[取消][cancel]，流程结束。
- 检查几种情况，例如你不是旁观者模式，或者主手中 [`ItemStack`][itemstack] 所需的所有[功能标志][featureflag]都已启用。如果至少有一个检查失败，流程结束。
- 根据你正在看的内容（使用 `Minecraft` 中的 [`HitResult`][hitresult]），发生不同的事情：
    - 如果你正在看一个在你触及范围内且不在世界边界之外的[实体]：
        - 触发 `PlayerInteractEvent.EntityInteractSpecific`。如果事件被取消，流程结束。
        - **在你正在看的实体上**调用 `Entity#interactAt`。如果它返回一个明确的结果，流程结束。
            - 如果你想为你自己的实体添加行为，请覆盖此方法。如果你想为 Vanilla 实体添加行为，请使用事件。
        - 如果实体打开一个界面（例如村民交易 GUI 或箱子矿车 GUI），流程结束。
        - 触发 `PlayerInteractEvent.EntityInteract`。如果事件被取消，流程结束。
        - **在你正在看的实体上**调用 `Entity#interact`。如果它返回一个明确的结果，流程结束。
            - 如果你想为你自己的实体添加行为，请覆盖此方法。如果你想为 Vanilla 实体添加行为，请使用事件。
            - 对于[`Mob`s][livingentity]，`Entity#interact` 的覆盖处理诸如用主手中的 `ItemStack` 是刷怪蛋时拴绳和生成幼崽，然后将特定于生物的处理委托给 `Mob#mobInteract`。`Entity#interact` 的结果规则在此同样适用。
        - 如果你正在看的实体是 `LivingEntity`，则在主手的 `ItemStack` 上调用 `Item#interactLivingEntity`。如果它返回一个明确的结果，流程结束。
    - 如果你正在看一个在你触及范围内且不在世界边界之外的[方块]：
        - 触发 `PlayerInteractEvent.RightClickBlock`。如果事件被取消，流程结束。你也可以在此事件中专门仅拒绝方块或物品使用。
        - 调用 `IItemExtension#onItemUseFirst`。如果它返回一个明确的结果，流程结束。
        - 如果 `IItemExtension#doesSneakBypassUse` 返回 false 且事件不拒绝方块使用，则触发 `UseItemOnBlockEvent`。如果事件被取消，则使用取消结果。否则，调用 `BlockBehaviour#useItemOn`。如果它返回一个明确的结果，流程结束。
        - 如果 `InteractionResult` 是 `TryEmptyHandInteraction` 的实例（例如 `TRY_WITH_EMPTY_HAND`）且执行手是主手，则调用 `BlockBehaviour#useWithoutItem`。如果它返回一个明确的结果，流程结束。
        - 如果事件不拒绝物品使用，则调用 `Item#useOn`。如果它返回一个明确的结果，流程结束。
     - 否则：
        - 触发 `PlayerInteractEvent.RightClickEmpty`。
- 触发 `PlayerInteractEvent.RightClickItem`。如果事件被取消，流程结束。
- 调用 `Item#use`。
    - 如果 `InteractionResult` 是 `Success` 的实例（例如 `SUCCESS`），则 `ItemStack` 更改为 `Success#heldItemTransformedTo`。
- 如果当前堆叠与原始堆叠不匹配且新堆叠为空，则触发 `PlayerDestroyItemEvent`。
- 上述过程运行第二次，这次使用副手而不是主手。

### `InteractionResult`

`InteractionResult` 是一个密封接口，表示物品或空手与某个对象（例如实体、方块等）之间交互的结果。该接口分为四个记录，其中有六种潜在的默认状态。

首先是 `InteractionResult.Success`，它表示操作应被视为成功，结束流程。成功状态有两个参数：`SwingSource`，表示实体是否应在相应的[逻辑端][side]上挥手；以及 `InteractionResult.ItemContext`，它保存交互是否由持有的物品引起，以及使用后持有的物品变成了什么。挥手源由默认状态之一确定：客户端挥手的 `InteractionResult#SUCCESS`，服务器挥手的 `InteractionResult#SUCCESS_SERVER`，以及不挥手的 `InteractionResult#CONSUME`。如果 `ItemStack` 更改，则通过 `Success#heldItemTransformedTo` 设置物品上下文，或者如果持有的物品与对象之间没有交互，则通过 `withoutItem` 设置。默认设置是有物品交互但没有转换。

``` java
// 在某个返回交互结果的方法中

// 手中的物品将变成苹果
return InteractionResult.SUCCESS.heldItemTransformedTo(new ItemStack(Items.APPLE));
```

:::note
通常永远不应在同一方法中使用 `SUCCESS` 和 `SUCCESS_SERVER`。如果客户端有足够的信息来确定何时挥手，则应始终使用 `SUCCESS`。否则，如果它依赖于客户端不存在的服务器信息，则应使用 `SUCCESS_SERVER`。
:::

然后是 `InteractionResult.Fail`，由 `InteractionResult#FAIL` 实现，表示操作应被视为失败，不允许进一步交互发生。流程将结束。这可以在任何地方使用，但在 `Item#useOn` 和 `Item#use` 之外应谨慎使用。在许多情况下，使用 `InteractionResult#PASS` 更有意义。

最后是 `InteractionResult.Pass` 和 `InteractionResult.TryWithEmptyHandInteraction`，分别由 `InteractionResult#PASS` 和 `InteractionResult#TRY_WITH_EMPTY_HAND` 实现。这些记录表示操作应被视为既不成功也不失败，流程应继续。`PASS` 是所有 `InteractionResult` 方法的默认行为，除了 `BlockBehaviour#useItemOn`，它返回 `TRY_WITH_EMPTY_HAND`。更具体地说，如果 `BlockBehaviour#useItemOn` 返回除 `TRY_WITH_EMPTY_HAND` 之外的任何内容，则无论物品是否在主手中，`BlockBehaviour#useWithoutItem` 都不会被调用。

一些方法具有特殊行为或要求，在以下章节中解释。

#### `Item#useOn`

如果你希望操作被视为成功，但你不希望手臂挥动或授予 `ITEM_USED` 统计点，请使用 `InteractionResult#CONSUME` 并调用 `#withoutItem`。

``` java
// 在 Item#useOn 中
return InteractionResult.CONSUME.withoutItem();
```

#### `Item#use`

这是唯一一个使用来自 `Success` 变体（`SUCCESS`、`SUCCESS_SERVER`、`CONSUME`）的转换后 `ItemStack` 的实例。`Success#heldItemTransformedTo` 设置的最终 `ItemStack` 替换了启动使用时的 `ItemStack`，如果它已更改。

`Item#use` 的默认实现在物品可食用（具有 `DataComponents#CONSUMABLE`）且玩家可以吃物品（因为他们饿了，或者因为物品总是可食用）时返回 `InteractionResult#CONSUME`，在物品可食用（具有 `DataComponents#CONSUMABLE`）但玩家不能吃物品时返回 `InteractionResult#FAIL`。如果物品可装备（具有 `DataComponents#EQUIPPABLE`），则在用交换的物品（通过 `heldItemTransformedTo`）替换持有的物品时返回 `InteractionResult#SUCCESS`，或者如果盔甲上的附魔具有 `EnchantmentEffectComponents#PREVENT_ARMOR_CHANGE` 组件，则返回 `InteractionResult#FAIL`。如果物品可以格挡攻击（具有 `DataComponents#BLOCKS_ATTACKS`），则在返回 `InteractionResult#CONSUME` 之前调用 `Item#startUsingItem`。否则返回 `InteractionResult#PASS`。

在主手的情况下返回 `InteractionResult#FAIL` 将阻止副手行为运行。如果你希望副手行为运行（通常你希望），请改为返回 `InteractionResult#PASS`。

## 中键单击

- 如果 `Minecraft.getInstance().hitResult` 中的 [`HitResult`][hitresult] 为 null 或类型为 `MISS`，流程结束。
- 使用左鼠标按钮和主手触发 `InputEvent.InteractionKeyMappingTriggered`。如果[事件][event]被[取消][cancel]，流程结束。
- 根据你正在看的内容（使用 `Minecraft.getInstance().hitResult` 中的 `HitResult`），发生不同的事情：
    - 如果你正在看一个在你触及范围内的[实体]：
        - 如果 `Entity#isPickable` 返回 false，流程结束。
        - 如果 `Player#canInteractWithEntity` 返回 false，流程结束。
        - 调用 `Entity#getPickResult`。如果存在匹配的快捷键栏槽位，则将其设置为活动状态。否则，如果玩家处于创造模式，则将结果 `ItemStack` 添加到玩家的物品栏中。
            - 默认情况下，此方法转发到 `Entity#getPickResult`，模组开发者可以覆盖。
    - 如果你正在看一个在你触及范围内的[方块]：
        - 如果 `Player#canInteractWithBlock` 返回 false，流程结束。
        - 调用 `IBlockExtension#getCloneItemStack`（默认委托给 `BlockBehaviour#getCloneItemStack`）并成为“选定的” `ItemStack`。
            - 默认情况下，这返回 `Block` 的 `Item` 表示。
        - 如果按住 Control 键，玩家处于创造模式且目标方块具有 [`BlockEntity`][blockentity]：
            - 从 `BlockEntity#saveCustomOnly` 获取 `BlockEntity` 的数据
                - 作为后处理步骤调用 `BlockEntity#removeComponentsFromTag`。
            - 通过 `DataComponents#BLOCK_ENTITY_DATA` 将 `BlockEntity` 的数据添加到“选定的” `ItemStack`。
        - 如果存在匹配的快捷键栏槽位，则将其设置为活动状态。否则，如果玩家处于创造模式，则将“选定的” `ItemStack` 添加到玩家的物品栏中。

[attribute]: ../entities/attributes.md
[attributemodifier]: ../entities/attributes.md#attribute-modifiers
[block]: ../blocks/index.md
[blockbreak]: ../blocks/index.md#breaking-a-block
[blockentity]: ../blockentities/index.md
[cancel]: ../concepts/events.md#cancellable-events
[critical]: https://minecraft.wiki/w/Damage#Critical_hit
[effect]: mobeffects.md
[entity]: ../entities/index.md
[event]: ../concepts/events.md
[featureflag]: ../advanced/featureflags.md
[hitresult]: #hitresults
[hurt]: ../entities/index.md#damaging-entities
[itemstack]: index.md#itemstacks
[itemuseon]: #itemuseon
[livingentity]: ../entities/livingentity.md
[physicalside]: ../concepts/sides.md#the-physical-side
[side]: ../concepts/sides.md#the-logical-side