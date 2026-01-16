---
sidebar_position: 3
---
# 活体实体、生物与玩家

活体实体是[实体][entities]的一个大类，它们都继承自通用的 `LivingEntity` 超类。这包括生物（通过 `Mob` 子类）、玩家（通过 `Player` 子类）和盔甲架（通过 `ArmorStand` 子类）。

活体实体拥有许多常规实体所没有的额外属性。这些包括[属性][attributes]、[状态效果][mobeffects]、伤害追踪等等。

## 生命值、伤害与治疗

_另请参阅：[属性][attributes]。_

活体实体区别于其他实体的最显著特征之一是其完善的生命值系统。活体实体通常拥有最大生命值、当前生命值，有时还包括护甲或自然生命恢复等。

默认情况下，最大生命值由 `minecraft:max_health` [属性][attributes]决定，当前生命值在[生成][spawning]时被设置为相同的值。当通过调用实体上的 [`Entity#hurtServer`][hurt] 使其受到伤害时，当前生命值会根据伤害计算减少。许多实体，如僵尸，默认会保持这个降低后的生命值，而另一些实体，如玩家，则可以恢复这些失去的生命值。

要获取或设置最大生命值，需要直接读取或写入属性，如下所示：

``` java
// 获取我们实体的属性映射。
AttributeMap attributes = entity.getAttributes();

// 获取我们实体的最大生命值。
float maxHealth = attributes.getValue(Attributes.MAX_HEALTH);
// 上述操作的快捷方式。
maxHealth = entity.getMaxHealth();

// 设置最大生命值必须通过获取 AttributeInstance 并调用 #setBaseValue，或者
// 通过添加属性修饰符来完成。我们在这里使用前者。更多细节请参考属性文章。
attributes.getInstance(Attributes.MAX_HEALTH).setBaseValue(50);
```

当[受到伤害][damage]时，活体实体会应用一些额外的计算，例如考虑 `minecraft:armor` 属性（除非[伤害类型][damagetypes]位于 `minecraft:bypasses_armor` [标签][tags]中）以及 `minecraft:absorption` 属性。活体实体还可以重写 `#onDamageTaken` 来执行攻击后的行为；仅当最终伤害值大于零时才会调用此方法。

### 伤害事件

由于伤害处理流程的复杂性，有多个事件可供你挂钩（hook into），这些事件按列出顺序触发。这通常用于你希望对非（或不一定是你自己的）实体进行的伤害修改，例如，如果你想修改对来自Minecraft或其他模组的实体造成的伤害，或者如果你想修改对任何实体（可能是也可能不是你的实体）造成的伤害。

所有这些事件共有的是 `DamageContainer`。一个新的 `DamageContainer` 在攻击开始时实例化，并在攻击结束后丢弃。它包含原始的 [`DamageSource`][damagesources]、原始伤害值以及所有独立修改项的列表 - 护甲、伤害吸收、[附魔][enchantments]、[状态效果][mobeffects]等。`DamageContainer` 会传递给下面列出的所有事件，你可以检查已经进行了哪些修改，以便根据需要做出自己的更改。

#### `EntityInvulnerabilityCheckEvent`

此事件允许模组为实体绕过或添加无敌状态。此事件也对非活体实体触发。你可以使用此事件使实体对攻击免疫，或移除其可能拥有的现有免疫效果。

由于技术原因，挂钩到此事件的处理程序应该是确定性的，并且只取决于伤害类型。这意味着，用于无敌效果的随机几率，或仅在一定伤害量以下才应用的无敌效果，应该改为在 `LivingIncomingDamageEvent` 中添加（见下文）。

#### `LivingIncomingDamageEvent`

此事件仅在服务器端调用，应用于两个主要用例：动态取消攻击，以及添加伤害减免修改器回调。

动态取消攻击基本上就是添加非确定性的无敌效果，例如取消伤害的随机几率、取决于时间或已承受伤害量的无敌效果等。一致的无敌效果应通过 `EntityInvulnerabilityCheckEvent` 执行（见上文）。

伤害减免修改器回调允许你修改所执行伤害减免的一部分。例如，它允许你将护甲伤害减免的效果降低50%。然后这将正确地传播到状态效果，使得状态效果处理的伤害量不同，等等。可以像这样添加伤害减免修改器回调：

