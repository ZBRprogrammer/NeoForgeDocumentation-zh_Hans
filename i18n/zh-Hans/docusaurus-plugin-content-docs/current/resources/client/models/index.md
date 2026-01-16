# 模型(Models)

模型是JSON文件，用于确定方块或物品的视觉形状和纹理。模型由长方体元素组成，每个元素有自己的大小，然后为每个面分配纹理。

物品使用由其[客户端物品][citems]定义的关联模型，而方块使用[方块状态文件][bsfile]中的关联模型。这些位置相对于`models`目录，因此引用名称为`examplemod:item/example_model`的模型将由`assets/examplemod/models/item/example_model.json`处的JSON定义。
 
## 规范(Specification)

_另请参见：[Minecraft Wiki][mcwiki]上的[模型][mcwikimodel]_

模型是一个根标签中包含以下可选属性的JSON文件：

- `loader`：NeoForge添加。设置自定义模型加载器。有关更多信息，请参见[模型加载器][custommodelloader]。
- `parent`：设置父模型，以相对于`models`文件夹的[资源位置][rl]形式。所有父属性将被应用，然后由声明模型中设置的属性覆盖。常见的父级包括：
    - `minecraft:block/block`：所有方块模型的公共父级。
    - `minecraft:block/cube`：使用1x1x1立方体模型的所有模型的父级。
    - `minecraft:block/cube_all`：立方体模型的变体，在所有六个面上使用相同的纹理，例如圆石或木板。
    - `minecraft:block/cube_bottom_top`：立方体模型的变体，在所有四个水平面上使用相同的纹理，顶部和底部使用单独的纹理。常见的例子包括砂岩或錾制石英。
    - `minecraft:block/cube_column`：立方体模型的变体，具有侧面纹理以及底部和顶部纹理。例如包括原木，以及石英和紫珀柱。
    - `minecraft:block/cross`：使用两个具有相同纹理的平面的模型，一个顺时针旋转45°，另一个逆时针旋转45°，从上方看形成X形（因此得名）。例如包括大多数植物，例如草、树苗和花。
    - `minecraft:item/generated`：经典2D平面物品模型的父级。游戏中大多数物品使用此父级。忽略`elements`块，因为其四边形从纹理生成。
    - `minecraft:item/handheld`：看起来实际上由玩家持有的2D平面物品模型的父级。主要由工具使用。`item/generated`的子模型，这导致它也会忽略`elements`块。
    - 方块物品通常（但不总是）使用其对应的方块模型作为其[物品模型][itemmodels]。例如，圆石客户端物品使用`minecraft:block/cobblestone`模型。
- `ambientocclusion`：是否启用[环境光遮蔽][ao]。仅对方块模型有效。默认为`true`。如果您的自定义方块模型有奇怪的着色，请尝试将其设置为`false`。
- `render_type`：NeoForge添加。设置要使用的渲染类型组。有关更多信息，请参见[渲染类型组][rendertype]。
- `gui_light`：可以是`"front"`或`"side"`。如果为`"front"`，光照将来自前面，适用于平面2D模型。如果为`"side"`，光照将来自侧面，适用于3D模型（尤其是方块模型）。默认为`"side"`。仅对物品模型有效。
- `textures`：一个子对象，将名称（称为纹理变量）映射到[纹理位置][textures]。纹理变量然后可以在[元素]中使用。它们也可以在元素中指定，但留空以便子模型指定它们。
    - 方块模型应另外指定`particle`纹理。此纹理在方块掉落、跑过或破坏时使用。
    - 物品模型也可以使用层纹理，命名为`layer0`、`layer1`等，其中索引较高的层渲染在索引较低的层之上（例如，`layer1`将渲染在`layer0`之上）。仅当父级为`item/generated`时才有效，并且最多只适用于5层（`layer0`到`layer4`）。
