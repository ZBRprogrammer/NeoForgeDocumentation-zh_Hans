import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 客户端物品(Client Items)

客户端物品是`ItemStack`应如何在游戏内提交渲染的代码内表示，指定给定状态下应使用什么模型。客户端物品位于[`assets`文件夹][assets]内的`items`子目录中，由`DataComponents#ITEM_MODEL`内的相对位置指定。默认情况下，这是对象的注册名称（例如，`minecraft:apple`默认位于`assets/minecraft/items/apple.json`）。

客户端物品存储在`ModelManager`内，可以通过`Minecraft.getInstance().modelManager`访问。然后，您可以通过其[`ResourceLocation`][rl]调用`ModelManager#getItemModel`或`getItemProperties`来获取客户端物品信息。

:::warning
这些不要与游戏中实际烘焙和渲染的[模型][models]混淆。
:::

## 概述(Overview)

客户端物品的JSON可以分为两部分：由`model`定义的模型；以及由`properties`定义的属性。`model`负责定义在给定上下文中提交`ItemStack`进行渲染时要使用的模型JSON。另一方面，`properties`负责渲染器使用的设置。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    // 定义提交渲染的模型
    "model": {
        "type": "minecraft:model",
        // 指向相对于'models'目录的模型JSON
        // 位于'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item"
    },
    // 定义渲染过程中使用的一些设置
    "properties": {
        // 当为false时，禁用物品交换时将物品提升到其正常位置的动画
        "hand_animation_on_swap": false,
        // 当为true时，允许模型在GUI中渲染到其定义的槽边界之外（在GuiItemRenderState#bounds中定义）
        // 而不是被裁剪
        "oversized_in_gui": false
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.register(
        EXAMPLE_ITEM.get(),
        new ClientItem(
            // 定义提交渲染的模型
            new BlockModelWrapper.Unbaked(
                // 指向相对于'models'目录的模型JSON
                // 位于'assets/examplemod/models/item/example_item.json'
                ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                Collections.emptyList()
            ),
            // 定义渲染过程中使用的一些设置
            new ClientItem.Properties(
                // 当为false时，禁用物品交换时将物品提升到其正常位置的动画
                false
            )
        )
    );
}
```

</TabItem>
</Tabs>

有关物品模型如何提交渲染的更多信息，请参见[下文][itemmodel]。

## 基本模型(A Basic Model)

`model`内的`type`字段决定如何选择为物品提交渲染的模型。最简单的类型由`minecraft:model`（或`BlockModelWrapper`）处理，它在功能上定义要提交渲染的模型JSON，相对于`models`目录（例如`assets/<namespace>/models/<path>.json`）。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:model",
        // 指向相对于'models'目录的模型JSON
        // 位于'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item"
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new BlockModelWrapper.Unbaked(
            // 指向相对于'models'目录的模型JSON
            // 位于'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            Collections.emptyList()
        )
    );
}
```

</TabItem>
</Tabs>

### 着色(Tinting)

