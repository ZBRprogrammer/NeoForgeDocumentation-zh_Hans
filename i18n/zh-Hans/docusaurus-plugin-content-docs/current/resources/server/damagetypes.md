# 伤害类型与伤害源(Damage Types & Damage Sources)

伤害类型表示对[实体(entity)]施加的是哪种伤害——物理伤害、火焰伤害、溺水伤害、魔法伤害、虚空伤害等。伤害类型的区分用于各种免疫（例如，烈焰人不会受到火焰伤害）、附魔（例如，爆炸保护仅保护免受爆炸伤害）以及更多用例。

伤害类型可以说是伤害源的模板。或者换句话说，伤害源可以看作是伤害类型的实例。伤害类型在代码中作为[`ResourceKey`s][rk]存在，但它们的所有属性都在数据包中定义。另一方面，伤害源由游戏根据需要创建，基于数据包文件中的值。它们可以保存额外的上下文，例如攻击实体。

## 创建伤害类型(Creating Damage Types)

首先，您需要创建自己的 `DamageType`。`DamageType`s 是一个[数据包注册表(datapack registry)][dr]，因此，新的 `DamageType`s 不是在代码中注册的，而是在添加相应文件时自动注册的。但是，我们仍然需要为代码提供一个获取伤害源的点。我们通过指定一个[资源键(resource key)][rk]来实现：

```java
public static final ResourceKey<DamageType> EXAMPLE_DAMAGE =
        ResourceKey.create(Registries.DAMAGE_TYPE, ResourceLocation.fromNamespaceAndPath(ExampleMod.MOD_ID, "example"));
```

现在我们可以从代码中引用它了，让我们在数据文件中指定一些属性。我们的数据文件位于 `data/examplemod/damage_type/example.json`（将 `examplemod` 和 `example` 替换为模组ID和资源位置的名称），并包含以下内容：

```json5
{
    // 伤害类型的死亡消息ID。完整的死亡消息翻译键将是
    // "death.attack.examplemod.example"（替换了模组ID和名称）。
    "message_id": "examplemod.example",
    // 此伤害类型的伤害量是否随难度缩放。有效的原版值有：
    // - "never"：伤害值在任何难度下保持不变。常用于玩家造成的伤害类型。
    // - "when_caused_by_living_non_player"：如果伤害由某种生物实体（包括间接造成，例如骷髅射出的箭）造成，且不是玩家，则伤害值会缩放。
    // - "always"：伤害值总是缩放。常用于爆炸类伤害。
    "scaling": "when_caused_by_living_non_player",
    // 受到此类伤害时造成的消耗度(exhaustion)量。
    "exhaustion": 0.1,
    // 受到此类伤害时应用的效果（目前仅为音效）。可选。
    // 有效的原版值有 "hurt"（默认）、"thorns"、"drowning"、"burning"、"poking" 和 "freezing"。
    "effects": "hurt",
    // 死亡消息类型。决定如何构建死亡消息。可选。
    // 有效的原版值有 "default"（默认）、"fall_variants" 和 "intentional_game_design"。
    "death_message_type": "default"
}
```

:::提示
`scaling`、`effects` 和 `death_message_type` 字段在内部分别由枚举 `DamageScaling`、`DamageEffects` 和 `DeathMessageType` 控制。如果需要，可以[扩展(extended)][extenum]这些枚举以添加自定义值。
:::

相同的格式也用于原版的伤害类型，如果需要，包开发者可以更改这些值。

## 创建和使用伤害源(Creating and Using Damage Sources)

`DamageSource`s 通常是在调用 [`Entity#hurt`][entityhurt] 时动态创建的。请注意，由于伤害类型是[数据包注册表(datapack registry)][dr]，您将需要 `RegistryAccess` 来查询它们，这可以通过 `Level#registryAccess` 获得。要创建 `DamageSource`，请使用最多四个参数调用 `DamageSource` 构造函数：