- `elements`：长方体[元素]的列表。
- `display`：一个子对象，保存不同[视角]的不同显示选项，有关可能的键，请参见链接文章。仅对物品模型有效，但通常在方块模型中指定，以便物品模型可以继承显示选项。每个视角都是一个可选的子对象，可能包含以下选项，按该顺序应用：
    - `translation`：模型的平移，指定为`[x, y, z]`。
    - `rotation`：模型的旋转，指定为`[x, y, z]`。
    - `scale`：模型的缩放，指定为`[x, y, z]`。
    - `right_rotation`：NeoForge添加。缩放后应用的第二个旋转，指定为`[x, y, z]`。
- `transform`：参见[根变换][roottransforms]。

:::tip
如果您在指定某些内容时遇到困难，请查看执行类似操作的原版模型。
:::

### 渲染类型组(Render Type Groups)

使用可选的NeoForge添加的`render_type`字段，您可以为模型设置`RenderTypeGroup`。渲染类型组由两部分组成：一个`ChunkSectionLayer`用于模型作为方块时应该如何渲染，以及一个`RenderType`用于模型作为物品时应该如何渲染。如果未设置此字段（所有原版模型都是这种情况），游戏将回退到`ItemBlockRenderTypes`中硬编码的层和渲染类型。如果`ItemBlockRenderTypes`不包含该层或渲染类型，它将回退到方块的`ChunkSectionLayer#SOLID`和物品的`Sheets#TRANSLUCENT_ITEM_CULL_BLOCK_SHEET`。原版和NeoForge公开了以下渲染类型组：

- `minecraft:solid`：用于完全实心的模型，例如石头。
- `minecraft:cutout`：用于任何像素要么完全实心要么完全透明的模型，即具有完全或不透明度的模型，例如玻璃。
- `minecraft:cutout_mipped`：`minecraft:cutout`的变体，将在远距离缩放纹理以避免视觉伪影（[mipmapping]）。不将mipmapping应用于物品渲染，因为物品上通常不需要且可能导致伪影。例如树叶使用此类型。
- `minecraft:cutout_mipped_all`：`minecraft:cutout_mipped`的变体，也将mipmapping应用于物品模型。
- `minecraft:translucent`：用于任何像素可能部分透明的模型，例如染色玻璃。
- `minecraft:tripwire`：用于具有渲染到天气目标的特殊要求的模型，即绊线。
- `neoforge:item_unlit`：NeoForge添加。应由作为物品渲染时不考虑光照方向的模型使用。

选择正确的渲染类型组在某种程度上是性能问题。实心渲染比切除渲染快，切除渲染比半透明渲染快。因此，您应该为您的用例指定“最严格”的渲染类型，因为它也将是最快的。

如果您愿意，也可以添加自己的渲染类型组。为此，订阅[模组总线][modbus]上的[事件]`RegisterNamedRenderTypesEvent`并`#register`您的渲染类型组。`#register`有三个参数：

- 渲染类型组的名称。应为以您的模组ID为前缀的`ResourceLocation`。
- 区块截面层。可以使用任何`ChunkSectionLayer`。
- 实体渲染类型。必须是具有`DefaultVertexFormat.NEW_ENTITY`顶点格式的渲染类型。

### 元素(Elements)

元素是长方体对象的JSON表示。它具有以下属性：

- `from`：长方体起始角的坐标，指定为`[x, y, z]`。以1/16方块单位指定。例如，`[0, 0, 0]`将是“左下”角，`[8, 8, 8]`将是中心，`[16, 16, 16]`将是方块的“右上”角。
- `to`：长方体结束角的坐标，指定为`[x, y, z]`。与`from`一样，以1/16方块单位指定。

:::tip
`from`和`to`中的值受Minecraft限制在范围`[-16, 32]`内。然而，强烈建议不要超过`[0, 16]`，因为这会导致光照和/或剔除问题。
:::

