---
sidebar_position: 5
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 盔甲(Armor)

盔甲是[物品(Item)][item]，其主要用途是利用各种抗性和效果保护[`LivingEntity`(生物实体)][livingentity]免受伤害。许多模组添加了新的盔甲套装（例如铜盔甲）。

## 自定义盔甲套装(Custom Armor Sets)

类人实体的盔甲套装通常由四个物品组成：头盔、胸甲、护腿和靴子。还有适用于狼、马和羊驼的盔甲，它们被应用于专门为动物设置的“身体”盔甲槽位。所有这些物品通常通过七个[数据组件(Data Component)][datacomponents]来实现：

- `DataComponents#MAX_DAMAGE` 和 `#DAMAGE` 用于耐久度
- `#MAX_STACK_SIZE` 将堆叠大小设置为 `1`
- `#REPAIRABLE` 用于在铁砧中修复盔甲部件
- `#ENCHANTABLE` 用于最大[附魔(Enchanting)][enchantment]值
- `#ATTRIBUTE_MODIFIERS` 用于盔甲值、盔甲韧性和击退抗性
- `#EQUIPPABLE` 用于实体如何装备该物品。

通常，每个盔甲都使用 `Item.Properties#humanoidArmor` 为类人实体设置，使用 `wolfArmor` 为狼设置，使用 `horseArmor` 为马设置。它们都使用 `ArmorMaterial` 结合 `ArmorType`（对于类人实体）来设置组件。参考值可以在 `ArmorMaterials` 中找到。此示例使用铜盔甲材料，你可以根据需要调整其值。

``` java
// 用于链接下面讨论的 `EquipmentClientInfo` JSON 的设备资产资源键。
// 指向 assets/examplemod/equipment/copper.json
public static final ResourceKey<EquipmentAsset> COPPER_ASSET = ResourceKey.create(EquipmentAssets.ROOT_ID, ResourceLocation.fromNamespaceAndPath("examplemod", "copper"));

public static final ArmorMaterial COPPER_ARMOR_MATERIAL = new ArmorMaterial(
    // 盔甲材料的耐久度乘数。
    // ArmorType 有不同的单位耐久度，乘数将应用于此：
    // - HELMET: 11
    // - CHESTPLATE: 16
    // - LEGGINGS: 15
    // - BOOTS: 13
    // - BODY: 16
    15,
    // 决定防御值（或护甲条上的半护甲数量）。
    // 基于 ArmorType。
    Util.make(new EnumMap<>(ArmorType.class), map -> {
        map.put(ArmorItem.Type.BOOTS, 2);
        map.put(ArmorItem.Type.LEGGINGS, 4);
        map.put(ArmorItem.Type.CHESTPLATE, 6);
        map.put(ArmorItem.Type.HELMET, 2);
        map.put(ArmorItem.Type.BODY, 4);
    }),
    // 决定盔甲的可附魔性。这表示此盔甲上附魔的效果将有多好。
    // 金使用 25；我们将铜设置得略低于此。
    20,
    // 决定装备此盔甲时播放的声音。
    // 此声音被 Holder 包装。
    SoundEvents.ARMOR_EQUIP_GENERIC,
     // 返回盔甲的韧性值。韧性值是伤害计算中包含的额外值，
    // 更多信息，请参阅 Minecraft Wiki 关于盔甲机制的文章：
    // https://minecraft.wiki/w/Armor#Armor_toughness
    // 只有钻石和下界合金的韧性值大于 0，因此我们只返回 0。
    0,
    // 返回盔甲的击退抗性值。穿戴此盔甲时，玩家在一定程度免疫击退。
    // 如果玩家从所有盔甲部件组合获得的总击退抗性值大于或等于 1，
    // 他们将完全不会受到击退。
    // 只有下界合金的击退抗性值大于 0，因此我们只返回 0。
    0,
    // 决定哪些物品可以修复此盔甲的标签。
    Tags.Items.INGOTS_COPPER,
    // 下面讨论的 EquipmentClientInfo JSON 的资源键。
    COPPER_ASSET
);
```

现在我们有了 `ArmorMaterial`，我们可以用它来[注册(Registering)][registering]盔甲：

