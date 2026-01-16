---
sidebar_position: 3
---
# 消耗品(Consumables)

消耗品是可以在一定时间内使用，并在过程中被“消耗”的[物品(Item)][item]。在 Minecraft 中任何可以被吃或喝的东西都是某种消耗品。

## `Consumable` 数据组件

任何可以被消耗的物品都有 [`DataComponents#CONSUMABLE` 组件][datacomponent]。支撑的记录 `Consumable` 定义了如何消耗物品以及消耗后应用什么效果。

可以通过直接调用记录构造函数或通过 `Consumable#builder` 创建 `Consumable`，后者为每个字段设置默认值，完成后调用 `build`：

- `consumeSeconds` - 一个 `float`，表示完全消耗物品所需的秒数。在所有分配的时间过去后，调用 `Item#finishUsingItem`。默认为 1.6 秒，或 32 刻。
- `animation` - 设置在使用物品时播放的 [`ItemUseAnimation`][animation]。默认为 `ItemUseAnimation#EAT`。
- `sound` - 设置消耗物品时播放的 [`SoundEvent`][sound]。这必须是一个 `Holder` 实例。默认为 `SoundEvents#GENERIC_EAT`。
    - 如果原版实例不是 `Holder<SoundEvent>`，可以通过调用 `BuiltInRegistries.SOUND_EVENT.wrapAsHolder(soundEvent)` 获取包装为 `Holder` 的版本。
- `soundAfterConsume` - 设置物品完全消耗后播放的 [`SoundEvent`][sound]。这委托给 [`PlaySoundConsumeEffect`][consumeeffect]。
- `hasConsumeParticles` - 为 `true` 时，每四刻以及物品完全消耗时生成物品[粒子(Particles)][particles]。默认为 `true`。
- `onConsume` - 添加一个 [`ConsumeEffect`][consumeeffect]，在物品通过 `Item#finishUsingItem` 完全消耗后应用。

原版在其 `Consumables` 类中提供了一些消耗品，例如 [食物(Food)][food] 物品的 `#defaultFood` 和 [药水(Potions)][potions] 以及牛奶桶的 `#defaultDrink`。

可以通过调用 `Item.Properties#component` 添加 `Consumable` 组件：

``` java
// 假设有一些 DeferredRegister.Items ITEMS
public static final DeferredItem<Item> CONSUMABLE = ITEMS.registerSimpleItem(
    "consumable",
    new Item.Properties().component(
        DataComponents.CONSUMABLE,
        Consumable.builder()
            // 花费 2 秒，或 40 刻，来消耗
            .consumeSeconds(2f)
            // 设置消耗时播放的动画
            .animation(ItemUseAnimation.BLOCK)
            // 每刻播放声音
            .sound(SoundEvents.ARMOR_EQUIP_CHAIN)
            // 完全消耗后播放声音
            .soundAfterConsume(SoundEvents.BREEZE_WIND_CHARGE_BURST)
            // 吃的时候不显示粒子
            .hasConsumeParticles(false)
            .onConsume(
                // 完全消耗后，以 30% 的几率应用效果
                new ApplyStatusEffectsConsumeEffect(new MobEffectInstance(MobEffects.HUNGER, 600, 0), 0.3F)
            )
            // 可以有多个
            .onConsume(
                // 在 50 格半径内随机传送实体
                new TeleportRandomlyConsumeEffect(100f)
            )
            .build()
    )
);
```

### `ConsumeEffect`

当消耗品使用完毕后，你可能希望触发某种逻辑执行，例如添加药水效果。这些由 `ConsumeEffect` 处理，通过调用 `Consumable.Builder#onConsume` 添加到 `Consumable`。

可以在 `ConsumeEffect` 中找到原版效果的列表。

每个 `ConsumeEffect` 都有两个方法：`getType`，指定注册对象 `ConsumeEffect.Type`；以及 `apply`，在物品完全消耗时调用。`apply` 接受三个参数：消耗实体所在的 `Level`，调用消耗品的 `ItemStack`，以及消耗对象的 `LivingEntity`。当效果成功应用时，方法返回 `true`，否则返回 `false`。