- `neoforge_data`：参见[额外面数据][extrafacedata]。
- `faces`：一个对象，包含最多6个面的数据，分别命名为`north`、`south`、`east`、`west`、`up`和`down`。每个面都有以下数据：
    - `uv`：面的uv，指定为`[u1, v1, u2, v2]`，其中`u1, v1`是左上uv坐标，`u2, v2`是右下uv坐标。
    - `texture`：用于面的纹理。必须是前缀为`#`的纹理变量。例如，如果您的模型有一个名为`wood`的纹理，您将使用`#wood`来引用该纹理。技术上可选，如果缺失将使用缺失纹理。
    - `rotation`：可选。将纹理顺时针旋转90、180或270度。
    - `cullface`：可选。告诉渲染引擎当指定方向有完整方块接触时跳过渲染该面。方向可以是`north`、`south`、`east`、`west`、`up`或`down`。
    - `tintindex`：可选。指定可由颜色处理器使用的色调索引，有关更多信息，请参见[着色][tinting]。默认为-1，表示无着色。
    - `neoforge_data`：参见[额外面数据][extrafacedata]。

此外，它可以指定以下可选属性：

- `shade`：仅用于方块模型。可选。此元素的面是否应具有方向相关的着色。默认为true。
- `rotation`：对象的旋转，指定为包含以下数据的子对象：
    - `angle`：旋转角度，以度为单位。可以为-45到45，步长为22.5度。
    - `axis`：要围绕旋转的轴。目前不可能围绕多个轴旋转对象。
    - `origin`：可选。要围绕旋转的原点，指定为`[x, y, z]`。请注意，这些是绝对值，即它们不相对于立方体的位置。如果未指定，将使用`[0, 0, 0]`。

#### 额外面数据(Extra Face Data)

额外面数据（`neoforge_data`）可以应用于元素和元素的单个面。在所有可用的上下文中都是可选的。如果同时指定了元素级和面级额外面数据，则面级数据将覆盖元素级数据。额外数据可以指定以下数据：

- `color`：用给定颜色为面着色。必须是ARGB值。可以指定为字符串或十进制整数（JSON不支持十六进制字面量）。默认为`0xFFFFFFFF`。如果颜色值是常量，这可以用作着色的替代方案。
- `block_light`：覆盖此面使用的方块光照值。默认为0。
- `sky_light`：覆盖此面使用的天空光照值。默认为0。
- `ambient_occlusion`：禁用或启用此面的环境光遮蔽。默认为模型中设置的值。

### 根变换(Root Transforms)

在模型顶层添加`transform`属性告诉加载器，在[方块状态文件][bsfile]（对于方块模型）中的旋转或`display`块（对于物品模型）中的变换应用之前，应对所有几何体应用变换。这是由NeoForge添加的。

根变换可以通过两种方式指定。第一种方式是作为名为`matrix`的单个属性，包含变换3x4矩阵（行主序，最后一行省略）作为嵌套JSON数组的形式。矩阵是按该顺序组成的平移、左旋转、缩放、右旋转和变换原点。示例如下所示：

```json5
{
    // ...
    "transform": {
        "matrix": [
            [0, 0, 0, 0],
            [0, 0, 0, 0],
            [0, 0, 0, 0]
        ]
    }
}
```

第二种方式是指定一个JSON对象，包含以下任何条目的组合，按该顺序应用：

- `translation`：相对平移。指定为三维向量（`[x, y, z]`），如果不存在则默认为`[0, 0, 0]`。
- `rotation`或`left_rotation`：围绕平移后的原点在缩放之前应用的旋转。默认为无旋转。以下列方式之一指定：
    - 具有单个轴到旋转映射的JSON对象，例如`{"x": 90}`
    - 具有单个轴到旋转映射的JSON对象数组，按指定顺序应用，例如`[{"x": 90}, {"y": 45}, {"x": -22.5}]`
    - 具有三个值的数组，每个值指定围绕每个轴的旋转，例如`[90, 45, -22.5]`
    - 具有四个值的数组，直接指定四元数，例如`[0.38268346, 0, 0, 0.9238795]`（= 围绕X轴45度）