``` java
// ITEMS 是一个 DeferredRegister.Items
public static final DeferredItem<Item> COPPER_HELMET = ITEMS.registerItem(
    "copper_helmet",
    props -> new Item(
        props.humanoidArmor(
            // 使用的材料。
            COPPER_ARMOR_MATERIAL,
            // 要创建的盔甲类型。
            ArmorType.HELMET
        )
    )
);

public static final DeferredItem<Item> COPPER_CHESTPLATE =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));
public static final DeferredItem<Item> COPPER_LEGGINGS =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));
public static final DeferredItem<Item> COPPER_BOOTS =
    ITEMS.registerItem("copper_chestplate", props -> new Item(props.humanoidArmor(...)));

public static final DeferredItem<Item> COPPER_WOLF_ARMOR = ITEMS.registerItem(
    "copper_wolf_armor",
    props -> new Item(
        // 使用的材料。
        props.wolfArmor(COPPER_ARMOR_MATERIAL)
    )
);

public static final DeferredItem<Item> COPPER_HORSE_ARMOR =
    ITEMS.registerItem("copper_horse_armor", props -> new Item(props.horseArmor(...)));
```

如果你想从头开始创建盔甲或类似盔甲的物品，可以使用以下部分的组合来实现：

- 通过 `Item.Properties#component` 设置 `DataComponents#EQUIPPABLE` 来添加一个带有你自己要求的 `Equippable`。
- 通过 `Item.Properties#attributes` 为物品添加属性（例如盔甲值、韧性、击退抗性）。
- 通过 `Item.Properties#durability` 为物品添加耐久度。
- 通过 `Item.Properties#repairable` 允许物品被修复。
- 通过 `Item.Properties#enchantable` 允许物品被附魔。
- 将你的盔甲添加到一些 `minecraft:enchantable/*` `ItemTags` 中，以便你的物品可以应用某些附魔。

### `Equippable`

`Equippable` 是一个数据组件，它包含实体如何装备此物品以及游戏中处理渲染的方式。这使得任何物品，无论是否被视为“盔甲”，只要拥有此组件就可以被装备（例如，马鞍、羊驼上的地毯）。每个拥有此组件的物品只能装备到单个 `EquipmentSlot`。

可以通过直接调用记录构造函数或通过 `Equippable#builder` 创建 `Equippable`，后者为每个字段设置默认值，完成后调用 `build`：

``` java
// 用于链接下面讨论的 `EquipmentClientInfo` JSON 的设备资产资源键。
// 指向 assets/examplemod/equipment/equippable.json
public static final ResourceKey<EquipmentAsset> EXAMPLE_EQUIPABBLE = ResourceKey.create(EquipmentAssets.ROOT_ID, ResourceLocation.fromNamespaceAndPath("examplemod", "equippable"));

// 假设有一些 DeferredRegister.Items ITEMS
public static final DeferredItem<Item> EQUIPPABLE = ITEMS.registerSimpleItem(
    "equippable",
    new Item.Properties().component(
        DataComponents.EQUIPPABLE,
        // 设置此物品可以装备到的槽位。
        Equippable.builder(EquipmentSlot.HELMET)
            // 决定装备此物品时播放的声音。
            // 此声音被 Holder 包装。
            // 默认为 SoundEvents#ARMOR_EQUIP_GENERIC。
            .setEquipSound(SoundEvents.ARMOR_EQUIP_GENERIC)
            // 下面讨论的 EquipmentClientInfo JSON 的资源键。
            // 未设置时，不渲染装备。
            .setAsset(ResourceKey.create(EXAMPLE_EQUIPABBLE))
            // 穿戴时在玩家屏幕上覆盖纹理的相对位置（例如，南瓜模糊效果）。
            // 指向 assets/examplemod/textures/equippable.png
            // 未设置时，不渲染覆盖层。
            .setCameraOverlay(ResourceLocation.withDefaultNamespace("examplemod", "equippable"))
            // 可以装备此物品的实体类型（直接或标签）的 HolderSet。
            // 未设置时，任何实体都可以装备此物品。
            .setAllowedEntities(EntityType.ZOMBIE)
            // 物品是否可以从发射器中装备。
            // 默认为 true。
            .setDispensable(true),
            // 物品是否可以在快速装备时从玩家身上换下。
            // 默认为 true。
            .setSwappable(false),
            // 物品在受到攻击时是否应该受到损坏（通常用于装备）。
            // 还必须是一个可损坏的物品。
            // 默认为 true。
            .setDamageOnHurt(false)
            // 物品是否可以通过交互（例如，右键单击）装备到另一个实体上。
            // 默认为 false。
            .setEquipOnInteract(true)
            // 为 true 时，具有 SHEAR_REMOVE_ARMOR 物品能力的物品可以移除装备的物品。
            // 默认为 false。
            .setCanBeSheared(true)
            // 剪切此装备物品时播放的声音。
            // 此声音被 Holder 包装。
            // 默认为 SoundEvents#SHEARS_SNIP。
            .setShearingSound(SoundEvents.SADDLE_UNEQUIP)
            .build()
    )
);
```