``` java
@SubscribeEvent // 在游戏事件总线上
public static void decreaseArmor(LivingIncomingDamageEvent event) {
    // 我们只将此减免应用于玩家，保持僵尸等不变
    if (event.getEntity() instanceof Player) {
        // 添加我们的减免修改器回调。
        event.getDamageContainer().addModifier(
            // 要针对的伤害减免项。可能的取值请参见 DamageContainer.Reduction 枚举。
            DamageContainer.Reduction.ARMOR,
            // 要执行的修改。输入为伤害容器和基础减免值，
            // 输出为新的减免值。输入和输出的减免值均为浮点数。
            (container, baseReduction) -> baseReduction * 0.5f
        );
    }
}
```

回调按添加的顺序应用。这意味着在更高[优先级][priority]的事件处理器中添加的回调将首先运行。

#### `LivingShieldBlockEvent`

此事件可用于完全自定义盾牌格挡。这包括引入额外的盾牌格挡、阻止盾牌格挡、修改原版盾牌格挡检查、更改对盾牌或攻击物品造成的伤害、更改盾牌的视野范围、允许弹射物但格挡近战攻击（反之亦然）、被动格挡攻击（即不使用盾牌）、仅格挡一定百分比的伤害等。

请注意，此事件不适用于“类似盾牌”物品范围之外的免疫或攻击取消。

#### `ArmorHurtEvent`

此事件应该相当自解释。它在计算攻击造成的护甲伤害时触发，可用于修改对哪个护甲部件造成多少耐久度伤害（如果有的话）。

#### `LivingDamageEvent.Pre`

此事件在伤害即将造成之前立即调用。此时 `DamageContainer` 已完全填充，最终伤害值可用，事件不再可取消，因为攻击到此时点被认为已成功。

此时，各种修改器都可用，允许你精细地修改伤害值。请注意，像护甲伤害这样的事情此时已经完成。

#### `LivingDamageEvent.Post`

此事件在伤害已造成、伤害吸收已减少、战斗跟踪器已更新、统计数据和游戏事件已处理之后调用。它不可取消，因为攻击已经发生。此事件通常用于攻击后的效果。请注意，即使伤害量为零也会触发此事件，因此如果需要，请相应地检查该值。

如果你在自己的实体上调用此事件，应考虑改为重写 `ILivingEntityExtension#onDamageTaken()`。与 `LivingDamageEvent.Post` 不同，仅当伤害大于零时才会调用此方法。

## 状态效果

_参见[状态效果与药水][mobeffects]。_

## 装备

_参见[实体上的容器][containers]。_

## 类层次结构

活体实体具有复杂的类层次结构。如前所述，有三个直接子类（红色类是 `abstract`，蓝色类不是）：

``` mermaid
graph LR;
    LivingEntity-->ArmorStand;
    LivingEntity-->Mob;
    LivingEntity-->Avatar;
    
    class LivingEntity,Mob,Avatar red;
    class ArmorStand blue;
```

其中，`ArmorStand` 没有子类（并且也是唯一的非抽象类），因此我们将专注于 `Mob` 和 `Avatar` 的类层次结构。

### `Mob` 的层次结构

`Mob` 的类层次结构如下所示（红色类是 `abstract`，蓝色类不是）：

``` mermaid
graph LR;
    Mob-->AmbientCreature;
    AmbientCreature-->Bat;
    Mob-->EnderDragon;
    Mob-->Ghast;
    Mob-->Phantom;
    Mob-->PathfinderMob;
    PathfinderMob-->AbstractGolem;
    AbstractGolem-->CopperGolem;
    AbstractGolem-->IronGolem;
    AbstractGolem-->Shulker;
    AbstractGolem-->SnowGolem;
    PathfinderMob-->AgeableMob;
    AgeableMob-->AbstractVillager;
    AbstractVillager-->Villager;
    AbstractVillager-->WanderingTrader;
    AgeableMob-->AgeableWaterCreature;
    AgeableWaterCreature-->Dolphin;
    AgeableWaterCreature-->Squid;
    Squid-->GlowSquid;
    AgeableMob-->Animal;
    PathfinderMob-->Allay;
    PathfinderMob-->Monster;
    PathfinderMob-->WaterAnimal;
    WaterAnimal-->AbstractFish;
    AbstractFish-->AbstractSchoolingFish;
    AbstractSchoolingFish-->Cod;
    AbstractSchoolingFish-->Salmon;
    AbstractSchoolingFish-->TropicalFish;
    AbstractFish-->Pufferfish;
    AbstractFish-->Tadpole;
    Mob-->Slime;
    Slime-->MagmaCube;
    
    class Mob,AmbientCreature,PathfinderMob,AbstractGolem,AgeableMob,AbstractVillager,AgeableWaterCreature,Animal,Monster,WaterAnimal,AbstractFish,AbstractSchoolingFish red;
    class Bat,CopperGolem,EnderDragon,Ghast,Phantom,IronGolem,Shulker,SnowGolem,Villager,WanderingTrader,Dolphin,Squid,GlowSquid,Allay,Cod,Salmon,TropicalFish,Pufferfish,Tadpole,Slime,MagmaCube blue;
```