可以通过实现接口并向 `BuiltInRegistries#CONSUME_EFFECT_TYPE` [注册(Registering)][registering] 关联的 `MapCodec` 和 `StreamCodec` 来创建 `ConsumeEffect`：

``` java
public record UsePortalConsumeEffect(ResourceKey<Level> level)
    implements ConsumeEffect, Portal {

    @Override
    public boolean apply(Level level, ItemStack stack, LivingEntity entity) {
        if (entity.canUsePortal(false)) {
            entity.setAsInsidePortal(this, entity.blockPosition());

            // 可以成功使用传送门
            return true;
        }

        // 不能使用传送门
        return false;
    }

    @Override
    public ConsumeEffect.Type<? extends ConsumeEffect> getType() {
        // 设置为注册对象
        return USE_PORTAL.get();
    }

    @Override
    @Nullable
    public TeleportTransition getPortalDestination(ServerLevel level, Entity entity, BlockPos pos) {
        // 设置传送位置
    }
}

// 在某个注册类中
// 假设有一些 DeferredRegister<ConsumeEffect.Type<?>> CONSUME_EFFECT_TYPES
public static final Supplier<ConsumeEffect.Type<UsePortalConsumeEffect>> USE_PORTAL =
    CONSUME_EFFECT_TYPES.register("use_portal", () -> new ConsumeEffect.Type<>(
        ResourceKey.codec(Registries.DIMENSION).optionalFieldOf("dimension")
            .xmap(UsePortalConsumeEffect::new, UsePortalConsumeEffect::level),
        ResourceKey.streamCodec(Registries.DIMENSION)
            .map(UsePortalConsumeEffect::new, UsePortalConsumeEffect::level)
    ));

// 对于添加 CONSUMABLE 组件的某些 Item.Properties
Consumable.builder()
    .onConsume(
        new UsePortalConsumeEffect(Level.END)
    )
    .build();
```

### `ItemUseAnimation`

`ItemUseAnimation` 在功能上是一个枚举，除了其 id 和名称外不定义任何内容。它的用途被硬编码到第一人称的 `ItemHandRenderer#renderArmWithItem` 和第三人称的 `PlayerRenderer#getArmPose` 中。因此，仅创建一个新的 `ItemUseAnimation` 将只起到与 `ItemUseAnimation#NONE` 类似的作用。

要应用某些动画，你需要为第一人称实现 `IClientItemExtensions#applyForgeHandTransform` 和/或为第三人称渲染实现 `IClientItemExtensions#getArmPose`。

#### 创建 `ItemUseAnimation`

首先，让我们创建一个新的 `ItemUseAnimation`。这是使用[可扩展枚举(Extensible Enum)][extensibleenum]系统完成的：

``` json5
{
    "entries": [
        {
            "enum": "net/minecraft/world/item/ItemUseAnimation",
            "name": "EXAMPLEMOD_ITEM_USE_ANIMATION",
            "constructor": "(ILjava/lang/String;)V",
            "parameters": [
                // id，应始终为 -1
                -1,
                // 名称，应是一个唯一标识符
                "examplemod:item_use_animation"
            ]
        }
    ]
}
```

然后我们可以通过 `valueOf` 获取枚举常量：

``` java
public static final ItemUseAnimation EXAMPLE_ANIMATION = ItemUseAnimation.valueOf("EXAMPLEMOD_ITEM_USE_ANIMATION");
```

从那里，我们可以开始应用变换。为此，我们必须创建一个新的 `IClientItemExtensions`，实现所需的方法，并通过 [**模组事件总线**][modbus] 上的 `RegisterClientExtensionsEvent` 注册它：

``` java
public class ConsumableClientItemExtensions implements IClientItemExtensions {
    // 在这里实现方法
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerClientExtensions(RegisterClientExtensionsEvent event) {
    event.registerItem(
        // 物品扩展的实例
        new ConsumableClientItemExtensions(),
        // 使用此扩展的物品的可变参数
        CONSUMABLE
    )
}
```

#### 第一人称

所有消耗品都有的第一人称变换通过 `IClientItemExtensions#applyForgeHandTransform` 实现：