## 设备资产(Equipment Assets)

现在我们在游戏中有了一些盔甲，但如果我们尝试穿戴它，什么也不会渲染，因为我们从未指定如何渲染装备。为此，我们需要在 `Equippable#assetId` 指定的位置创建一个 `EquipmentClientInfo` JSON，相对于[资源包(Resource Pack)][respack]的 `equipment` 文件夹（`assets` 文件夹）。`EquipmentClientInfo` 指定了用于渲染每个图层的关联纹理。

`EquipmentClientInfo` 在功能上是一个从 `EquipmentClientInfo.LayerType` 到要应用的 `EquipmentClientInfo.Layer` 列表的映射。

`LayerType` 可以被视为要为某个实例渲染的一组纹理。例如，`LayerType#HUMANOID` 被 `HumanoidArmorLayer` 用于在类人实体上渲染头部、胸部和脚部；`LayerType#WOLF_BODY` 被 `WolfArmorLayer` 用于渲染身体盔甲。如果它们是同一种可装备物品（如铜盔甲），可以将它们组合到一个设备信息 JSON 中。

`LayerType` 映射到要应用和渲染的 `Layer` 列表，按照提供的顺序渲染纹理。一个 `Layer` 实际上代表要渲染的单个纹理。第一个参数表示纹理的位置，相对于 `textures/entity/equipment`。

第二个参数是一个可选参数，指示[纹理是否可以被染色(Tinting)][tinting]，作为一个 `EquipmentClientInfo.Dyeable`。`Dyeable` 对象持有一个整数，当存在时，表示用于染色纹理的默认 RGB 颜色。如果此可选参数不存在，则使用纯白色。

:::warning
要将非未染色颜色的染色应用于物品，该物品必须在 [`ItemTags#DYEABLE`][tag] 中，并且 `DataComponents#DYED_COLOR` 组件必须设置为某个 RGB 值。
:::

第三个参数是一个布尔值，指示在渲染期间是否应使用提供的纹理，而不是 `Layer` 中定义的纹理。一个例子是玩家的自定义披风或自定义鞘翅纹理。

让我们为铜盔甲材料创建一个设备信息。我们还将假设每个图层有两个纹理：一个是实际的盔甲纹理，另一个是覆盖并染色的纹理。对于动物盔甲，我们将假设有一些动态纹理可以使用。

<Tabs>
<TabItem value="json" label="JSON" default>

