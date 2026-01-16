---
sidebar_position: 6
---
# 状态效果(Mob Effects)与药水(Potions)

状态效果，有时称为药水效果，在代码中称为 `MobEffect`，是每刻影响一个 [`LivingEntity`（生物实体）][livingentity] 的效果。本文解释了如何使用它们、效果和药水之间的区别，以及如何添加你自己的效果。

## 术语

- 一个 `MobEffect` 每刻影响一个实体。像[方块][block]或[物品][item]一样，`MobEffect` 是注册对象，这意味着它们必须[注册][registration]并且是单例。
    - **瞬间状态效果**是一种特殊的状态效果，设计为应用一刻。Vanilla 有两个瞬间效果，瞬间治疗和瞬间伤害。
- 一个 `MobEffectInstance` 是一个 `MobEffect` 的实例，具有设置的持续时间、放大倍数和一些其他属性（见下文）。`MobEffectInstance` 与 `MobEffect` 的关系就像 [`ItemStack`s][itemstack] 与 `Item` 的关系一样。
- 一个 `Potion` 是 `MobEffectInstance` 的集合。Vanilla 主要将药水用于四种药水物品（见下文），但是，它们可以随意应用于任何物品。由物品决定是否以及如何使用其上设置的药水。
- **药水物品**是一种旨在设置药水的物品。这是一个非正式术语，Vanilla 的 `PotionItem` 类与此无关（它指的是“普通”药水物品）。Minecraft 目前有四种药水物品：药水、喷溅药水、滞留药水和药箭；但是模组可能会添加更多。

## `MobEffect`s

要创建你自己的 `MobEffect`，请继承 `MobEffect` 类：

``` java
public class MyMobEffect extends MobEffect {
    public MyMobEffect(MobEffectCategory category, int color) {
        super(category, color);
    }
    
    @Override
    public boolean applyEffectTick(ServerLevel level, LivingEntity entity, int amplifier) {
        // 在此处应用你的效果逻辑。

        // 如果当 shouldApplyEffectTickThisTick 返回 true 时返回 false，效果将立即被移除
        return true;
    }
    
    // 效果是否应该在此刻应用。例如，再生效果仅每 x 刻应用一次，取决于刻计数和放大倍数。
    @Override
    public boolean shouldApplyEffectTickThisTick(int tickCount, int amplifier) {
        return tickCount % 2 == 0; // 用你想要的任何检查替换此内容
    }
    
    // 首次将效果添加到实体时调用的实用方法。
    // 直到此效果的所有实例都从实体中移除后，才会再次调用此方法。
    @Override
    public void onEffectAdded(LivingEntity entity, int amplifier) {
        super.onEffectAdded(entity, amplifier);
    }

    // 将效果添加到实体时调用的实用方法。
    // 每次将此效果添加到实体时都会调用此方法。
    @Override
    public void onEffectStarted(LivingEntity entity, int amplifier) {
    }
}
```

像所有注册对象一样，`MobEffect` 必须[注册][registration]，如下所示：

``` java
// MOB_EFFECTS 是一个 DeferredRegister<MobEffect>
public static final Holder<MobEffect> MY_MOB_EFFECT = MOB_EFFECTS.register("my_mob_effect", () -> new MyMobEffect(
        //可以是 BENEFICIAL、NEUTRAL 或 HARMFUL。用于确定此效果的药水工具提示颜色。
        MobEffectCategory.BENEFICIAL,
        //效果粒子的颜色，RGB 格式。
        0xffffff
));
```

`MobEffect` 类还为向受影响的实体添加[属性修饰符][attributemodifier]提供了默认功能，并在效果过期或通过其他方式移除时移除它们。例如，速度效果为移动速度添加了一个属性修饰符。效果属性修饰符添加如下：

``` java
public static final Holder<MobEffect> MY_MOB_EFFECT = MOB_EFFECTS.register("my_mob_effect", () -> new MyMobEffect(...)
        .addAttributeModifier(Attributes.ATTACK_DAMAGE, ResourceLocation.fromNamespaceAndPath("examplemod", "effect.strength"), 2.0, AttributeModifier.Operation.ADD_VALUE)
);
```

### `InstantenousMobEffect`

如果你想创建一个瞬间效果，你可以使用辅助类 `InstantenousMobEffect` 而不是常规的 `MobEffect` 类，如下所示：

``` java
public class MyMobEffect extends InstantenousMobEffect {
    public MyMobEffect(MobEffectCategory category, int color) {
        super(category, color);
    }

    @Override
    public void applyEffectTick(ServerLevel level, LivingEntity entity, int amplifier) {
        // 在此处应用你的效果逻辑。
    }
}
```

然后，像正常一样[注册][registration]你的效果。

### 事件

许多效果在其他地方应用它们的逻辑。例如，飘浮效果在生物实体移动处理程序中应用。对于模组的 `MobEffect`，在[事件处理程序][events]中应用它们通常是有意义的。NeoForge 还提供了一些与效果相关的事件：