``` java
public class ConsumableClientItemExtensions implements IClientItemExtensions {

    // ...

    @Override
    public boolean applyForgeHandTransform(
        PoseStack poseStack, LocalPlayer player, HumanoidArm arm, ItemStack itemInHand,
        float partialTick, float equipProcess, float swingProcess
    ) {
        // 我们首先需要检查物品是否正在被使用并具有我们的动画
        HumanoidArm usingArm = entity.getUsedItemHand() == InteractionHand.MAIN_HAND
            ? entity.getMainArm()
            : entity.getMainArm().getOpposite();
        if (
            entity.isUsingItem() && entity.getUseItemRemainingTicks() > 0
            && usingArm == arm && itemInHand.getUseAnimation() == EXAMPLE_ANIMATION
        ) {
            // 对姿势堆叠应用变换（平移、缩放、mulPose）
            // ...
            return true;
        }

        // 什么都不做
        return false;
    }
}
```

#### 第三人称

第三人称变换（除了 `EAT` 和 `DRINK` 有特殊逻辑外）通过 `IClientItemExtensions#getArmPose` 实现，其中 `HumanoidModel.ArmPose` 也可以扩展以进行自定义变换。

由于 `ArmPose` 在其构造函数中需要一个 lambda 作为其一部分，因此必须使用 `EnumProxy` 引用：

``` json5
{
    "entries": [
        {
            "name": "EXAMPLEMOD_ITEM_USE_ANIMATION",
            // ...
        },
        {
            "enum": "net/minecraft/client/model/HumanoidModel$ArmPose",
            "name": "EXAMPLEMOD_ARM_POSE",
            "constructor": "(ZLnet/neoforged/neoforge/client/IArmPoseTransformer;)V",
            "parameters": {
                // 指向代理所在的类
                // 应单独作为一个仅客户端的类
                "class": "example/examplemod/client/MyClientEnumParams",
                // 枚举代理的字段名称
                "field": "CUSTOM_ARM_POSE"
            }
        }
    ]
}
```

``` java
// 创建枚举参数
public class MyClientEnumParams {
    public static final EnumProxy<HumanoidModel.ArmPose> CUSTOM_ARM_POSE = new EnumProxy<>(
        HumanoidModel.ArmPose.class,
        // 姿势是否使用双臂
        false,
        // 姿势变换器
        (IArmPoseTransformer) MyClientEnumParams::applyCustomModelPose
    );

    private static void applyCustomModelPose(
        HumanoidModel<?> model, HumanoidRenderState state, HumanoidArm arm
    ) {
        // 在此处应用模型变换
        // ...
    }
}

// 在某个仅客户端类中
public static final HumanoidModel.ArmPose EXAMPLE_POSE = HumanoidModel.ArmPose.valueOf("EXAMPLEMOD_ARM_POSE");
```

然后，通过 `IClientItemExtensions#getArmPose` 设置手臂姿势：

``` java
public class ConsumableClientItemExtensions implements IClientItemExtensions {

    // ...

    @Override
    public HumanoidModel.ArmPose getArmPose(
        LivingEntity entity, InteractionHand hand, ItemStack stack
    ) {
        // 我们首先需要检查物品是否正在被使用并具有我们的动画
        if (
            entity.isUsingItem() && entity.getUseItemRemainingTicks() > 0
            && entity.getUsedItemHand() == hand
            && itemInHand.getUseAnimation() == EXAMPLE_ANIMATION
        ) {
            // 返回要应用的姿势
            return EXAMPLE_POSE;
        }

        // 否则返回 null
        return null;
    }
}
```

### 在实体上覆盖声音

有时，实体可能希望在消耗物品时播放不同的声音。在这些情况下，[`LivingEntity`][livingentity] 实例可以实现 `Consumable.OverrideConsumeSound` 并让 `getConsumeSound` 返回他们希望实体播放的 `SoundEvent`。

``` java
public class MyEntity extends LivingEntity implements Consumable.OverrideConsumeSound {
    
    // ...

    @Override
    public SoundEvent getConsumeSound(ItemStack stack) {
        // 返回要播放的声音
    }
}
```

## `ConsumableListener`

虽然消耗品和消耗后应用的效果很有用，但有时效果的属性需要作为其他[数据组件(Data Components)][datacomponents]在外部可用。例如，猫和狼也吃[食物(Food)][food]并查询其营养价值，或者带有药水内容的物品查询其颜色用于渲染。在这些情况下，数据组件实现 `ConsumableListener` 以提供消耗逻辑。