``` json5
// 在 assets/examplemod/equipment/copper.json 中
{
    // 图层映射
    "layers": {
        // 要应用的 EquipmentClientInfo.LayerType 的序列化名称。
        // 用于类人实体头部、胸部和脚部
        "humanoid": [
            // 按提供顺序渲染的图层列表
            {
                // 盔甲的相关纹理
                // 指向 assets/examplemod/textures/entity/equipment/humanoid/copper/outer.png
                "texture": "examplemod:copper/outer"
            },
            {
                // 覆盖纹理
                // 指向 assets/examplemod/textures/entity/equipment/humanoid/copper/outer_overlay.png
                "texture": "examplemod:copper/outer_overlay",
                // 指定时，允许纹理染成 DataComponents#DYED_COLOR 中的颜色
                // 否则，不能染色
                "dyeable": {
                    // 一个 RGB 值（总是不透明颜色）
                    // 0x7683DE 作为十进制
                    // 未指定时，设置为 0（意味着透明或不可见）
                    "color_when_undyed": 7767006
                }
            }
        ],
        // 用于类人实体腿部
        "humanoid_leggings": [
            {
                // 指向 assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner.png
                "texture": "examplemod:copper/inner"
            },
            {
                // 指向 assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner_overlay.png
                "texture": "examplemod:copper/inner_overlay",
                "dyeable": {
                    "color_when_undyed": 7767006
                }
            }
        ],
        // 用于狼盔甲
        "wolf_body": [
            {
                // 指向 assets/examplemod/textures/entity/equipment/wolf_body/copper/wolf.png
                "texture": "examplemod:copper/wolf",
                // 为 true 时，使用传入图层渲染器的纹理
                "use_player_texture": true
            }
        ],
        // 用于马盔甲
        "horse_body": [
            {
                // 指向 assets/examplemod/textures/entity/equipment/horse_body/copper/horse.png
                "texture": "examplemod:copper/horse",
                "use_player_texture": true
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="数据生成">

``` java
public class MyEquipmentInfoProvider extends EquipmentAssetProvider {

    public MyEquipmentInfoProvider(PackOutput output) {
        super(output);
    }

    @Override
    protected void registerModels(BiConsumer<ResourceKey<EquipmentAsset>, EquipmentClientInfo> output) {
        output.accept(
            // 必须匹配 Equippable#assetId
            COPPER_ASSET,
            EquipmentClientInfo.builder()
                // 用于类人实体头部、胸部和脚部
                .addLayers(
                    EquipmentClientInfo.LayerType.HUMANOID,
                    // 基础纹理
                    new EquipmentClientInfo.Layer(
                        // 盔甲的相关纹理
                        // 指向 assets/examplemod/textures/entity/equipment/humanoid/copper/outer.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/outer"),
                        Optional.empty(),
                        false
                    ),
                    // 覆盖纹理
                    new EquipmentClientInfo.Layer(
                        // 覆盖纹理
                        // 指向 assets/examplemod/textures/entity/equipment/humanoid/copper/outer_overlay.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/outer_overlay"),
                        // 一个 RGB 值（总是不透明颜色）
                        // 未指定时，设置为 0（意味着透明或不可见）
                        Optional.of(new EquipmentClientInfo.Dyeable(Optional.of(0x7683DE))),
                        false
                    )
                )
                // 用于类人实体腿部
                .addLayers(
                    EquipmentClientInfo.LayerType.HUMANOID_LEGGINGS,
                    new EquipmentClientInfo.Layer(
                        // 指向 assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/inner"),
                        Optional.empty(),
                        false
                    ),
                    new EquipmentClientInfo.Layer(
                        // 指向 assets/examplemod/textures/entity/equipment/humanoid_leggings/copper/inner_overlay.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/inner_overlay"),
                        Optional.of(new EquipmentClientInfo.Dyeable(Optional.of(0x7683DE))),
                        false
                    )
                )
                // 用于狼盔甲
                .addLayers(
                    EquipmentClientInfo.LayerType.WOLF_BODY,
                    // 基础纹理
                    new EquipmentClientInfo.Layer(
                        // 指向 assets/examplemod/textures/entity/equipment/wolf_body/copper/wolf.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/wolf"),
                        Optional.empty(),
                        // 为 true 时，使用传入图层渲染器的纹理
                        true
                    )
                )
                // 用于马盔甲
                .addLayers(
                    EquipmentClientInfo.LayerType.HORSE_BODY,
                    // 基础纹理
                    new EquipmentClientInfo.Layer(
                        // 指向 assets/examplemod/textures/entity/equipment/horse_body/copper/horse.png
                        ResourceLocation.fromNamespaceAndPath("examplemod", "copper/horse"),
                        Optional.empty(),
                        true
                    )
                )
                .build()
        );
    }
}

@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(MyEquipmentInfoProvider::new);
}
```

</TabItem>
</Tabs>

## 装备渲染(Equipment Rendering)

设备信息通过 `EquipmentLayerRenderer` 在 `EntityRenderer` 或其 `RenderLayer` 之一的渲染函数中渲染。`EquipmentLayerRenderer` 作为渲染上下文的一部分通过 `EntityRendererProvider.Context#getEquipmentRenderer` 获取。如果需要 `EquipmentClientInfo`，也可以通过 `EntityRendererProvider.Context#getEquipmentAssets` 获取。