与大多数模型一样，客户端物品可以根据堆栈的属性更改指定纹理的颜色。因此，`minecraft:model`类型具有`tints`字段来定义要应用的不透明颜色。这些称为`ItemTintSource`，在`ItemTintSources`中定义。它们还有一个`type`字段来定义要使用的源。它们应用的`tintindex`由它们在列表中的索引指定。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:model",
        // 指向'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item",
        // 要应用的着色列表
        "tints": [
            {
                // 当tintindex: 0时
                "type": "minecraft:constant",
                // 0x00FF00（或纯绿色）
                "value": 65280
            },
            {
                // 当tintindex: 1时
                "type": "minecraft:dye",
                // 0x0000FF（或纯蓝色）
                // 仅当未设置`DataComponents#DYED_COLOR`时调用
                "default": 255
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new BlockModelWrapper.Unbaked(
            // 指向'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            // 要应用的着色列表
            List.of(
                // 当tintindex: 0时
                new Constant(
                    // 纯绿色
                    0x00FF00
                ),
                // 当tintindex: 1时
                new Dye(
                    // 纯蓝色
                    // 仅当未设置`DataComponents#DYED_COLOR`时调用
                    0x0000FF
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建您自己的`ItemTintSource`类似于任何其他基于编解码器的注册对象。您创建一个实现`ItemTintSource`的类，创建一个`MapCodec`来编码和解码对象，并通过[模组事件总线][modbus]上的`RegisterColorHandlersEvent.ItemTintSources`将编解码器注册到其注册表。`ItemTintSource`只包含一个方法`calculate`，它接受当前的`ItemStack`、堆栈所在的维度、持有堆栈的实体，并返回ARGB格式的不透明颜色，其中前8位为0xFF。

```java
public record DamageBar(int defaultColor) implements ItemTintSource {

    // 要注册的映射编解码器
    public static final MapCodec<DamageBar> MAP_CODEC = ExtraCodecs.RGB_COLOR_CODEC.fieldOf("default")
        .xmap(DamageBar::new, DamageBar::defaultColor);

    public DamageBar(int defaultColor) {
        // 确保传入的颜色是不透明的
        this.defaultColor = ARGB.opaque(defaultColor);
    }

    @Override
    public int calculate(ItemStack stack, @Nullable ClientLevel level, @Nullable LivingEntity entity) {
        return stack.isDamaged() ? ARGB.opaque(stack.getBarColor()) : defaultColor;
    }

    @Override
    public MapCodec<DamageBar> type() {
        return MAP_CODEC;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerItemTintSources(RegisterColorHandlersEvent.ItemTintSources event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "damage_bar"),
        // 映射编解码器
        DamageBar.MAP_CODEC
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:model",
        // 指向'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item",
        // 要应用的着色列表
        "tints": [
            {
                // 当tintindex: 0时
                "type": "examplemod:damage_bar",
                // 0x00FF00（或纯绿色）
                "default": 65280
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new BlockModelWrapper.Unbaked(
            // 指向'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            // 要应用的着色列表
            List.of(
                // 当tintindex: 0时
                new DamageBar(
                    // 纯绿色
                    0x00FF00
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 复合模型(Composite Models)

有时，您可能希望为单个物品注册多个模型。虽然这可以直接使用[复合模型加载器][composite]完成，但对于物品模型，有一个自定义的`minecraft:composite`类型，它接受要提交渲染的模型列表。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:composite",

        // 要提交渲染的模型
        // 将按它们在列表中出现的顺序绘制
        "models": [
            {
                "type": "minecraft:model",
                // 指向'assets/examplemod/models/item/example_item_1.json'
                "model": "examplemod:item/example_item_1"
            },
            {
                "type": "minecraft:model",
                // 指向'assets/examplemod/models/item/example_item_2.json'
                "model": "examplemod:item/example_item_2"
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new CompositeModel.Unbaked(
            // 要提交渲染的模型
            // 将按它们在列表中出现的顺序绘制
            List.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                    // 要应用的着色列表
                    Collections.emptyList()
                ),
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 属性模型(Property Models)

一些物品根据存储在其堆栈中的数据更改其状态（例如，拉弓、损坏的鞘翅、给定维度中的时钟等）。为了允许模型基于状态更改，物品模型可以指定要跟踪的属性，并根据该条件选择模型。属性模型有三种不同类型：范围分发、选择和条件。它们分别作为某个浮点数、开关情况和布尔值的表达式。

### 范围分发模型(Range Dispatch Models)

范围分发模型具有定义某个`RangeSelectItemModelProperty`的类型，以获取某个浮点数来切换模型。然后每个条目具有某个阈值，该阈值必须大于该阈值才能提交渲染。选择的模型是具有最接近阈值但不大于属性值的模型（例如，如果属性值为`4`，阈值为`3`和`5`，则将绘制与`3`关联的模型，如果值为`6`，则将绘制与`5`关联的模型）。可用的`RangeSelectItemModelProperty`可以在`RangeSelectItemModelProperties`中找到。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:range_dispatch",

        // 要使用的`RangeSelectItemModelProperty`
        "property": "minecraft:count",
        // 乘以计算出的属性值的标量
        // 如果计数为0.3，标量为0.2，则检查的阈值为0.3*0.2=0.06
        "scale": 1,
        "fallback": {
            // 如果没有阈值匹配，则使用的回退模型
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item.json'
            "model": "examplemod:item/example_item"
        },

        // `Count`定义的属性
        // 当为true时，使用其最大堆叠大小归一化计数
        "normalize": true,

        // 具有阈值信息的条目
        "entries": [
            {
                // 当计数是其当前最大堆叠大小的三分之一时
                "threshold": 0.33,
                "model": {
                    // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当计数是其当前最大堆叠大小的三分之二时
                "threshold": 0.66,
                "model": {
                    // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new RangeSelectItemModel.Unbaked(
            new Count(
                // 当为true时，使用其最大堆叠大小归一化计数
                true
            ),
            // 乘以计算出的属性值的标量
            // 如果计数为0.3，标量为0.2，则检查的阈值为0.3*0.2=0.06
            1,
            // 具有阈值信息的条目
            List.of(
                new RangeSelectItemModel.Entry(
                    // 当计数是其当前最大堆叠大小的三分之一时
                    0.33,
                    // 可以是任何未烘焙的模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_1.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                ),
                new RangeSelectItemModel.Entry(
                    // 当计数是其当前最大堆叠大小的三分之二时
                    0.66,
                    // 可以是任何未烘焙的模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_2.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                )
            ),
            // 如果没有阈值匹配，则使用的回退模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建您自己的`RangeSelectItemModelProperty`类似于任何其他基于编解码器的注册对象。您创建一个实现`RangeSelectItemModelProperty`的类，创建一个`MapCodec`来编码和解码对象，并通过[模组事件总线][modbus]上的`RegisterRangeSelectItemModelPropertyEvent`将编解码器注册到其注册表。`RangeSelectItemModelProperty`只包含一个方法`get`，它接受当前的`ItemStack`、堆栈所在的维度、持有堆栈的实体以及某个种子值，返回一个由范围分发模型解释的任意浮点数。

```java
public record AppliedEnchantments() implements RangeSelectItemModelProperty {

    public static final MapCodec<AppliedEnchantments> MAP_CODEC = MapCodec.unit(new AppliedEnchantments());

    @Override
    public float get(ItemStack stack, @Nullable ClientLevel level, @Nullable ItemOwner owner, int seed) {
        return (float) stack.getTagEnchantments().size();
    }

    @Override
    public MapCodec<AppliedEnchantments> type() {
        return MAP_CODEC;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerRangeProperties(RegisterRangeSelectItemModelPropertyEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "applied_enchantments"),
        // 映射编解码器
        AppliedEnchantments.MAP_CODEC
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:range_dispatch",

        // 要使用的`RangeSelectItemModelProperty`
        "property": "examplemod:applied_enchantments",
        // 乘以计算出的属性值的标量
        // 如果计数为0.3，标量为0.2，则检查的阈值为0.3*0.2=0.06
        "scale": 0.5,
        "fallback": {
            // 如果没有阈值匹配，则使用的回退模型
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item.json'
            "model": "examplemod:item/example_item"
        },

        // 具有阈值信息的条目
        "entries": [
            {
                // 当至少存在一个附魔时
                // 由于1 * 标量0.5 = 0.5
                "threshold": 0.5,
                "model": {
                    // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当至少存在两个附魔时
                // 由于2 * 标量0.5 = 1
                "threshold": 1,
                "model": {
                    // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new RangeSelectItemModel.Unbaked(
            new AppliedEnchantments(),
            // 乘以计算出的属性值的标量
            // 如果计数为0.3，标量为0.2，则检查的阈值为0.3*0.2=0.06
            0.5,
            // 具有阈值信息的条目
            List.of(
                new RangeSelectItemModel.Entry(
                    // 当至少存在一个附魔时
                    0.5,
                    // 可以是任何未烘焙的模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_1.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                ),
                new RangeSelectItemModel.Entry(
                    // 当至少存在两个附魔时
                    1,
                    // 可以是任何未烘焙的模型类型
                    new BlockModelWrapper.Unbaked(
                        // 指向'assets/examplemod/models/item/example_item_2.json'
                        ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                        // 要应用的着色列表
                        Collections.emptyList()
                    )
                )
            ),
            // 如果没有阈值匹配，则使用的回退模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

### 选择模型(Select Models)

选择模型类似于范围分发模型，但它们基于`SelectItemModelProperty`定义的某个值进行切换，就像枚举的switch语句一样。选择的模型是与开关情况中值完全匹配的属性。可用的`SelectItemModelProperty`可以在`SelectItemModelProperties`中找到。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:select",

        // 要使用的`SelectItemModelProperty`
        "property": "minecraft:display_context",
        "fallback": {
            // 如果没有情况匹配，则使用的回退模型
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            "model": "examplemod:item/example_item"
        },

        // 基于可选择的属性的开关情况
        "cases": [
            {
                // 当显示上下文是`ItemDisplayContext#GUI`时
                "when": "gui",
                "model": {
                    // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当显示上下文是`ItemDisplayContext#FIRST_PERSON_RIGHT_HAND`时
                "when": "firstperson_righthand",
                "model": {
                     // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SelectItemModel.Unbaked(
            new SelectItemModel.UnbakedSwitch(
                // 要使用的`SelectItemModelProperty`
                new DisplayContext(),
                // 基于可选择属性的开关情况
                List.of(
                    new SelectItemModel.SwitchCase(
                        // 此模型匹配的情况列表
                        List.of(ItemDisplayContext.GUI),
                        // 可以是任何未烘焙的模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_1.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    ),
                    new SelectItemModel.SwitchCase(
                        // 此模型匹配的情况列表
                        List.of(ItemDisplayContext.FIRST_PERSON_RIGHT_HAND),
                        // 可以是任何未烘焙的模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_2.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    )
                )
            ),
            // 如果没有情况匹配，则使用的回退模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建您自己的`SelectItemModelProperty`类似于基于编解码器的注册对象。您创建一个实现`SelectItemModelProperty<T>`的类，创建一个`Codec`来序列化和反序列化属性值，创建一个`MapCodec`来编码和解码对象，并通过[模组事件总线][modbus]上的`RegisterSelectItemModelPropertyEvent`将编解码器注册到其注册表。`SelectItemModelProperty`有一个泛型`T`，表示要切换的值。它只包含一个方法`get`，它接受当前的`ItemStack`、堆栈所在的维度、持有堆栈的实体、某个种子值和物品的显示上下文，返回一个由选择模型解释的任意`T`。

```java
// 选择属性类
public record StackRarity() implements SelectItemModelProperty<Rarity> {

    // 包含相关编解码器的要注册的对象
    public static final SelectItemModelProperty.Type<StackRarity, Rarity> TYPE = SelectItemModelProperty.Type.create(
        // 此属性的映射编解码器
        MapCodec.unit(new StackRarity()),
        // 被选择的对象的编解码器
        // 用于序列化情况条目（"when": <property value>）
        Rarity.CODEC
    );

    @Nullable
    @Override
    public Rarity get(ItemStack stack, @Nullable ClientLevel level, @Nullable LivingEntity entity, int seed, ItemDisplayContext displayContext) {
        // 当null时，使用回退模型
        return stack.get(DataComponents.RARITY);
    }

    @Override
    public SelectItemModelProperty.Type<StackRarity, Rarity> type() {
        return TYPE;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerSelectProperties(RegisterSelectItemModelPropertyEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "rarity"),
        // 属性类型
        StackRarity.TYPE
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:select",

        // 要使用的`SelectItemModelProperty`
        "property": "examplemod:rarity",
        "fallback": {
            // 如果没有情况匹配，则使用的回退模型
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            "model": "examplemod:item/example_item"
        },

        // 基于可选择的属性的开关情况
        "cases": [
            {
                // 当稀有度是`Rarity#UNCOMMON`时
                "when": "uncommon",
                "model": {
                    // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_1.json'
                    "model": "examplemod:item/example_item_1"
                }
            },
            {
                // 当稀有度是`Rarity#RARE`时
                "when": "rare",
                "model": {
                     // 可以是任何未烘焙的模型类型
                    "type": "minecraft:model",
                    // 指向'assets/examplemod/models/item/example_item_2.json'
                    "model": "examplemod:item/example_item_2"
                }
            }
        ]
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SelectItemModel.Unbaked(
            new SelectItemModel.UnbakedSwitch(
                // 要使用的`SelectItemModelProperty`
                new StackRarity(),
                // 基于可选择属性的开关情况
                List.of(
                    new SelectItemModel.SwitchCase(
                        // 此模型匹配的情况列表
                        List.of(Rarity.UNCOMMON),
                        // 可以是任何未烘焙的模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_1.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    ),
                    new SelectItemModel.SwitchCase(
                        // 此模型匹配的情况列表
                        List.of(Rarity.RARE),
                        // 可以是任何未烘焙的模型类型
                        new BlockModelWrapper.Unbaked(
                            // 指向'assets/examplemod/models/item/example_item_2.json'
                            ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                            // 要应用的着色列表
                            Collections.emptyList()
                        )
                    )
                )
            ),
            // 如果没有情况匹配，则使用的回退模型
            Optional.of(
                new BlockModelWrapper.Unbaked(
                    // 指向'assets/examplemod/models/item/example_item.json'
                    ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
                    // 要应用的着色列表
                    Collections.emptyList()
                )
            )
        )
    );
}
```

</TabItem>
</Tabs>

### 条件模型(Conditional Models)

条件模型是三种中最简单的。该类型定义某个`ConditionalItemModelProperty`来获取一个布尔值以切换模型。选择的模型基于返回的布尔值是true还是false。可用的`ConditionalItemModelProperty`可以在`ConditionalItemModelProperties`中找到。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:condition",

        // 要使用的`ConditionalItemModelProperty`
        "property": "minecraft:damaged",

        // 布尔结果是什么
        "on_true": {
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_1.json'
            "model": "examplemod:item/example_item_1"
            
        },
        "on_false": {
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_2.json'
            "model": "examplemod:item/example_item_2"
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new ConditionalItemModel.Unbaked(
            // 要检查的属性
            new Damaged(),
            // 当布尔值为true时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_1.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                // 要应用的着色列表
                Collections.emptyList()
            ),
            // 当布尔值为false时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_2.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                // 要应用的着色列表
                Collections.emptyList()
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建您自己的`ConditionalItemModelProperty`类似于任何其他基于编解码器的注册对象。您创建一个实现`ConditionalItemModelProperty`的类，创建一个`MapCodec`来编码和解码对象，并通过[模组事件总线][modbus]上的`RegisterConditionalItemModelPropertyEvent`将编解码器注册到其注册表。`RangeSelectItemModelProperty`只包含一个方法`get`，它接受当前的`ItemStack`、堆栈所在的维度、持有堆栈的实体、某个种子值和物品的显示上下文，返回一个由条件模型解释的任意布尔值（`on_true`或`on_false`）。

```java
public record BarVisible() implements ConditionalItemModelProperty {

    public static final MapCodec<BarVisible> MAP_CODEC =  MapCodec.unit(new BarVisible());

    @Override
    public boolean get(ItemStack stack, @Nullable ClientLevel level, @Nullable LivingEntity entity, int seed, ItemDisplayContext context) {
        return stack.isBarVisible();
    }

    @Override
    public MapCodec<BarVisible> type() {
        return MAP_CODEC;
    }
}

// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerConditionalProperties(RegisterConditionalItemModelPropertyEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "bar_visible"),
        // 映射编解码器
        BarVisible.MAP_CODEC
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:condition",

        // 要使用的`ConditionalItemModelProperty`
        "property": "examplemod:bar_visible",

        // 布尔结果是什么
        "on_true": {
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_1.json'
            "model": "examplemod:item/example_item_1"
            
        },
        "on_false": {
            // 可以是任何未烘焙的模型类型
            "type": "minecraft:model",
            // 指向'assets/examplemod/models/item/example_item_2.json'
            "model": "examplemod:item/example_item_2"
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new ConditionalItemModel.Unbaked(
            // 要检查的属性
            new BarVisible(),
            // 当布尔值为true时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_1.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_1"),
                // 要应用的着色列表
                Collections.emptyList()
            ),
            // 当布尔值为false时
            new BlockModelWrapper.Unbaked(
                // 指向'assets/examplemod/models/item/example_item_2.json'
                ResourceLocation.fromNamespaceAndPath("examplemod", "item/example_item_2"),
                // 要应用的着色列表
                Collections.emptyList()
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 特殊模型(Special Models)

并非所有模型都可以使用基本模型JSON表示。有些模型可以具有动态组件，或者使用为[`BlockEntityRenderer`][ber]创建的现有`Model`。在这些情况下，有一个特殊的模型类型，允许用户指定要提交渲染的[特性]。这些称为`SpecialModelRenderer`，在`SpecialModelRenderers`内定义。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:special",

        // 从中读取粒子纹理和显示变换的父模型
        // 指向'assets/minecraft/models/item/template_skull.json'
        "base": "minecraft:item/template_skull",
        "model": {
            // 要使用的特殊模型渲染器
            "type": "minecraft:head",

            // 由`SkullSpecialRenderer.Unbaked`定义的属性
            // 头颅方块的类型
            "kind": "wither_skeleton",
            // 渲染头部时使用的纹理
            // 指向'assets/examplemod/textures/entity/heads/skeleton_override.png'
            "texture": "examplemod:heads/skeleton_override",
            // 用于动画头部模型的动画浮点数
            "animation": 0.5
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SpecialModelWrapper.Unbaked(
            // 从中读取粒子纹理和显示变换的父模型
            // 指向'assets/minecraft/models/item/template_skull.json'
            ResourceLocation.fromNamespaceAndPath("minecraft", "item/template_skull"),
            // 要使用的特殊模型渲染器
            new SkullSpecialRenderer.Unbaked(
                // 头颅方块的类型
                SkullBlock.Types.WITHER_SKELETON,
                // 渲染头部时使用的纹理
                // 指向'assets/examplemod/textures/entity/heads/skeleton_override.png'
                Optional.of(
                    ResourceLocation.fromNamespaceAndPath("examplemod", "heads/skeleton_override")
                ),
                // 用于动画头部模型的动画浮点数
                0.5f
            )
        )
    );
}
```

</TabItem>
</Tabs>

创建您自己的`SpecialModelRenderer`分为三部分：用于提交渲染[特性]的`SpecialModelRenderer`实例、用于读取和写入JSON的`SpecialModelRenderer.Unbaked`实例，以及注册渲染器以在作为物品时或在必要时在作为方块时使用。

首先，是`SpecialModelRenderer`。这类似于任何其他渲染器类（例如方块实体渲染器、实体渲染器）。它应接受提交过程中使用的静态数据（例如，`Model`子类、纹理的`Material`等）。有两个方法需要注意。首先是`extractArgument`。这用于通过仅从`ItemStack`提供必要的数据来限制`submit`方法可用的数据量。

:::note
如果您不知道可能需要什么数据，您可以只让它返回相关的`ItemStack`。如果不需要来自堆栈的数据，您可以改用`NoDataSpecialModelRenderer`，它会为您实现此方法。
:::

接下来是`submit`方法。这接受从`extractArgument`返回的值、物品的显示上下文、姿势堆栈、用于提交所需特性的收集器、打包光照、叠加纹理、堆栈是否箔化（例如附魔）和轮廓颜色。所有特性提交都应在此方法中进行。

```java
public record ExampleSpecialRenderer(MaterialSet materialSet, Model.Simple model, Material material) implements SpecialModelRenderer<Boolean> {

    @Nullable
    public Boolean extractArgument(ItemStack stack) {
        // 提取要使用的数据
        return stack.isBarVisible();
    }

    // 提交模型的特性
    @Override
    public void submit(ItemDisplayContext displayContext, PoseStack poseStack, SubmitNodeCollector collector, int lightCoords, int overlayCoords, boolean hasFoil, int outlineColor) {
        collector.submitModel(
            this.model, Unit.INSTANCE,
            poseStack, this.material.renderType(barVisible ? RenderType::entityCutout : RenderType::entitySolid),
            lightCoords, overlayCoords, -1, this.materialSet.get(this.material), outlineColor, null
        );
    }
}
```

接下来是`SpecialModelRenderer.Unbaked`实例。这应包含可以从文件中读取以确定要传递给特殊渲染器的数据。这还包含两个方法：`bake`，用于构建特殊渲染器实例；和`type`，它定义用于编码/解码到文件的`MapCodec`。

```java
public record ExampleSpecialRenderer(MaterialSet materialSet, Model.Simple model, Material material) implements SpecialModelRenderer<Boolean> {

    // ...

    public record Unbaked(ResourceLocation texture) implements SpecialModelRenderer.Unbaked {

        public static final MapCodec<ExampleSpecialRenderer.Unbaked> MAP_CODEC = ResourceLocation.CODEC.fieldOf("texture")
            .xmap(ExampleSpecialRenderer.Unbaked::new, ExampleSpecialRenderer.Unbaked::texture);

        @Override
        public MapCodec<ExampleSpecialRenderer.Unbaked> type() {
            return MAP_CODEC;
        }

        @Override
        public SpecialModelRenderer<?> bake(SpecialModelRenderer.BakingContext ctx) {
            // 将资源位置解析为绝对路径
            ResourceLocation textureLoc = this.texture.withPath(path -> "textures/entity/" + path + ".png");

            // 获取要渲染的模型和材质
            return new ExampleSpecialRenderer(ctx.materials(), ...);
        }
    }
}
```

最后，我们将对象注册到其必要的位置。对于客户端物品，这是通过[模组事件总线][modbus]上的`RegisterSpecialModelRendererEvent`完成的。如果特殊渲染器也应作为`BlockEntityRenderer`的一部分使用，例如在某些类似物品的上下文中渲染时（例如，末影人手持方块），则应通过[模组事件总线][modbus]上的`RegisterSpecialBlockModelRendererEvent`为方块注册一个`Unbaked`版本。

```java
// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerSpecialRenderers(RegisterSpecialModelRendererEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "example_special"),
        // 映射编解码器
        ExampleSpecialRenderer.Unbaked.MAP_CODEC
    )
}

// 用于在类似物品的上下文中渲染方块
// 假设某个DeferredBlock<ExampleBlock> EXAMPLE_BLOCK
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerSpecialBlockRenderers(RegisterSpecialBlockModelRendererEvent event) {
    event.register(
        // 要渲染的方块
        EXAMPLE_BLOCK.get()
        // 要使用的未烘焙实例
        new ExampleSpecialRenderer.Unbaked(ResourceLocation.fromNamespaceAndPath("examplemod", "entity/example_special"))
    )
}
```

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "minecraft:special",

        // 从中读取粒子纹理和显示变换的父模型
        // 指向'assets/minecraft/models/item/template_skull.json'
        "base": "minecraft:item/template_skull",
        "model": {
            // 要使用的特殊模型渲染器
            "type": "examplemod:example_special",

            // 由`ExampleSpecialRenderer.Unbaked`定义的属性
            // 要使用的纹理
            // 指向'assets/examplemod/textures/entity/example/example_texture.png'
            "texture": "examplemod:example/example_texture"
        }
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new SpecialModelWrapper.Unbaked(
            // 从中读取粒子纹理和显示变换的父模型
            // 指向'assets/minecraft/models/item/template_skull.json'
            ResourceLocation.fromNamespaceAndPath("minecraft", "item/template_skull"),
            // 要使用的特殊模型渲染器
            new ExampleSpecialRenderer.Unbaked(
                // 要使用的纹理
                // 指向'assets/examplemod/textures/entity/example/example_texture.png'
                ResourceLocation.fromNamespaceAndPath("examplemod", "example/example_texture")
            )
        )
    );
}
```

</TabItem>
</Tabs>

## 动态流体容器(Dynamic Fluid Container)

NeoForge添加了一个构造动态流体容器的物品模型，能够在运行时重新纹理化以匹配所包含的流体。

:::note
为了使流体着色应用于流体纹理，相关物品必须附加`Capabilities.FluidHandler.ITEM`。如果您的物品不直接使用`BucketItem`（也不是子类型），那么您需要[将能力注册到您的物品][capability]。
:::

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "neoforge:fluid_container",

        // 用于构造容器的纹理
        // 这些是相对于方块图集的引用，因此它们相对于`textures`目录
        "textures": {
            // 设置模型粒子精灵
            // 如果未设置，使用第一个不为null的纹理：
            // - 流体静止纹理
            // - 容器基础纹理
            // - 容器覆盖纹理，如果不作为遮罩使用
            // 指向'assets/minecraft/textures/item/bucket.png'
            "particle": "minecraft:item/bucket",
            // 设置在第一层使用的纹理，通常是流体的容器
            // 如果未设置，该层将不会添加
            // 指向'assets/minecraft/textures/item/bucket.png'
            "base": "minecraft:item/bucket",
            // 设置用作静止流体纹理遮罩的纹理
            // 看到流体的区域应为纯白色
            // 如果未设置或流体为空，则不绘制该层
            // 指向'assets/neoforge/textures/item/mask/bucket_fluid.png'
            "fluid": "neoforge:item/mask/bucket_fluid",
            // 设置用作以下之一的纹理
            // - 当'cover_is_mask'为false时的叠加纹理
            // - 当'cover_is_mask'为true时应用于基础纹理的遮罩（应为纯白色以看到）
            // 如果未设置或在'cover_is_mask'为true时未设置基础纹理，则不绘制该层
            // 指向'assets/neoforge/textures/item/mask/bucket_fluid_cover.png'
            "cover": "neoforge:item/mask/bucket_fluid_cover",
        },

        // 当为true时，对于密度为负或零的流体，将模型旋转180度
        // 默认为false
        "flip_gas": true,
        // 当为true时，使用覆盖纹理作为基础纹理的遮罩
        // 默认为true
        "cover_is_mask": true,
        // 当为true时，对于光照等级大于零的流体，将其流体纹理层的光照图设置为其最大值
        // 默认为true
        "apply_fluid_luminosity": false
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<ExampleFluidContainerItem> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new DynamicFluidContainerModel.Unbaked(
            // 用于构造容器的纹理
            // 这些是相对于方块图集的引用，因此它们相对于`textures`目录
            new DynamicFluidContainerModel.Textures(
                // 设置模型粒子精灵
                // 如果未设置，使用第一个不为null的纹理：
                // - 流体静止纹理
                // - 容器基础纹理
                // - 容器覆盖纹理，如果不作为遮罩使用
                // 指向'assets/minecraft/textures/item/bucket.png'
                Optional.of(ResourceLocation.withDefaultNamespace("item/bucket")),
                // 设置在第一层使用的纹理，通常是流体的容器
                // 如果未设置，该层将不会添加
                // 指向'assets/minecraft/textures/item/bucket.png'
                Optional.of(ResourceLocation.withDefaultNamespace("item/bucket")),
                // 设置用作静止流体纹理遮罩的纹理
                // 看到流体的区域应为纯白色
                // 如果未设置或流体为空，则不渲染该层
                // 指向'assets/neoforge/textures/item/mask/bucket_fluid.png'
                Optional.of(ResourceLocation.fromNamespaceAndPath("neoforge", "item/mask/bucket_fluid")),
                // 设置用作以下之一的纹理
                // - 当'cover_is_mask'为false时的叠加纹理
                // - 当'cover_is_mask'为true时应用于基础纹理的遮罩（应为纯白色以看到）
                // 如果未设置或在'cover_is_mask'为true时未设置基础纹理，则不渲染该层
                // 指向'assets/neoforge/textures/item/mask/bucket_fluid_cover.png'
                Optional.of(ResourceLocation.fromNamespaceAndPath("neoforge", "item/mask/bucket_fluid_cover"))
            ),
            // 当为true时，将模型旋转180度
            // 默认为false
            true,
            // 当为true时，使用覆盖纹理作为基础纹理的遮罩
            // 默认为true
            true,
            // 当为true时，将其流体纹理层的光照图设置为其最大值
            // 默认为true
            false
        )
    );
}
```

</TabItem>
</Tabs>

## 手动提交物品进行渲染(Manually Submitting an Item for Rendering)

如果您需要提交物品[特性][features]，例如在某些`BlockEntityRenderer`或`EntityRenderer`中，可以通过三个步骤实现。首先，相关的渲染器创建一个`ItemStackRenderState`来保存堆栈的状态。然后，`ItemModelResolver`使用其方法之一更新`ItemStackRenderState`，将状态更新为当前正在提交的物品。最后，通过`ItemStackRenderState#submit`提交物品。

`ItemStackRenderState`跟踪用于绘制的数据。每个“模型”都有自己的`ItemStackRenderState.LayerRenderState`，其中包含要渲染的`BakedQuad`s，以及其渲染类型、箔化状态、着色信息、动画标志、范围和使用的任何特殊渲染器。使用`newLayer`方法创建层，使用`clear`方法清除以供渲染。如果使用预定义数量的层，则使用`ensureCapacity`来确保有必要数量的`LayerRenderStates`以正确渲染。

:::note
[屏幕][screens]使用子类`TrackingItemStackRenderState`来保存模型身份元素，以便在帧之间缓存渲染状态。
:::

`ItemModelResolver`负责更新`ItemStackRenderState`。这可以通过用于由生物实体持有的物品的`updateForLiving`、用于由其他类型实体持有的物品的`updateForNonLiving`以及用于所有其他情况的`updateForTopItem`完成。这些方法接受渲染状态、要渲染的堆栈和当前显示上下文。其他参数更新有关持有手、维度、物品所有者和种子值的信息。每个方法在调用从`DataComponents#ITEM_MODEL`获取的`ItemModel`上的`update`之前调用`ItemStackRenderState#clear`。如果您不在某些渲染器上下文（例如，`BlockEntityRenderer`、`EntityRenderer`）中，可以始终通过`Minecraft#getItemModelResolver`获取`ItemModelResolver`。

## 自定义物品模型定义(Custom Item Model Definitions)

创建您自己的`ItemModel`分为三部分：用于更新渲染状态的`ItemModel`实例、用于读取和写入JSON的`ItemModel.Unbaked`实例，以及注册以使用`ItemModel`。

:::warning
请确保检查您所需的物品模型是否无法使用上述现有系统创建。在大多数情况下，不需要创建自定义`ItemModel`。
:::

首先，是`ItemModel`。它负责更新`ItemStackRenderState`，以便正确绘制物品。它应接受提交过程中使用的静态数据（例如，`BakedQuad`s列表、属性信息等）。唯一的方法是`update`，它接受渲染状态、堆栈、模型解析器、显示上下文、维度和物品所有者以及某个种子值来更新`ItemStackRenderState`。`ItemStackRenderState`应该是唯一修改的参数，其余部分视为只读数据。

```java
public record RenderTypeModelWrapper(List<BakedQuad> quads, List<ItemTintSource> tints, ModelRenderProperties properties, RenderType type) implements ItemModel {

    // 更新渲染状态
    @Override
    public void update(ItemStackRenderState state, ItemStack stack, ItemModelResolver resolver, ItemDisplayContext displayContext, @Nullable ClientLevel level, @Nullable ItemOwner owner, int seed) {
        // 设置模型使用的身份
        state.appendModelIdentityElement(this);

        // 创建一个新层
        ItemStackRenderState.LayerRenderState layerState = state.newLayer();

        // 设置要使用的箔化
        if (stack.hasFoil()) {
            layerState.setFoilType(ItemStackRenderState.FoilType.STANDARD);
            state.appendModelIdentityElement(ItemStackRenderState.FoilType.STANDARD);
            layerState.setAnimated();
        }


        // 应用着色源
        int tintSize = this.tints.size();
        int[] tintLayers = layerState.prepareTintLayers(tintSize);

        for (int idx = 0; idx < tintSize; idx++) {
            int tintColor = this.tints.get(idx).calculate(stack, level, owner.asLivingEntity());
            tintLayers[idx] = tintColor;
            state.appendModelIdentityElement(tintColor);
        }

        // 计算模型的边界
        // 用于GUI渲染边界（当过大时）和物品实体上下浮动
        layerState.setExtents(BlockModelWrapper.computeExtents(this.quads));

        // 设置当前渲染类型
        layerState.setRenderType(this.type);

        // 设置其他常见模型属性
        this.properties.applyToLayer(layerState, displayContext);

        // 添加要提交的四边形
        layerState.prepareQuadList().addAll(this.quads);
    }
}
```

接下来是`ItemModel.Unbaked`实例。这应包含可以从文件中读取以确定要传递给物品模型的数据。这还包含两个方法：`bake`，用于构建`ItemModel`实例；和`type`，它定义用于编码/解码到文件的`MapCodec`。

```java
public record RenderTypeModelWrapper(List<BakedQuad> quads, List<ItemTintSource> tints, ModelRenderProperties properties, RenderType type) implements ItemModel {

    // ...

     public record Unbaked(ResourceLocation model, List<ItemTintSource> tints, RenderType type) implements ItemModel.Unbaked {
        // 为编解码器创建渲染类型映射
        private static final BiMap<String, RenderType> RENDER_TYPES = Util.make(HashBiMap.create(), map -> {
            map.put("translucent_item", Sheets.translucentItemSheet());
            map.put("cutout_block", Sheets.cutoutBlockSheet());
        });
        private static final Codec<RenderType> RENDER_TYPE_CODEC = ExtraCodecs.idResolverCodec(Codec.STRING, RENDER_TYPES::get, RENDER_TYPES.inverse()::get);

        // 要注册的映射编解码器
        public static final MapCodec<RenderTypeModelWrapper.Unbaked> MAP_CODEC = RecordCodecBuilder.mapCodec(instance ->
            instance.group(
                ResourceLocation.CODEC.fieldOf("model").forGetter(RenderTypeModelWrapper.Unbaked::model),
                ItemTintSources.CODEC.listOf().optionalFieldOf("tints", List.of()).forGetter(RenderTypeModelWrapper.Unbaked::tints)
                RENDER_TYPE_CODEC.fieldOf("render_type").forGetter(RenderTypeModelWrapper.Unbaked::type)
            )
            .apply(instance, RenderTypeModelWrapper.Unbaked::new)
        );

        @Override
        public void resolveDependencies(ResolvableModel.Resolver resolver) {
            // 标记此物品模型的所有依赖项
            resolver.markDependency(this.model);
        }

        @Override
        public ItemModel bake(ItemModel.BakingContext context) {
            // 获取烘焙的四边形并返回
            ModelBaker baker = context.blockModelBaker();
            ResolvedModel resolvedModel = baker.getModel(this.model);
            TextureSlots slots = resolvedModel.getTopTextureSlots();

            return new RenderTypeModelWrapper(
                resolvedModel.bakeTopGeometry(slots, baker, BlockModelRotation.X0_Y0).getAll(),
                this.tints,
                ModelRenderProperties.fromResolvedModel(baker, resolvedModel, slots),
                this.type
            );
        }

        @Override
        public MapCodec<RenderTypeModelWrapper.Unbaked> type() {
            return MAP_CODEC;
        }
    }
}
```

然后，我们通过[模组事件总线][modbus]上的`RegisterItemModelsEvent`注册映射编解码器。

```java
// 在某个事件处理类中
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerItemModels(RegisterItemModelsEvent event) {
    event.register(
        // 作为类型引用的名称
        ResourceLocation.fromNamespaceAndPath("examplemod", "render_type"),
        // 映射编解码器
        RenderTypeModelWrapper.Unbaked.MAP_CODEC
    )
}
```

最后，我们可以在JSON或作为数据生成过程的一部分中使用`ItemModel`。

<Tabs>
<TabItem value="json" label="JSON" default>

```json5
// 对于某个物品'examplemod:example_item'
// JSON位于'assets/examplemod/items/example_item.json'
{
    "model": {
        "type": "examplemod:render_type",
        // 指向'assets/examplemod/models/item/example_item.json'
        "model": "examplemod:item/example_item",
        // 要应用于模型纹理的任何着色
        "tints": []
        // 设置渲染时使用的渲染类型
        "render_type": "cutout_block"
    }
}
```

</TabItem>

<TabItem value="datagen" label="Datagen">

```java
// 假设有某个DeferredItem<Item> EXAMPLE_ITEM
// 在扩展的ModelProvider内
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    itemModels.itemModelOutput.accept(
        EXAMPLE_ITEM.get(),
        new RenderTypeModelWrapper.Unbaked(
            // 指向'assets/examplemod/models/item/example_item.json'
            ModelLocationUtils.getModelLocation(EXAMPLE_ITEM.get()),
            // 要应用于模型纹理的任何着色
            List.of(),
            // 设置渲染时使用的渲染类型
            Sheets.cutoutBlockSheet()
        )
    );
}
```

</TabItem>
</Tabs>

[assets]: ../../index.md#assets
[ber]: ../../../blockentities/ber.md
[capability]: ../../../inventories/capabilities.md#registering-capabilities
[composite]: modelloaders.md#composite-model
[features]: ../../../rendering/feature.md
[itemmodel]: #manually-rendering-an-item
[modbus]: ../../../concepts/events.md#event-buses
[models]: modelsystem.md
[rl]: ../../../misc/resourcelocation.md
[screens]: ../../../rendering/screens.md#items