```java
DamageSource damageSource = new DamageSource(
        // 要使用的伤害类型持有者(Holder)。从注册表查询。这是唯一必需的参数。
        registryAccess.lookupOrThrow(Registries.DAMAGE_TYPE).getOrThrow(EXAMPLE_DAMAGE),
        // 直接实体。例如，如果骷髅射中了你，骷髅将是造成伤害的实体
        // （= 上面的参数），而箭将是直接实体（= 此参数）。与
        // 造成伤害的实体类似，这并非总是适用，因此可为 null。可选，默认为 null。
        null,
        // 造成伤害的实体。这并非总是适用（例如，当掉出世界时）
        // 因此可能为 null。可选，默认为 null。
        null,
        // 伤害源位置。这很少使用，一个例子是故意游戏设计
        // （= 下界床爆炸）。可为 null 且可选，默认为 null。
        null
);
```

:::警告
`DamageSources#source` 是 `new DamageSource` 的包装器，它颠倒了第二个和第三个参数（直接实体和造成伤害的实体）。请确保向正确的参数提供了正确的值。
:::

如果 `DamageSource`s 没有任何实体或位置上下文，将其缓存在字段中是有意义的。对于确实具有实体或位置上下文的 `DamageSource`s，通常会添加辅助方法，如下所示：

```java
public static DamageSource exampleDamage(Entity causer) {
    return new DamageSource(
            causer.level().registryAccess().lookupOrThrow(Registries.DAMAGE_TYPE).getOrThrow(EXAMPLE_DAMAGE),
            causer);
}
```

:::提示
原版的 `DamageSource` 工厂可以在 `DamageSources` 中找到，原版的 `DamageType` 资源键可以在 `DamageTypes` 中找到。实体也有方法 `Entity#damageSources`，这是 `DamageSources` 实例的便捷获取器。
:::

伤害源首要和最主要的用例是 `Entity#hurt`。每当实体受到伤害时就会调用此方法。要用我们自己的伤害类型伤害实体，我们只需自己调用 `Entity#hurt`：

```java
// 第二个参数是伤害量，以半心为单位。
entity.hurt(exampleDamage(player), 10);
```

其他特定于伤害类型的行为，例如无敌检查，通常通过伤害类型[标签(tags)]运行。这些由 Minecraft 和 NeoForge 添加，可以分别在 `DamageTypeTags` 和 `Tags.DamageTypes` 下找到。

## 数据生成(Datagen)

_更多信息，请参见[数据包注册表的数据生成(Data Generation for Datapack Registries)][drdatagen]。_

伤害类型 JSON 文件可以[数据生成(datagenned)][datagen]。由于伤害类型是数据包注册表，我们通过 `GatherDataEvent#createDatapackRegistryObjects` 添加一个 `DatapackBuiltinEntriesProvider`，并将我们的伤害类型放入 `RegistrySetBuilder` 中：

```java
// 在您的数据生成类中
@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void onGatherData(GatherDataEvent.Client event) {
    event.createDatapackRegistryObjects(new RegistrySetBuilder()
        // 为伤害类型添加数据包内置条目提供者。如果此 lambda 变得更长，
        // 为了可读性，应可能将其提取到单独的方法中。
        .add(Registries.DAMAGE_TYPE, bootstrap -> {
            // 使用 new DamageType() 创建伤害类型的代码内表示。
            // 参数按上面看到的顺序映射到 JSON 文件的值。
            // 除了消息ID和消耗度值之外的所有参数都是可选的。
            bootstrap.register(EXAMPLE_DAMAGE, new DamageType(EXAMPLE_DAMAGE.location(),
                DamageScaling.WHEN_CAUSED_BY_LIVING_NON_PLAYER,
                0.1f,
                DamageEffects.HURT,
                DeathMessageType.DEFAULT)
            )
        })
        // 如果适用，为其他数据包条目添加数据包提供者。
        .add(...)
    );

    // ...
}
```

[数据生成(datagen)]: ../index.md#data-generation
[数据包注册表(dr)]: ../../concepts/registries.md#datapack-registries
[数据包注册表数据生成(drdatagen)]: ../../concepts/registries.md#data-generation-for-datapack-registries
[实体(entity)]: ../../entities/index.md
[实体伤害(entityhurt)]: ../../entities/index.md#damaging-entities
[扩展枚举(extenum)]: ../../advanced/extensibleenums.md
[资源键(rk)]: ../../misc/resourcelocation.md#resourcekeys
[标签(tags)]: tags.md