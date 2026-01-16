---
sidebar_position: 3
---
# 生物实体、生物与玩家

生物实体是[实体][entities]的一个大类，它们都继承自共同的 `LivingEntity` 超类。这包括生物（通过 `Mob` 子类）、玩家（通过 `Player` 子类）和盔甲架（通过 `ArmorStand` 子类）。

生物实体具有许多常规实体没有的额外属性。这些包括[属性][attributes]、[状态效果][mobeffects]、伤害追踪等。

## 生命值、伤害与治疗

_另请参阅：[属性][attributes]。_

生物实体区别于其他实体的最显著特征之一是其完善的健康系统。生物实体通常有最大生命值、当前生命值，有时还有护甲或自然恢复等机制。

默认情况下，最大生命值由 `minecraft:max_health` [属性][attributes]决定，当前生命值在[生成][spawning]时被设置为相同的值。当实体通过调用 [`Entity#hurtServer`][hurt] 受到伤害时，当前生命值会根据伤害计算减少。许多实体，如僵尸，默认会保持在降低后的生命值，而有些实体，如玩家，可以恢复这些失去的生命值。

要获取或设置最大生命值，需要直接读取或写入属性，如下所示：

``` java
// 获取我们实体的属性映射。
AttributeMap attributes = entity.getAttributes();

// 获取我们实体的最大生命值。
float maxHealth = attributes.getValue(Attributes.MAX_HEALTH);
// 上述代码的快捷方式。
maxHealth = entity.getMaxHealth();

// 设置最大生命值必须通过获取 AttributeInstance 并调用 #setBaseValue 来完成，或者通过添加属性修饰符。我们在这里使用前者。更多细节请参阅属性文章。
attributes.getInstance(Attributes.MAX_HEALTH).setBaseValue(50);
```

当[受到伤害][damage]时，生物实体会应用一些额外的计算，例如考虑 `minecraft:armor` 属性（除非[伤害类型][damagetypes]属于 `minecraft:bypasses_armor` [标签][tags]）以及 `minecraft:absorption` 属性。生物实体还可以重写 `#onDamageTaken` 来执行攻击后的行为；只有当最终伤害值大于零时才会调用此方法。

### 伤害事件

由于伤害处理流程的复杂性，有多个事件可供你挂钩，这些事件按照列出的顺序触发。这通常用于修改并非（或不一定是）你拥有的实体所受到的伤害，例如，如果你想修改对来自 Minecraft 或其他模组的实体造成的伤害，或者你想修改对任何实体（可能是也可能不是你拥有的）造成的伤害。

所有这些事件的共同点是 `DamageContainer`。一个新的 `DamageContainer` 在攻击开始时被实例化，并在攻击结束后被丢弃。它包含原始的 [`DamageSource`][damagesources]、原始伤害量以及所有单独修改的列表——护甲、伤害吸收、[附魔][enchantments]、[状态效果][mobeffects]等。`DamageContainer` 被传递给下面列出的所有事件，你可以检查已经进行了哪些修改，以便根据需要做出自己的更改。

#### `EntityInvulnerabilityCheckEvent`

此事件允许模组同时为实体绕过和添加无敌效果。此事件也会对非生物实体触发。你可以使用此事件使实体免疫攻击，或剥离它可能拥有的现有免疫效果。

由于技术原因，挂钩到此事件的处理应该是确定性的，并且只取决于伤害类型。这意味着，无敌效果的随机概率，或只在特定伤害量以下应用的无敌效果，应改为在 `LivingIncomingDamageEvent` 中添加（见下文）。

#### `LivingIncomingDamageEvent`

此事件仅在服务器端调用，应主要用于两个主要用例：动态取消攻击，以及添加伤害减免修饰符回调。

动态取消攻击基本上就是添加一个非确定性的无敌效果，例如取消伤害的随机概率、取决于时间或所受伤害量的无敌效果等。一致的无敌效果应通过 `EntityInvulnerabilityCheckEvent` 执行（见上文）。

伤害减免修饰符回调允许你修改已执行的伤害减免的一部分。例如，它允许你将护甲伤害减免的效果降低 50%。这将正确地传播到状态效果，然后状态效果会使用不同的伤害量进行计算等。可以像这样添加伤害减免修饰符回调：

``` java
@SubscribeEvent // 在游戏事件总线上
public static void decreaseArmor(LivingIncomingDamageEvent event) {
    // 我们只将此减免应用于玩家，而不改变僵尸等
    if (event.getEntity() instanceof Player) {
        // 添加我们的伤害减免修饰符回调。
        event.getDamageContainer().addModifier(
            // 要目标的伤害减免。有关可能的值，请参阅 DamageContainer.Reduction 枚举。
            DamageContainer.Reduction.ARMOR,
            // 要执行的修改。输入为伤害容器和基础减免，输出为新的减免。输入和输出的减免都是浮点数。
            (container, baseReduction) -> baseReduction * 0.5f
        );
    }
}
```