图中缺少的所有其他活体实体都是 `Animal` 或 `Monster` 的子类。

你可能已经注意到，这非常混乱。例如，为什么蜜蜂、鹦鹉等不是飞行生物？当深入研究 `Animal` 和 `Monster` 的子类层次结构时，这个问题变得更糟，这里不详细讨论（如果有兴趣，请使用IDE的“显示层次结构”功能查看）。最好承认它，但不必担心。

让我们看一下最重要的类：

- `PathfinderMob`：包含（惊喜！）寻路逻辑。
- `AgeableMob`：包含成长逻辑和幼年实体逻辑。僵尸和其他有幼年变体的怪物不继承此类，它们反而是 `Monster` 的子类。
- `Animal`：大多数动物扩展的类。有进一步的抽象子类，例如 `AbstractHorse` 或 `TamableAnimal`。
- `Monster`：游戏认为的大多数怪物的抽象类。与 `Animal` 类似，这有进一步的抽象子类，例如 `AbstractPiglin`、`AbstractSkeleton`、`Raider` 和 `Zombie`。
- `WaterAnimal`：水生动物（如鱼、鱿鱼和海豚）的抽象类。由于寻路方式显著不同，这些动物与其他动物分开处理。

### `Avatar` 的层次结构

化身(Avatar)不仅定义玩家，还定义类似玩家的模型。根据化身所在的端(side)，会使用不同的类。除了 `FakePlayer` 和 `Mannequin`，你永远不需要自己实例化化身。

``` mermaid
graph LR;
    Avatar-->Mannequin;
    Mannequin-->ClientMannequin;
    Avatar--Player;
    Player-->AbstractClientPlayer;
    AbstractClientPlayer-->LocalPlayer;
    AbstractClientPlayer-->RemotePlayer;
    Player-->ServerPlayer;
    ServerPlayer-->FakePlayer;
    
    class Avatar,Player,AbstractClientPlayer red;
    class ClientMannequin,LocalPlayer,RemotePlayer,ServerPlayer,FakePlayer blue;
```

- `AbstractClientPlayer`：此类用作两个客户端玩家的基类，两者都用于表示[逻辑客户端][logicalsides]上的玩家。
- `LocalPlayer`：此类用于表示当前运行游戏的玩家。
- `RemotePlayer`：此类用于表示 `LocalPlayer` 在多人游戏中可能遇到的其他玩家。因此，在单机游戏环境中不存在 `RemotePlayer`。
- `ServerPlayer`：此类用于表示[逻辑服务器][logicalsides]上的玩家。
- `FakePlayer`：这是一个特殊的 `ServerPlayer` 子类，设计用作玩家的模拟，用于需要玩家上下文的非玩家机制。
- `Mannequin`：此类设计用作可摆姿势的玩家，通常没有任何AI。
- `ClientMannequin`：此类用于表示[逻辑客户端][logicalsides]上的模型。

## 生成

除了[常规的生成方式][spawning] - 即 `/summon` 命令和通过 `EntityType#spawn` 或 `Level#addFreshEntity` 的代码内方式 -，`Mob` 也可以通过一些其他方式生成。`ArmorStand` 可以通过常规方式生成，而 `Player` 不应该由你自己实例化，`FakePlayer` 除外。

### 刷怪蛋

为生物[注册][register]刷怪蛋是常见的（尽管不是必需的）。这是通过 `SpawnEggItem` 类和 `DataComponents#ENTITY_DATA` [数据组件][datacomponent]完成的：

``` java
// 假设我们有一个名为 ITEMS 的 DeferredRegister.Items
DeferredItem<SpawnEggItem> MY_ENTITY_SPAWN_EGG = ITEMS.registerItem("my_entity_spawn_egg",
    properties -> new SpawnEggItem(
        // 传入lambda函数的属性。
        // 使用 `spawnEgg` 来设置 DataComponent。
        // 这在lambda中完成，以防止实体类型在注册之前被解析。
        properties.spawnEgg(MY_ENTITY_TYPE.get())
    ));
```

像任何其他物品一样，该物品应添加到[创造模式标签页][creative]中，并且应添加[客户端物品][clientitem]、[模型][model]和[翻译][translation]。

### 自然生成