默认情况下，以下图层渲染关联的 `EquipmentClientInfo.LayerType`：

| `LayerType`             | `RenderLayer`          | 使用对象                                                         |
|:-----------------------:|:----------------------:|:---------------------------------------------------------------|
| `HUMANOID`              | `HumanoidArmorLayer`   | 玩家、类人生物（例如僵尸、骷髅）、盔甲架                                       |
| `HUMANOID_LEGGINGS`     | `HumanoidArmorLayer`   | 玩家、类人生物（例如僵尸、骷髅）、盔甲架                                       |
| `WINGS`                 | `WingsLayer`           | 玩家、类人生物（例如僵尸、骷髅）、盔甲架                                       |
| `WOLF_BODY`             | `WolfArmorLayer`       | 狼                                                             |
| `HORSE_BODY`            | `HorseArmorLayer`      | 马                                                             |
| `LLAMA_BODY`            | `LlamaDecorLayer`      | 羊驼、行商羊鹿                                                        |
| `PIG_SADDLE`            | `SimpleEquipmentLayer` | 猪                                                             |
| `STRIDER_SADDLE`        | `SimpleEquipmentLayer` | 炽足兽                                                           |
| `CAMEL_SADDLE`          | `SimpleEquipmentLayer` | 骆驼                                                            |
| `HORSE_SADDLE`          | `SimpleEquipmentLayer` | 马                                                             |
| `DONKEY_SADDLE`         | `SimpleEquipmentLayer` | 驴                                                             |
| `MULE_SADDLE`           | `SimpleEquipmentLayer` | 骡                                                             |
| `ZOMBIE_HORSE_SADDLE`   | `SimpleEquipmentLayer` | 僵尸马                                                           |
| `SKELETON_HORSE_SADDLE` | `SimpleEquipmentLayer` | 骷髅马                                                           |
| `HAPPY_GHAST_BODY`      | `SimpleEquipmentLayer` | 快乐的恶魂                                                        |

`EquipmentLayerRenderer` 只有一个方法来提交装备图层进行渲染：`renderLayers`。

``` java
// 在某个渲染方法中，其中 EquipmentLayerRenderer equipmentLayerRenderer 可用
this.equipmentLayerRenderer.renderLayers(
    // 要渲染的图层类型
    EquipmentClientInfo.LayerType.HUMANOID,
    // 代表 EquipmentClientInfo JSON 的资源键
    // 这将在 `EQUIPPABLE` 数据组件中通过 `assetId` 设置
    stack.get(DataComponents.EQUIPPABLE).assetId().orElseThrow(),
    // 要应用装备信息的模型
    // 这些通常是与实体模型分开的单独模型
    // 并且是链接到 LayerDefinition 的单独 ModelLayers
    model,
    // 代表作为模型渲染的物品的物品堆叠
    // 这仅用于获取可染色、闪烁和盔甲装饰信息
    stack,
    // 用于在正确位置渲染模型的姿势堆叠
    poseStack,
    // 将模型数据提交到的收集器
    collector,
    // 打包的光照坐标
    lightCoords,
    // 当图层中某个图层的 use_player_texture 为 true 时要渲染的纹理的绝对路径，如果不为 null
    // 代表 assets 文件夹内的绝对位置
    ResourceLocation.fromNamespaceAndPath("examplemod", "textures/other_texture.png"),
    // 模型轮廓的颜色
    // 仅当轮廓颜色不为 0 且 `RenderType` 具有或是轮廓类型时使用
    outlineColor,
    // 提交图层和装饰的起始顺序优先级，每次提交模型时递增
    // 默认情况下，此为 1
    order
);
```

[item]: index.md
[datacomponents]: datacomponents.md
[enchantment]: ../resources/server/enchantments/index.md#enchantment-costs-and-levels
[livingentity]: ../entities/livingentity.md
[registering]: ../concepts/registries.md#methods-for-registering
[rendering]: #equipment-rendering
[respack]: ../resources/index.md#assets
[tag]: ../resources/server/tags.md
[tinting]: ../resources/client/models/index.md#tinting