回调按照添加的顺序应用。这意味着在更高[优先级][priority]的事件处理器中添加的回调将首先运行。

#### `LivingShieldBlockEvent`

此事件可用于完全自定义盾牌格挡。这包括引入额外的盾牌格挡、防止盾牌格挡、修改原版盾牌格挡检查、更改对盾牌或攻击物品造成的伤害、更改盾牌的视野弧度、允许远程攻击但格挡近战攻击（或反之）、被动格挡攻击（即不使用盾牌）、仅格挡一定百分比的伤害等。

请注意，此事件不适用于超出“类似盾牌”物品范围的免疫或攻击取消。

#### `ArmorHurtEvent`

这个事件应该很容易理解。它在计算攻击造成的护甲伤害时触发，可用于修改对哪件护甲造成多少耐久度伤害（如果有的话）。

#### `LivingDamageEvent.Pre`

此事件在伤害发生前立即调用。`DamageContainer` 已完全填充，最终伤害量可用，并且事件不能再被取消，因为此时攻击被认为是成功的。

此时，所有类型的修饰符都可用，允许你精细地修改伤害量。请注意，像护甲伤害这样的事情此时已经计算完毕。

#### `LivingDamageEvent.Post`

此事件在伤害已经造成、伤害吸收已减少、战斗追踪器已更新、统计数据和游戏事件已处理后调用。它不可取消，因为攻击已经发生。此事件通常用于攻击后效果。请注意，即使伤害量为零也会触发此事件，因此如果需要，请相应地检查该值。

如果你在自己的实体上调用此事件，应考虑重写 `ILivingEntityExtension#onDamageTaken()`。与 `LivingDamageEvent.Post` 不同，这仅在伤害大于零时调用。

## 状态效果

_请参阅[状态效果与药水][mobeffects]。_

## 装备

_请参阅[实体上的容器][containers]。_

## 继承层次结构

生物实体具有复杂的类层次结构。如前所述，有三个直接子类（红色类为 `abstract`，蓝色类则不是）：

``` mermaid
graph LR;
    LivingEntity-->ArmorStand;
    LivingEntity-->Mob;
    LivingEntity-->Avatar;
    
    class LivingEntity,Mob,Avatar red;
    class ArmorStand blue;
```

其中，`ArmorStand` 没有子类（也是唯一的非抽象类），因此我们将重点关注 `Mob` 和 `Avatar` 的类层次结构。

### `Mob` 的层次结构

`Mob` 的类层次结构如下所示（红色类为 `abstract`，蓝色类则不是）：

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

图中缺失的所有其他生物实体都是 `Animal` 或 `Monster` 的子类。

你可能已经注意到，这非常混乱。例如，为什么蜜蜂、鹦鹉等不是飞行生物？当深入研究 `Animal` 和 `Monster` 的子类层次结构时，这个问题变得更加严重，这里将不详细讨论（如果你感兴趣，请使用 IDE 的“显示层次结构”功能查找它们）。最好承认它，但不要担心。

让我们回顾一下最重要的类：

- `PathfinderMob`：包含（惊喜！）路径查找逻辑。
- `AgeableMob`：包含年龄和幼年实体的逻辑。具有幼年变体的僵尸和其他怪物不继承此类，它们反而是 `Monster` 的子类。
- `Animal`：大多数动物扩展的类。有进一步的抽象子类，例如 `AbstractHorse` 或 `TamableAnimal`。
- `Monster`：大多数游戏认为是怪物的实体的抽象类。与 `Animal` 一样，这有进一步的抽象子类，例如 `AbstractPiglin`、`AbstractSkeleton`、`Raider` 和 `Zombie`。
- `WaterAnimal`：水生动物（如鱼、鱿鱼和海豚）的抽象类。由于路径查找显著不同，这些动物与其他动物分开。

### `Avatar` 的层次结构

Avatar(化身) 不仅定义了玩家，还定义了类似玩家的模型。根据 Avatar 所在的[逻辑端][logicalsides]，会使用不同的类。除了 `FakePlayer`(伪玩家) 和 `Mannequin`(人体模型) 外，你不应需要构造 Avatar。

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