- `MobEffectEvent.Applicable` 在游戏检查 `MobEffectInstance` 是否可以应用于实体时触发。此事件可用于拒绝或强制将效果实例添加到目标。
- `MobEffectEvent.Added` 在 `MobEffectInstance` 添加到目标时触发。此事件包含有关可能存在于目标上的先前 `MobEffectInstance` 的信息。
- `MobEffectEvent.Expired` 在 `MobEffectInstance` 过期时触发，即计时器变为零。
- `MobEffectEvent.Remove` 在效果通过过期以外的其他方式从实体中移除时触发，例如通过喝牛奶或通过命令。

## `MobEffectInstance`s

`MobEffectInstance` 简单来说就是应用于实体的效果。通过调用构造函数创建 `MobEffectInstance`：

``` java
MobEffectInstance instance = new MobEffectInstance(
        // 要使用的状态效果。
        MobEffects.REGENERATION,
        // 要使用的持续时间，以刻为单位。如果未指定，默认为0。
        500,
        // 要使用的放大倍数。这是效果的“强度”，即力量 I、力量 II 等。
        // 必须在0到255之间（包括）。如果未指定，默认为0。
        0,
        // 效果是否是“环境”效果，意味着它是由环境源应用的，
        // Minecraft 目前有信标和潮涌核心。如果未指定，默认为 false。
        false,
        // 效果是否在物品栏中可见。如果未指定，默认为 true。
        true,
        // 效果图标是否在右上角可见。如果未指定，默认为 true。
        true
);
```

有几个构造函数重载可用，分别省略最后1-5个参数。

:::info
`MobEffectInstance` 是可变的。如果你需要副本，请调用 `new MobEffectInstance(oldInstance)`。
:::

### 使用 `MobEffectInstance`s

可以将 `MobEffectInstance` 添加到 `LivingEntity`，如下所示：

``` java
MobEffectInstance instance = new MobEffectInstance(...);
livingEntity.addEffect(instance);
```

类似地，`MobEffectInstance` 也可以从 `LivingEntity` 中移除。由于 `MobEffectInstance` 会覆盖实体上相同 `MobEffect` 的现有 `MobEffectInstance`，因此每个 `MobEffect` 和实体只能有一个 `MobEffectInstance`。因此，在移除时指定 `MobEffect` 就足够了：

``` java
livingEntity.removeEffect(MobEffects.REGENERATION);
```

:::info
`MobEffect` 只能应用于 `LivingEntity` 或其子类，即玩家和生物。物品或投掷的雪球等不能受 `MobEffect` 影响。
:::

## `Potion`s

通过使用你希望药水具有的 `MobEffectInstance` 调用 `Potion` 的构造函数来创建 `Potion`。例如：

``` java
//POTIONS 是一个 DeferredRegister<Potion>
public static final Holder<Potion> MY_POTION = POTIONS.register("my_potion", registryName -> new Potion(
    // 应用于药水的后缀
    registryName.getPath(),
    // 药水使用的效果
    new MobEffectInstance(MY_MOB_EFFECT, 3600)
));
```

药水的名称是第一个构造函数参数。它用作翻译键的后缀；例如，Vanilla 中的长效和强效药水变体使用它来具有与其基础变体相同的名称。

`new Potion` 的 `MobEffectInstance` 参数是一个可变参数。这意味着你可以向药水添加任意数量的效果。这也意味着可以创建空药水，即没有任何效果的药水。只需调用 `new Potion()` 即可完成！（顺便说一句，Vanilla 就是这样添加 `awkward` 药水的。）

`PotionContents` 类提供了与药水物品相关的各种辅助方法。药水物品通过 `DataComponent#POTION_CONTENTS` 存储它们的 `PotionContents`。

### 酿造

现在你的药水已经添加，药水物品可用于你的药水。但是，无法在生存模式中获得你的药水，所以让我们改变这一点！

传统上，药水是在酿造台中制作的。不幸的是，Mojang 没有为酿造配方提供[数据包][datapack]支持，因此我们必须有点老派，并通过 `RegisterBrewingRecipesEvent` 事件通过代码添加我们的配方。这样做如下：

``` java
@SubscribeEvent // 在游戏事件总线上
public static void registerBrewingRecipes(RegisterBrewingRecipesEvent event) {
    // 获取要添加配方的构建器
    PotionBrewing.Builder builder = event.getBuilder();

    // 将为所有容器药水（例如药水、喷溅药水、滞留药水）添加酿造配方
    builder.addMix(
        // 要应用的初始药水
        Potions.AWKWARD,
        // 酿造材料。这是酿造台顶部的物品。
        Items.FEATHER,
        // 结果药水
        MY_POTION
    );
}
```

[attributemodifier]: ../entities/attributes.md#attribute-modifiers
[block]: ../blocks/index.md
[commonsetup]: ../concepts/events.md#event-buses
[datapack]: ../resources/index.md#data
[events]: ../concepts/events.md
[item]: index.md
[itemstack]: index.md#itemstacks
[livingentity]: ../entities/livingentity.md
[registration]: ../concepts/registries.md#methods-for-registering
[uuidgen]: https://www.uuidgenerator.net/version4