`ConsumableListener` 只有一个方法：`#onConsume`，它接受当前世界、消耗物品的实体、被消耗的物品以及物品上的 `Consumable` 实例。`onConsume` 在物品完全消耗时于 `Item#finishUsingItem` 期间调用。

添加你自己的 `ConsumableListener` 只需[注册一个新的数据组件][datacompreg]并实现 `ConsumableListener`。

``` java
public record MyConsumableListener() implements ConsumableListener {

    @Override
    public void onConsume(
        Level level, LivingEntity entity, ItemStack stack, Consumable consumable
    ) {
        // 在此处做事
    }
}
```

### 食物(Food)

食物是一种 `ConsumableListener`，是饥饿系统的一部分。食物物品的所有功能已经由 `Item` 类处理，因此只需将 `FoodProperties` 添加到 `DataComponents#FOOD` 以及一个消耗品即可。有一个名为 `food` 的辅助方法，它接受 `FoodProperties` 和 `Consumable` 对象，如果未指定，则使用 `Consumables#DEFAULT_FOOD`。

可以通过直接调用记录构造函数或通过 `new FoodProperties.Builder()` 创建 `FoodProperties`，完成后调用 `build`：

- `nutrition` - 设置恢复多少饥饿点。以半饥饿点计数，例如，Minecraft 的牛排恢复 8 饥饿点。
- `saturationModifier` - 用于计算吃这个食物时恢复的[饱和值(Hunger)][hunger]的饱和修改器。计算公式为 `min(2 * nutrition * saturationModifier, playerNutrition)`，这意味着使用 `0.5` 将使有效饱和值与营养值相同。
- `alwaysEdible` - 是否总是可以吃这个物品，即使饥饿条已满。默认 `false`，对于金苹果和其他提供超出填饱饥饿条好处的物品为 `true`。

``` java
// 假设有一些 DeferredRegister.Items ITEMS
public static final DeferredItem<Item> FOOD = ITEMS.registerSimpleItem(
    "food",
    new Item.Properties().food(
        new FoodProperties.Builder()
            // 治疗 1.5 颗心
            .nutrition(3)
            // 胡萝卜为 0.3
            // 生鳕鱼为 0.1
            // 熟鸡肉为 0.6
            // 熟牛肉为 0.8
            // 金苹果为 1.2
            .saturationModifier(0.3f)
            // 设置后，即使饥饿条已满也可以吃。
            .alwaysEdible()
    )
);
```

有关示例或查看 Minecraft 使用的各种值，请查看 `Foods` 类。

要获取物品的 `FoodProperties`，请调用 `ItemStack.get(DataComponents.FOOD)`。这可能返回 null，因为并非每个物品都是可食用的。要确定物品是否可食用，请检查 `getFoodProperties` 调用的结果是否为 null。

### 药水内容(Potion Contents)

[药水(Potion)][potions]的内容通过 `PotionContents` 是另一个 `ConsumableListener`，其效果在消耗时应用。它们包含一个要应用的可选药水、一个用于药水颜色的可选色调、一个与药水一起应用的自定义 [`MobEffectInstance` 列表][mobeffectinstance]，以及一个在获取堆叠名称时使用的可选翻译键。如果不是 `PotionItem` 的子类型，模组开发者需要覆盖 `Item#getName`。

[animation]: #itemuseanimation
[consumeeffect]: #consumeeffect
[datacomponent]: datacomponents.md
[datacompreg]: datacomponents.md#creating-custom-data-components
[extensibleenum]: ../advanced/extensibleenums.md
[food]: #food
[hunger]: https://minecraft.wiki/w/Hunger#Mechanics
[item]: index.md
[livingentity]: ../entities/livingentity.md
[modbus]: ../concepts/events.md#event-buses
[mobeffectinstance]: mobeffects.md#mobeffectinstances
[particles]: ../resources/client/particles.md
[potions]: mobeffects.md#potions
[sound]: ../resources/client/sounds.md#creating-soundevents
[registering]: ../concepts/registries.md#methods-for-registering