- `scale`：相对于平移后的原点的缩放。指定为三维向量（`[x, y, z]`），如果不存在则默认为`[1, 1, 1]`。
- `post_rotation`或`right_rotation`：围绕平移后的原点在缩放之后应用的旋转。默认为无旋转。与`rotation`相同的方式指定。
- `origin`：用于旋转和缩放的原点。变换也作为最后一步移动到这里。指定为三维向量（`[x, y, z]`）或使用三个内置值之一`"corner"`（= `[0, 0, 0]`）、`"center"`（= `[0.5, 0.5, 0.5]`）或`"opposing-corner"`（= `[1, 1, 1]`，默认）。

## 方块状态文件(Blockstate Files)

_另请参见：[Minecraft Wiki][mcwiki]上的[方块状态文件][mcwikiblockstate]_

方块状态文件被游戏用于将不同的模型分配给不同的[方块状态]。每个注册到游戏的方块必须恰好有一个方块状态文件。为方块状态指定方块模型以三种互斥的方式工作：通过变体、多部分或NeoForge添加的定义类型。

在`variants`块内部，每个方块状态都有一个元素。这是将方块状态与模型关联的主要方式，绝大多数方块使用此方式。
- 键是没有方块名称的方块状态的字符串表示，例如，对于非含水顶层台阶，为`"type=top,waterlogged=false"`，或者对于没有属性的方块，为`""`。值得注意的是，未使用的属性可能会被省略。例如，如果`waterlogged`属性对所选模型没有影响，则两个对象`type=top,waterlogged=false`和`type=top,waterlogged=true`可能合并为一个`type=top`对象。这也意味着空字符串对每个方块都有效。
- 值可以是单个模型对象或模型对象的数组。如果使用模型对象数组，将从中随机选择一个模型。模型对象由以下数据组成：
    - `type`：NeoForge添加。设置自定义方块状态模型加载器。有关更多信息，请参见[方块状态模型加载器][bsmmodelloader]。
    - `model`：模型文件位置的路径，相对于命名空间的`models`文件夹，例如`minecraft:block/cobblestone`。
    - `x`和`y`：模型在x轴/y轴上的旋转。限制为90度的步长。每个都是可选的，默认为0。
    - `uvlock`：旋转时是否锁定模型的UV。可选，默认为false。
    - `weight`：仅对模型对象数组有用。给对象一个权重，用于选择随机模型对象。可选，默认为1。

相比之下，在`multipart`块内部，元素根据方块状态的属性组合。此方法主要由栅栏和墙使用，它们基于布尔属性启用四个方向部分。多部分元素由两部分组成：一个`when`块和一个`apply`块。

- `when`块指定方块状态的字符串表示或必须满足的元素才能应用的条件列表。列表可以命名为`"OR"`或`"AND"`，对其内容执行相应的逻辑操作。单个方块状态和列表值都可以通过用`|`分隔（例如`facing=east|facing=west`）来指定多个实际值。
- `apply`块指定要使用的模型对象或模型对象数组。这与`variants`块完全相同。

最后，`neoforge:definition_type`可以指定自定义模型加载器来注册方块状态文件。有关更多信息，请参见[方块状态定义加载器][bsdmodelloader]。

## 客户端物品(Client Items)

[客户端物品][citems]被游戏用于将一个或多个模型分配给`ItemStack`的状态。虽然模型JSON中有一些物品特定字段，但客户端物品根据上下文消费模型进行渲染，因此它们的大部分信息已移到它们自己单独的[部分][citems]中。

## 着色(Tinting)