- `AbstractClientPlayer`：此类用作两个客户端玩家的基础，都用于表示[逻辑客户端][logicalsides]上的玩家。
- `LocalPlayer`：此类用于表示当前运行游戏的玩家。
- `RemotePlayer`：此类用于表示 `LocalPlayer` 在多人游戏中可能遇到的其他玩家。因此，`RemotePlayer` 在单人游戏环境中不存在。
- `ServerPlayer`：此类用于表示[逻辑服务器][logicalsides]上的玩家。
- `FakePlayer`：这是 `ServerPlayer` 的一个特殊子类，设计用作玩家的模拟，用于需要玩家上下文的非玩家机制。
- `Mannequin`：此类设计用作可摆姿势的玩家，通常没有任何 AI。
- `ClientMannequin`：此类用于表示[逻辑客户端][logicalsides]上的人体模型。

## 生成

除了[常规的生成方式][spawning]——即 `/summon` 命令和通过代码中的 `EntityType#spawn` 或 `Level#addFreshEntity` 的方式——，`Mob` 也可以通过其他一些方式生成。`ArmorStand` 可以通过常规方式生成，而 `Player` 不应由你自己实例化，除了 `FakePlayer`。

### 刷怪蛋

为生物[注册][register]刷怪蛋是常见的（虽然不是必需的）。这是通过 `SpawnEggItem` 类和 `DataComponents#ENTITY_DATA` [数据组件][datacomponent]完成的：

``` java
// 假设我们有一个名为 ITEMS 的 DeferredRegister.Items
DeferredItem<SpawnEggItem> MY_ENTITY_SPAWN_EGG = ITEMS.registerItem("my_entity_spawn_egg",
    properties -> new SpawnEggItem(
        // 传递给 lambda 的 properties。
        // 使用 `spawnEgg` 设置 DataComponent。
        // 这是在 lambda 中完成的，以防止实体类型在注册之前解析。
        properties.spawnEgg(MY_ENTITY_TYPE.get())
    ));
```

像任何其他物品一样，该物品应添加到[创造模式标签页][creative]，并且应添加[客户端物品][clientitem]、[模型][model]和[翻译][translation]。

### 自然生成

_另请参阅 [Entities/`MobCategory`][mobcategory], [Worldgen/Biome Modifers/Add Spawns][addspawns], [Worldgen/Biome Modifers/Add Spawn Costs][addspawncosts]; 以及 [Minecraft Wiki][mcwiki] 上的 [Spawn Cycle][spawncycle]。_

自然生成每刻对 `MobCategory#isFriendly()` 为 true 的实体（默认情况下所有非怪物实体）执行，每 400 刻（= 20 秒）对 `MobCategory#isFriendly()` 为 false 的实体（所有怪物）执行。如果 `MobCategory#isPersistent()` 返回 true（主要是动物），则此过程还会在区块生成时发生。

对于每个区块和生物类别，检查是否达到了生成上限。更技术地说，这是检查在周围的 `loadedChunks` 区域内，该 `MobCategory` 的实体是否少于 `MobCategory#getMaxInstancesPerChunk() * loadedChunks / 289` 个，其中 `loadedChunks` 最多是以当前区块为中心的 17x17 区块区域，如果加载的区块较少（由于渲染距离或类似原因），则可能更少。

接下来，对于每个区块，要求至少有一个玩家附近（附近意味着生物与玩家之间的距离小于等于128）的该 `MobCategory` 实体少于 `MobCategory#getMaxInstancesPerChunk()` 个，该 `MobCategory` 的生成才会发生。

如果满足条件，则从相关生物群系的生成数据中随机选择一个条目，如果找到合适的位置，则发生生成。最多尝试三次寻找随机位置；如果仍然找不到位置，则不会生成。

#### 示例

听起来复杂？让我们通过平原生物群系中的动物示例来走一遍这个过程。

在平原生物群系中，每刻，游戏尝试从 `CREATURE` 生物类别生成实体，该类别包含以下条目：

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

由于 `CREATURE` 的生成上限是 10，因此会扫描以每个玩家当前区块为中心的最多 17x17 区块，以查找其他 `CREATURE` 类型的实体。如果找到小于等于10 * chunkCount / 289 个实体（这基本上意味着在未加载的区块附近，生成的机会变得更高），则对每个找到的实体进行与最近玩家的距离检查。如果其中至少一个实体的距离大于 128，则可能发生生成。

如果所有这些检查都通过，则根据权重从上面的列表中选择一个生成条目。假设选择了猪。然后游戏检查区块中的一个随机位置是否适合生成实体。如果位置合适，则根据生成数据中指定的最小和最大计数生成实体（在我们的例子中正好是 4 头猪）。如果位置不合适，游戏会尝试另外两个不同的位置。如果仍然找不到位置，则取消生成。

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