_另请参阅 [实体/`MobCategory`][mobcategory]、[世界生成/生物群系修改器/添加生成][addspawns]、[世界生成/生物群系修改器/添加生成成本][addspawncosts]；以及 [Minecraft Wiki][mcwiki] 上的 [生成周期][spawncycle]。_

自然生成在每个游戏刻(tick)对 `MobCategory#isFriendly()` 为 true 的实体（默认情况下所有非怪物实体）执行一次，每 400 个游戏刻（= 20 秒）对 `MobCategory#isFriendly()` 为 false 的实体（所有怪物）执行一次。如果 `MobCategory#isPersistent()` 返回 true（主要是动物），则在区块生成时也会额外进行此过程。

对于每个区块和生物类别，会检查是否达到生成上限。更技术性地说，这是检查在周围的 `loadedChunks` 区域内，该 `MobCategory` 的实体数量是否少于 `MobCategory#getMaxInstancesPerChunk() * loadedChunks / 289`，其中 `loadedChunks` 最多是以当前区块为中心的 17x17 区块区域，如果加载的区块较少（由于渲染距离或类似原因），则会更少。

接下来，对于每个区块，要求在该生物类别的至少一个玩家附近（附近意味着生物与玩家之间的距离 <= 128）存在少于 `MobCategory#getMaxInstancesPerChunk()` 个该 `MobCategory` 的实体，该 `MobCategory` 的生成才会发生。

如果条件满足，则从相关生物群系的生成数据中随机选择一个条目，如果可以找到合适的位置，则发生生成。最多有三次尝试寻找随机位置；如果仍然找不到位置，则不会发生生成。

#### 示例

听起来很复杂？让我们以平原生物群系中的动物为例过一遍这个过程。

在平原生物群系中，每个游戏刻，游戏都尝试从 `CREATURE` 生物类别生成实体，该类别包含以下条目：

``` json5
[
    {"type": "minecraft:sheep",   "minCount": 4, "maxCount": 4, "weight": 12},
    {"type": "minecraft:pig",     "minCount": 4, "maxCount": 4, "weight": 10},
    {"type": "minecraft:chicken", "minCount": 4, "maxCount": 4, "weight": 10},
    {"type": "minecraft:cow",     "minCount": 4, "maxCount": 4, "weight": 8 },
    {"type": "minecraft:horse",   "minCount": 2, "maxCount": 6, "weight": 5 },
    {"type": "minecraft:donkey",  "minCount": 1, "maxCount": 3, "weight": 1 }
]
```

由于 `CREATURE` 的生成上限是 10，因此会扫描每个玩家当前区块周围最多 17x17 个区块内的其他 `CREATURE` 类型实体。如果找到 <= 10 * chunkCount / 289 个实体（这基本上意味着在未加载区块附近，生成的机会变得更高），则对每个找到的实体进行与最近玩家的距离检查。如果其中至少一个实体的距离大于 128，则可以发生生成。

如果所有这些检查都通过，则根据权重从上述列表中选择一个生成条目。假设选择了猪。然后游戏检查区块中的一个随机位置是否适合生成该实体。如果位置合适，则根据生成数据中指定的最小和最大数量生成实体（在我们的例子中正好是 4 头猪）。如果位置不合适，游戏会尝试另外两个不同的位置。如果仍然找不到位置，则取消生成。

[addspawncosts]: ../worldgen/biomemodifier.md#add-spawn-costs
[addspawns]: ../worldgen/biomemodifier.md#add-spawns
[attributes]: attributes.md
[clientitem]: ../resources/client/models/items.md
[containers]: ../inventories/container.md
[creative]: ../items/index.md#creative-tabs
[damage]: index.md#damaging-entities
[damagesources]: ../resources/server/damagetypes.md#creating-and-using-damage-sources
[damagetypes]: ../resources/server/damagetypes.md
[datacomponent]: ../items/datacomponents.md
[enchantments]: ../resources/server/enchantments/index.md
[entities]: index.md
[hurt]: index.md#damaging-entities
[logicalsides]: ../concepts/sides.md#the-logical-side
[mcwiki]: https://minecraft.wiki
[mobcategory]: index.md#mobcategory
[mobeffects]: ../items/mobeffects.md
[model]: ../resources/client/models/index.md
[priority]: ../concepts/events.md#priority
[register]: ../concepts/registries.md
[spawncycle]: https://minecraft.wiki/w/Mob_spawning#Spawn_cycle
[spawning]: index.md#spawning-entities
[tags]: ../resources/server/tags.md
[translation]: ../resources/client/i18n.md