一些方块，例如草或树叶，会根据其位置和/或属性更改其纹理颜色。[模型元素][elements]可以在其面上指定色调索引，这将允许颜色处理器处理相应的面。代码方面的工作通过三个事件进行，一个用于方块颜色处理器，一个用于基于生物群系的方块着色（与方块颜色处理器结合使用），一个用于物品着色源。它们的工作方式非常相似，因此让我们首先看一下方块处理器：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerBlockColorHandlers(RegisterColorHandlersEvent.Block event) {
    // 参数是方块的状态、方块所在的维度、方块的位置和色调索引。
    // 维度和位置可能为null。
    event.register((state, level, pos, tintIndex) -> {
        // 替换为您自己的计算。有关原版参考，请参见BlockColors类。
        // 颜色为ARGB格式。通常，如果色调索引为-1，则表示不应进行着色，应使用默认值。
        return 0xFFFFFFFF;
    },
    // 应用着色的方块可变参数
    EXAMPLE_BLOCK.get(), ...);
}
```

以下是颜色解析器的示例：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerColorResolvers(RegisterColorHandlersEvent.ColorResolvers event) {
    // 参数是当前的生物群系、方块的X位置和方块的Z位置。
    event.register((biome, x, z) -> {
        // 替换为您自己的计算。有关原版参考，请参见BiomeColors类。
        // 颜色为ARGB格式。
        return 0xFFFFFFFF;
    });
}
```

有关物品着色，请参见[客户端物品文章中的相关部分][itemtints]。

## 注册独立模型(Registering Standalone Models)

不以某种方式与方块或物品关联但仍在其他上下文中（例如[方块实体渲染器][ber]）所需的模型可以通过`ModelEvent.RegisterStandalone`注册：

```java
// 这可以是任何类型，只要可以从ResolvedModel和ModelBaker获取
// 泛型类型应该是UnbakedStandaloneModel<T>的泛型类型
public static final StandaloneModelKey<QuadCollection> EXAMPLE_KEY = new StandaloneModelKey<>(
    new ModelDebugName() {
        @Override
        public String debugName() {
            // 独立模型的名称
            // 可以是任何字符串，但应包含模组ID
            return "examplemod: Example Model";
        }
    }
);

@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerAdditional(ModelEvent.RegisterStandalone event) {
    event.register(
        // 要获取的模型
        EXAMPLE_KEY,
        // 我们关心的UnbakedStandaloneModel<T>，在这种情况下返回QuadCollection
        // 为简单起见，可以使用SimpleUnbakedStandaloneModel<T>的静态方法
        SimpleUnbakedStandaloneModel.quadCollection(
            // 模型ID，相对于`assets/<namespace>/models/<path>.json`
            ResourceLocation.fromNamespaceAndPath("examplemod", "block/example_unused_model")
        )
    );
}
```

[ao]: https://en.wikipedia.org/wiki/Ambient_occlusion
[ber]: ../../../blockentities/ber.md
[bsfile]: #blockstate-files
[bsdmodelloader]: modelloaders.md#block-state-definition-loaders
[bsmmodelloader]: modelloaders.md#block-state-model-loaders
[custommodelloader]: modelloaders.md#model-loaders
[elements]: #elements
[event]: ../../../concepts/events.md
[extrafacedata]: #extra-face-data
[citems]: items.md
[itemmodel]: items.md#a-basic-model
[itemtints]: items.md#tinting
[mcwiki]: https://minecraft.wiki
[mcwikiblockstate]: https://minecraft.wiki/w/Tutorials/Models#Block_states
[mcwikimodel]: https://minecraft.wiki/w/Model
[mipmapping]: https://en.wikipedia.org/wiki/Mipmap
[modbus]: ../../../concepts/events.md#event-buses
[perspectives]: modelsystem.md#perspectives
[rendertype]: #render-types
[roottransforms]: #root-transforms
[rl]: ../../../misc/resourcelocation.md
[textures]: ../textures.md
[tinting]: #tinting