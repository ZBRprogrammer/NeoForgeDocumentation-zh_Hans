# 屏幕 (Screens)

屏幕 (Screens) 通常是 Minecraft 中所有图形用户界面 (GUI) 的基础：接收用户输入，在服务器端验证，并将结果操作同步回客户端。它们可以与[容器菜单 (menus)]结合，为类似物品栏的视图创建通信网络，也可以是独立的，由模组开发者通过自己的[网络 (network)]实现来处理。

屏幕由众多部分组成，因此很难完全理解 Minecraft 中的“屏幕”到底是什么。因此，本文档将在讨论屏幕本身之前，逐一介绍屏幕的各个组成部分及其应用方式。

## 渲染 GUI (Rendering a GUI)

渲染 GUI 分为两个步骤：提交阶段 (submission phase) 和渲染阶段 (render phase)。

提交阶段负责收集要渲染到屏幕的所有元素（例如按钮、文本、物品）。每次提交都将存储在 `GuiRenderState` 中，以便在渲染阶段进行处理和渲染。Vanilla 提供了四种类型的元素：`GuiElementRenderState`、`GuiItemRenderState`、`GuiTextRenderState` 和 `PictureInPictureRenderState`。下面章节讨论的所有内容都是在提交阶段通过内部创建上述渲染状态之一来进行的。

渲染阶段，顾名思义，将元素渲染到屏幕上。首先，准备 `PictureInPictureRenderState`、`GuiItemRenderState` 和 `GuiTextRenderState`，并将其处理为 `GuiElementRenderState`s。然后，对元素进行排序，最后绘制到屏幕上。最后，重置 `GuiRenderState`，准备用于下一个 GUI 或渲染刻。

### 相对坐标 (Relative Coordinates)

每当有任何内容提交到渲染状态时，都需要一些坐标来指定元素的渲染位置。由于存在大量抽象，Minecraft 的大多数渲染调用都接受 X 和 Y 坐标。X 值从左到右增加，Y 值从上到下增加。但是，坐标的范围并不固定于特定范围。它们的范围可以根据屏幕大小和游戏选项中指定的 GUI 缩放比例而改变。因此，必须特别注意确保传递给渲染调用的坐标值能够正确缩放——正确地相对化 (relativized) ——以适应可变的屏幕大小。

关于如何将坐标相对化的信息在[屏幕 (screen)]部分。

:::caution
如果你选择使用固定坐标或不正确地缩放屏幕，渲染的对象可能会看起来很奇怪或错位。检查你是否正确相对化了坐标的一个简单方法是点击视频设置中的“GUI 缩放 (Gui Scale)”按钮。在确定 GUI 应渲染的比例时，此值用作显示宽度和高度的除数。
:::

### GuiGraphics

提交到 `GuiRenderState` 的任何元素通常都通过 `GuiGraphics` 处理。`GuiGraphics` 是提交阶段几乎所有方法的第一个参数，包含将常用对象提交以供渲染的方法。

`GuiGraphics` 将当前姿态 (pose) 作为 `Matrix3x2fStack` 公开，以应用任何 XY 变换：

```java
// 对于某个 GuiGraphics graphics

// 将新矩阵推入栈中
graphics.pose().pushMatrix();

// 应用你希望元素渲染时使用的变换

// 接受一些 XY 偏移
graphics.pose().translate(10, 10);
// 接受一些旋转角度（弧度）
graphics.pose().rotate((float) Math.PI);
// 接受一些 XY 缩放因子
graphics.pose().scale(2f, 2f);

// 将元素提交到 `GuiRenderState`
graphics.blitSprite(...);

// 弹出矩阵以重置变换
graphics.pose().popMatrix();
```

此外，可以使用 `enableScissor` 和 `disableScissor` 将元素裁剪到特定区域：

```java
// 对于某个 GuiGraphics graphics

// 启用剪裁，并设置渲染区域的边界
graphics.enableScissor(
    // 左侧 X 坐标
    0,
    // 顶部 Y 坐标
    0,
    // 右侧 X 坐标
    10,
    // 底部 Y 坐标
    10
);

// 将元素提交到 `GuiRenderState`
graphics.blitSprite(...);

// 禁用剪裁以重置渲染区域
graphics.disableScissor();
```

### 节点树与层级 (Node Trees and Strata)

当一个元素提交到 `GuiRenderState` 时，它不仅仅是添加到某个列表中。如果是那样，某些元素可能会根据提交的顺序被其他元素完全覆盖。为了克服这个障碍，元素最初被分类到某个层级 (stratum) 的节点树 (node trees) 中。元素的分类方式基于其定义的 `ScreenArea#bounds`；否则，该元素将不会被提交渲染。

`GuiRenderState` 由作为单链表的 `GuiRenderState.Node`s 组成，使用“向上 (up)”来保存对下一个元素的引用。节点从第一个元素开始“向上”渲染。每个节点持有自己的图层数据，包含渲染状态。当一个元素最初提交到 `GuiRenderState` 时，它会根据其定义的 `ScreenArea#bounds` 决定使用或创建哪个节点。所选或创建的节点，是位于具有相交元素的最高节点之上的一个节点。

:::warning
尽管 `ScreenArea#bounds` 标记为可为空 (nullable)，但如果未定义边界，则提交到渲染状态的元素将不会被添加。该方法之所以可为空，是因为在渲染阶段提交的元素会被添加到当前节点，而不是基于边界计算其节点。
:::

每个节点列表在渲染状态中称为一个层级 (stratum)。渲染状态可以通过调用 `GuiGraphics#nextStratum` 拥有多个层级，从而创建一个新的节点列表。新的层级将渲染在所有先前层级的元素之上（例如，物品提示框 (tooltips)）。一旦调用 `nextStratum`，就无法导航回先前的层级。

### `GuiElementRenderState`

`GuiElementRenderState` 保存有关 GUI 元素如何渲染到屏幕的元数据。元素渲染状态继承 `ScreenArea`，以定义屏幕上的 `bounds`。边界应始终包含渲染的整个元素，以便在节点列表中正确排序。边界计算通常包含下面提到的一些参数，包括位置和姿态。

`scissorArea` 裁剪元素可以渲染的区域。如果 `scissorArea` 为 `null`，则整个元素渲染到屏幕上。同样，如果 `scissorArea` 矩形不与 `bounds` 相交，则不会渲染任何内容。

剩下的三个方法处理元素的实际渲染。`pipeline` 定义元素使用的着色器和元数据。`textureSetup` 可以指定片段着色器 (fragment shader) 中的 `Sampler0`、`Sampler1`、`Sampler2` 或其某种组合。最后，`buildVertices` 传递要上传到缓冲区的顶点 (vertices)。它接收要传递顶点的 `VertexConsumer`。

NeoForge 添加了方法 `GuiGraphics#submitGuiElementRenderState`，如果 `GuiGraphics` 提供的可用方法不够用，可以提交自定义的元素渲染状态。

```java
// 对于某个 GuiGraphics graphics
graphics.submitGuiElementRenderState(new GuiElementRenderState() {

    // 存储栈的当前姿态
    private final Matrix3x2f pose = new Matrix3x2f(graphics.pose());
    // 存储当前的剪裁区域
    @Nullable
    private final ScreenRectangle scissorArea = graphics.peekScissorStack();

    @Override
    public ScreenRectangle bounds() {
        // 我们假设边界是 0, 0, 10, 10
        
        // 计算初始矩形
        ScreenRectangle rectangle = new ScreenRectangle(
            // XY 位置
            0, 0,
            // 元素的宽度和高度
            10, 10
        );

        // 使用姿态将矩形变换到其适当的位置
        rectangle = rectangle.transformMaxBounds(this.pose);

        // 如果定义了剪裁区域，则返回两个矩形的交集
        // 否则，返回完整边界
        return this.scissorArea != null
            ? this.scissorArea.intersection(rectangle)
            : rectangle;
    }

    @Override
    @Nullable
    public ScreenRectangle scissorArea() {
        return this.scissorArea;
    }

    @Override
    public RenderPipeline pipeline() {
        return RenderPipelines.GUI;
    }

    @Override
    public TextureSetup textureSetup() {
        // 返回片段着色器中采样器 (samplers) 使用的纹理。
        // 在片段着色器中使用时：
        // - Sampler0 通常包含元素纹理
        // - Sampler1 通常提供第二个元素纹理，目前仅用于末地传送门管线 (end portal pipeline)
        // - Sampler2 通常包含游戏的光照图纹理 (lightmap texture)

        // 通常至少应在 Sampler0 中指定一个纹理
        return TextureSetup.noTexture();
    }

    @Override
    public void buildVertices(VertexConsumer consumer) {
        // 使用管线指定的顶点格式 (vertex format) 构建顶点
        // 对于 GUI，使用具有位置和颜色的四边形 (quads)
        // 颜色必须为 ARGB 格式
        consumer.addVertexWith2DPose(this.pose, 0,   0).setUv(0, 0).setColor(0xFFFFFFFF);
        consumer.addVertexWith2DPose(this.pose, 0,  10).setUv(0, 1).setColor(0xFFFFFFFF);
        consumer.addVertexWith2DPose(this.pose, 10, 10).setUv(1, 1).setColor(0xFFFFFFFF);
        consumer.addVertexWith2DPose(this.pose, 10,  0).setUv(1, 0).setColor(0xFFFFFFFF);
    }
});
```

### 元素排序 (Element Ordering)

到目前为止，上面展示的元素仅对 XY 坐标进行操作。Z 坐标在 GUI 渲染中被忽略，因为绘制到屏幕上的所有元素都使用禁用深度测试 (depth test) 的 `RenderPipeline`。即使是那些具有更高级管线的 3D 元素，默认情况下也使用 `RenderPipeline#GUI_TEXTURED_PREMULTIPLIED_ALPHA` 绘制到 2D 纹理上，该管线同样禁用深度测试，从而防止常见用例中的 Z 冲突 (Z-fighting)。

因此，在渲染阶段，每个层级按顺序渲染，节点列表中的节点从第一个元素开始“向上”渲染。但是在给定节点内呢？这是通过 `GuiRenderer#ELEMENT_SORT_COMPARATOR` 处理的，它根据元素的 `GuiElementRenderState#scissorArea`、`pipeline`，然后是 `textureSetup` 进行排序。

:::warning
为文本渲染的字形 (glyphs) 不会被排序，并且总是在当前节点的所有元素之后渲染。
:::

未指定 `scissorArea` 的元素将始终最先渲染，然后是顶部 Y、底部 Y、左侧 X，最后是右侧 X。如果两个元素的 `scissorArea` 匹配，则将使用 `pipeline` 的排序键 (sort key)（通过 `RenderPipeline#getSortKey`）。排序键基于 `RenderPipeline`s 的构建顺序，在 Vanilla 中这是 `RenderPipelines` 中静态常量的类加载顺序。如果排序键匹配，则使用 `textureSetup`。未指定 `textureSetup` 的元素最先排序，然后是纹理元素的排序键（通过 `TextureSetup#getSortKey`）。

:::warning
从技术层面讲，由于 `RenderPipeline` 和 `TextureSetup`，元素排序不是确定性的 (deterministic)。这是因为排序的目标不是确定性，而是尽可能以最少的管线切换和纹理切换来渲染元素。
:::

## `GuiGraphics` 中的方法

`GuiGraphics` 包含用于提交常用对象以供渲染的方法。这些方法分为六类：彩色矩形、字符串、纹理、物品、提示框 (tooltips) 和画中画 (picture-in-pictures)。这些方法中的每一个都提交一个元素，从 `pose` 继承当前姿态，并根据 `enableScissor` / `disableScissor` 从 `peekScissorStack` 获取剪裁区域。提供给方法的任何颜色都必须是 [ARGB][argb] 格式。

### 彩色矩形 (Colored Rectangles)

彩色矩形使用 `ColoredRectangleRenderState` 提交。所有填充方法都可以接受可选的 `RenderPipeline` 和 `TextureSetup` 参数，以指定矩形应如何渲染。可以提交三种类型的彩色矩形。

首先，有一个像素宽的彩色水平和垂直线，分别是 `hLine` 和 `vLine`。`hLine` 接受两个定义左和右（含）的 X 坐标、顶部 Y 坐标和颜色。`vLine` 接受左侧 X 坐标、两个定义顶部和底部（含）的 Y 坐标，以及颜色。

其次，有 `fill` 方法，它提交一个要绘制到屏幕的矩形。线方法在内部调用此方法。它接受左侧 X 坐标、顶部 Y 坐标、右侧 X 坐标、底部 Y 坐标和颜色。

第三，有 `submitOutline` 方法，它提交四个一个像素宽的矩形作为轮廓。它接受左侧 X 坐标、顶部 Y 坐标、轮廓的宽度、轮廓的高度和颜色。

最后，有 `fillGradient` 方法，它绘制一个带有垂直渐变的矩形。它接受左侧 X 坐标、顶部 Y 坐标、右侧 X 坐标、底部 Y 坐标以及底部和顶部颜色。

### 字符串 (Strings)

字符串 (Strings)、[`Component`s][component] 和 `FormattedCharSequence`s 使用 `GuiTextRenderState` 提交。每个字符串都通过提供的 `Font` 绘制，该字体用于创建 `BakedGlyph.GlyphInstance` 和可选的 `BakedGlyph.Effect`，并使用指定的 `GlyphRenderTypes#guiPipeline`。然后在渲染阶段，文本渲染状态根据字符串中的每个字符转换为 `GlyphRenderState`s 和可能的 `GlyphEffectRenderState`。

字符串可以以两种对齐方式渲染：左对齐字符串 (`drawString`) 和居中对齐字符串 (`drawCenteredString`)。两者都接受字符串渲染所使用的字体、要绘制的字符串、分别代表字符串左侧或中心的 X 坐标、顶部 Y 坐标和颜色。左对齐字符串还可以接受是否绘制文本的阴影。

如果文本应在给定边界内自动换行 (wrap)，则可以使用 `drawWordWrap`。如果文本应具有某种矩形背景，则可以使用 `drawStringWithBackdrop`。默认情况下，它们都提交左对齐字符串。

:::note
字符串通常应作为 [`Component`s][component] 传递，因为它们处理各种用例，包括该方法的另外两个重载。
:::

### 纹理 (Textures)

纹理通过 `BlitRenderState` 提交，因此方法名为 `blit`。`BlitRenderState` 复制图像的位 (bits) 并通过 `RenderPipeline` 参数将它们渲染到屏幕上。每个 `blit` 还接受一个 `ResourceLocation`，它表示纹理的绝对位置：

```java
// 指向 'assets/examplemod/textures/gui/container/example_container.png'
private static final ResourceLocation TEXTURE = ResourceLocation.fromNamespaceAndPath("examplemod", "textures/gui/container/example_container.png");
```

虽然有很多不同的 `blit` 重载，但我们只讨论其中的两个。

第一个 `blit` 接受两个整数，然后是两个浮点数，最后再接受四个整数，假设图像在 PNG 文件上。它接受屏幕上的左侧 X 和顶部 Y 坐标、PNG 内的左侧 X 和顶部 Y 坐标、要渲染的图像的宽度和高度，以及 PNG 文件的宽度和高度。

:::tip
必须指定 PNG 文件的大小，以便将坐标归一化 (normalized) 以获得相关的 UV 值。
:::

第二个 `blit` 在末尾添加了一个额外的整数，代表要绘制的图像的着色 (tint) 颜色。如果未指定，着色颜色为 `0xFFFFFFFF`。

#### `blitSprite`

`blitSprite` 是 `blit` 的一个特殊实现，其中纹理是从 GUI 纹理图集 (texture atlas) 中获取的。大多数覆盖背景的纹理，例如熔炉 GUI 中的“燃烧进度”覆盖层，都是精灵图 (sprites)。所有精灵图纹理都相对于 `textures/gui/sprites`，并且不需要指定文件扩展名。

```java
// 指向 'assets/examplemod/textures/gui/sprites/container/example_container/example_sprite.png'
private static final ResourceLocation SPRITE = ResourceLocation.fromNamespaceAndPath("examplemod", "container/example_container/example_sprite");
```

一组 `blitSprite` 方法的参数与 `blit` 相同，除了处理 PNG 坐标、宽度和高度的四个整数。

另一组 `blitSprite` 方法接受更多的纹理信息，以允许绘制精灵图的一部分。这些方法接受精灵图的宽度和高度、精灵图中的 X 和 Y 坐标、屏幕上的左侧 X 和顶部 Y 坐标、着色颜色，以及要渲染的图像的宽度和高度。

如果精灵图大小与纹理大小不匹配，则可以通过以下三种方式之一缩放精灵图：`stretch`、`tile` 和 `nine_slice`。`stretch` 将图像从纹理大小拉伸到屏幕大小。`tile` 反复渲染纹理，直到达到屏幕大小。`nine_slice` 将纹理划分为一个中心、四个边和四个角，以将纹理平铺到所需的屏幕大小。

这是通过在纹理文件同名的 mcmeta 文件中添加 `gui.scaling` JSON 对象来设置的。

```json5
// 对于某个纹理文件 example_sprite.png
// 在 example_sprite.png.mcmeta 中

// 拉伸示例
{
    "gui": {
        "scaling": {
            "type": "stretch"
        }
    }
}

// 平铺示例
{
    "gui": {
        "scaling": {
            "type": "tile",
            // 开始平铺的大小
            // 这通常是纹理的大小
            "width": 40,
            "height": 40
        }
    }
}

// 九宫格切片示例
{
    "gui": {
        "scaling": {
            "type": "nine_slice",
            // 开始平铺的大小
            // 这通常是纹理的大小
            "width": 40,
            "height": 40,
            "border": {
                // 将被切片到边框纹理中的纹理填充 (padding)
                "left": 1,
                "right": 1,
                "top": 1,
                "bottom": 1
            },
            // 为 true 时，纹理的中心部分将像拉伸类型一样应用，而不是九宫格平铺。
            "stretch_inner": true
        }
    }
}
```

:::note
当使用设置了平铺或九宫格切片的纹理时，`blitSprite` 将使用 `TiledBlitRenderState` 提交其元素，除了 `BlitRenderState` 中的其他参数外，它还指定了平铺的宽度和高度。
:::

### 物品 (Items)

物品使用 `GuiItemRenderState` 提交。然后在渲染阶段，物品渲染状态根据物品边界和客户端物品属性 (client item properties) 转换为 `BlitRenderState` 或 `OversizedItemRenderState`。

`renderItem` 接受一个 `ItemStack`，以及屏幕上的左侧 X 和顶部 Y 坐标。它还可以选择接受持有的 `LivingEntity`、`ItemStack` 所在的当前 `Level`，以及一个种子值。还有一个替代方法 `renderFakeItem`，它将 `LivingEntity` 设置为 `null`。

物品装饰——例如耐久条 (durability bar)、冷却时间 (cooldown) 和数量——通过 `renderItemDecorations` 处理。它接受与基础 `renderItem` 相同的参数，以及 `Font` 和数量文本覆盖。

### 提示框 (Tooltips)

提示框通过各种上述渲染状态提交。提示框方法分为两类：“下一帧 (next frame)”和“立即 (immediate)”。两种方法都接受渲染文本的 `Font`、一些 `Component`s 列表、用于特殊渲染的可选 `TooltipComponent`、左侧 X 和顶部 Y、用于调整位置的 `ClientTooltipPositioner`，以及背景和边框纹理。

下一帧提示框实际上并不会在下一帧提交提示框，而是将提示框的提交延迟到调用 `Screen#render` 之后。提示框被添加到一个新的层级，这意味着它将在屏幕所有元素之上渲染。下一帧方法的形式是 `set*Tooltip*ForNextFrame`。它们还可以接受一个额外的布尔值，指示如果存在当前延迟的提示框是否覆盖它，以及渲染的提示框应使用的 `ItemStack`。

另一方面，立即提示框在调用方法时立即提交。立即方法的形式是 `renderTooltip`。它们还接受提示框悬停在其上的 `ItemStack`。

### 画中画 (Picture-in-Picture)

画中画 (PiP) 允许将任意对象绘制到屏幕上。PiP 不是直接绘制到输出，而是将对象绘制到一个中间纹理，或者说“画 (picture)”，然后在渲染阶段将其作为 `BlitRenderState` 提交到 `GuiRenderState`（默认情况下）。`GuiGraphics` 提供了地图、实体、玩家皮肤、书本模型、旗帜图案、告示牌和分析器图表 (profiler chart) 的方法，通过 `submit*RenderState`。

:::note
超过默认 16x16 边界的物品，当 `ClientItem.Properties#oversizedInGui` 为 true 时，使用 `OversizedItemRenderer` PiP 作为其渲染机制。
:::

每个 PiP 提交一个 `PictureInPictureRenderState` 以将对象渲染到屏幕上。与 `GuiElementRenderState` 类似，`PictureInPictureRenderState` 也继承 `ScreenArea` 来定义其 `bounds` 和通过 `scissorArea` 的剪裁。然后，`PictureInPictureRenderState` 定义画 (picture) 的渲染位置和大小，指定左侧 X (`x0`)、右侧 X (`x1`)、顶部 Y (`y0`) 和底部 Y (`y1`)。画内的元素也可以通过某个浮点值进行 `scale`。最后，可以使用额外的 `pose` 来变换画的 XY 坐标。默认情况下，这是单位姿态 (identity pose)，因为通常渲染的对象已经在画本身内进行了变换。为了便于实现，可以使用 `PictureInPictureRenderState#getBounds` 计算 `bounds`，但如果修改了 `pose`，则需要实现自己的逻辑。

```java
// 可以添加其他参数，但这是实现所有方法所需的最小要求
public record ExampleRenderState(
    int x0, // 左侧 X
    int x1, // 右侧 X
    int y0, // 顶部 Y
    int y1, // 底部 Y
    float scale, // 绘制到画时的缩放因子
    @Nullable ScreenRectangle scissorArea, // 渲染区域
    @Nullable ScreenRectangle bounds // 元素的边界
) implements PictureInPictureRenderState {

    // 额外的构造函数
    public ExampleRenderState(int x, int y, int width, int height, @Nullable ScreenRectangle scissorArea) {
        this(
            x, // x0
            x + width, // x1
            y, // y0
            y + height, // y1
            1f, // scale
            scissorArea,
            PictureInPictureRenderState.getBounds(x, y, x + width, y + height, scissorArea)
        );
    }
}
```

要绘制 PiP 渲染状态并将其提交到画，每个 PiP 都有自己的 `PictureInPictureRenderer<T>`，其中 `T` 是实现的 `PictureInPictureRenderState`。有许多方法可以被重写，允许用户几乎完全控制整个管线，但有三个必须实现。

第一个是 `getRenderStateClass`，它简单地返回 `PictureInPictureRenderState` 的类。在 Vanilla 中，此方法用于注册渲染器用于哪种渲染状态。NeoForge 仍然使用渲染状态类，但通过事件提供注册，以映射到动态的渲染器池，而不是调用 `getRenderStateClass`。

然后是 `getTextureLabel`，它为正在写入的画提供唯一的调试标签 (debug label)。最后是 `renderToTexture`，它实际上将对象绘制到画中，类似于其他渲染方法。

```java
public class ExampleRenderer extends PictureInPictureRenderer<ExampleRenderState> {

    // 接收用于将对象写入画的缓冲区
    public ExampleRenderer(MultiBufferSource.BufferSource bufferSource) {
        super(bufferSource);
    }

    @Override
    public Class<ExampleRenderState> getRenderStateClass() {
        // 返回渲染状态类
        return ExampleRenderState.class;
    }

    @Override
    protected String getTextureLabel() {
        // 可以是任何字符串，但应该是唯一的
        // 为更清晰起见，可以加上模组 ID 前缀
        return "examplemod: example pip";
    }

    @Override
    protected void renderToTexture(ExampleRenderState renderState, PoseStack pose) {
        // 如果需要，修改姿态
        // 如果需要，可以 push/pop，但为写入画会创建一个新的 `PoseStack`
        pose.translate(...);

        // 将对象渲染到屏幕
        VertexConsumer consumer = this.bufferSource.getBuffer(RenderType.lines());
        consumer.addVertex(...).setColor(...).setNormal(...);
        consumer.addVertex(...).setColor(...).setNormal(...);
    }

    // 额外的方法

    @Override
    protected void blitTexture(ExampleRenderState renderState, GuiRenderState guiState) {
        // 默认情况下，将画作为 `BlitRenderState` 提交到 gui 渲染状态
        // 如果你想修改 `BlitRenderState`，请重写此方法
        // 应调用 `GuiRenderState#submitBlitToCurrentLayer`
        // Bounds 可以为 `null`
        super.blitTexture(renderState, guiState);
    }

    @Override
    protected boolean textureIsReadyToBlit(ExampleRenderState renderState) {
        // 为 true 时，这会重用已经写入的画，而不是
        // 构造新画并使用 `renderToTexture` 写入。
        // 仅当保证两个元素将被渲染得*完全*相同时，才应为 true。
        return super.textureIsReadyToBlit(renderState);
    }

    @Override
    protected float getTranslateY(int scaledHeight, int guiScale) {
        // 设置 `PoseStack` 在 Y 方向上初始平移的偏移量。
        // 常见的实现使用 `scaledHeight / 2f` 来像 X 坐标一样将 Y 坐标居中。
        return scaledHeight;
    }

    @Override
    public boolean canBeReusedFor(ExampleRenderState state, int textureWidth, int textureHeight) {
        // NeoForge 添加的方法，用于检查此渲染器是否可以在后续帧中重用。
        // 为 true 时，将重用前一帧构造的状态和渲染器。
        // 为 false 时，将创建新的渲染器。
        return super.canBeReusedFor(state, textureWidth, textureHeight);
    }
}
```

要使用 PiP，必须在[模组事件总线 (mod event bus)][modbus]上的 `RegisterPictureInPictureRenderersEvent` 中注册渲染器。

```java
@SubscribeEvent // 在模组事件总线上
public static void registerPip(RegisterPictureInPictureRenderersEvent event) {
    event.register(
        // PiP 渲染状态类
        ExampleRenderState.class,
        // 一个工厂，接收 `MultiBufferSource.BufferSource` 并返回 PiP 渲染器
        ExampleRenderer::new
    );
}
```

然后可以使用 NeoForge 添加的 `GuiGraphics#submitPictureInPictureRenderState` 提交 PiP 渲染状态：

```java
// 对于某个 GuiGraphics graphics
graphics.submitPictureInPictureRenderState(new ExampleRenderState(
    0, 0,
    10, 10,
    // 从栈中获取剪裁区域
    graphics.peekScissorStack()
));
```

:::note
NeoForge 修复了一个错误，该错误阻止在任何给定帧中提交 PiP 渲染状态的多个实例。
:::

## Renderable

`Renderable`s 本质上是可渲染的对象。这些包括屏幕、按钮、聊天框、列表等。`Renderable`s 只有一个方法：`#render`。它接受用于将内容渲染到屏幕的 `GuiGraphics`、鼠标相对于屏幕大小的 X 和 Y 位置，以及刻增量 (tick delta)（自上一帧以来经过的刻数）。

一些常见的可渲染对象是屏幕和“窗口部件 (widgets)”：通常在屏幕上渲染的可交互元素，例如 `Button`、其子类型 `ImageButton` 和用于在屏幕上输入文本的 `EditBox`。

## GuiEventListener

在 Minecraft 中渲染的任何屏幕都实现 `GuiEventListener`。`GuiEventListener`s 负责处理用户与屏幕的交互。这些包括来自鼠标（移动、点击、释放、拖动、滚动、鼠标悬停）和键盘（按下、释放、输入）的输入。每个方法返回相关操作是否成功影响了屏幕。按钮、聊天框、列表等窗口部件也实现了此接口。

### ContainerEventHandler

与 `GuiEventListener`s 几乎同义的是它们的子类型：`ContainerEventHandler`s。这些负责处理包含窗口部件的屏幕上的用户交互，管理当前焦点以及相关交互的应用方式。`ContainerEventHandler`s 添加了三个额外特性：可交互子项 (interactable children)、拖动 (dragging) 和聚焦 (focusing)。

事件处理器 (event handlers) 持有子项 (children)，用于确定元素的交互顺序。在鼠标事件处理器期间（拖动除外），列表中鼠标悬停的第一个子项会执行其逻辑。

使用鼠标拖动元素（通过 `#mouseClicked` 和 `#mouseReleased` 实现）提供了更精确的执行逻辑。

聚焦允许在事件执行期间（例如在键盘事件或拖动鼠标时）首先检查和处理特定的子项。焦点通常通过 `#setFocused` 设置。此外，可以使用 `#nextFocusPath` 循环遍历可交互子项，根据传入的 `FocusNavigationEvent` 选择子项。

:::note
屏幕通过 `AbstractContainerEventHandler` 实现 `ContainerEventHandler`，它添加了用于拖动和聚焦子项的设置器和获取器逻辑。
:::

## NarratableEntry

`NarratableEntry`s 是可以通过 Minecraft 的可访问性旁白 (narration) 功能来讲述的元素。每个元素可以根据悬停或选择的内容提供不同的旁白，通常按焦点、悬停和所有其他情况的优先级排序。

`NarratableEntry`s 有四个方法：两个确定元素在被朗读时的优先级（`#narrationPriority` 和 `#getTabOrderGroup`），一个确定是否讲述旁白（`#isActive`），最后一个提供旁白到其关联的输出，无论是朗读还是阅读（`#updateNarration`）。

:::note
Minecraft 的所有窗口部件都是 `NarratableEntry`s，因此如果使用可用的子类型，通常不需要手动实现。
:::

## Screen 子类型

有了以上所有知识，就可以构建一个基本的屏幕。为了更容易理解，将按照通常遇到屏幕组件的顺序进行说明。

首先，所有屏幕都接受一个 `Component`，代表屏幕的标题。该组件通常由其某个子类型绘制到屏幕上。它仅在基础屏幕中用于旁白消息。

```java
// 在某个 Screen 子类中
public MyScreen(Component title) {
    super(title);
}
```

### 初始化 (Initialization)

屏幕初始化后，会调用 `#init` 方法。`init` 方法从 `Minecraft` 实例中设置屏幕内的初始设置，包括相对宽度和高度（经过游戏缩放）。任何设置，例如添加窗口部件或预计算相对坐标，都应在此方法中进行。如果游戏窗口调整大小，屏幕将通过调用 `init` 方法重新初始化。

有三种方法可以将窗口部件添加到屏幕，每种方法都有不同的目的：

| 方法 (Method) | 描述 (Description) |
|:--------------------:|:------------------------------------------------------------------------------|
|`addWidget` | 添加一个可交互且有旁白的窗口部件，但不渲染。 |
|`addRenderableOnly` | 添加一个仅被渲染的窗口部件；它不可交互也没有旁白。 |
|`addRenderableWidget` | 添加一个可交互、有旁白且被渲染的窗口部件。 |

通常，`addRenderableWidget` 将最常被使用。

```java
// 在某个 Screen 子类中
@Override
protected void init() {
    super.init();

    // 添加窗口部件和预计算值
    this.addRenderableWidget(new EditBox(/* ... */));
}
```

### 屏幕刻处理 (Ticking Screens)

屏幕还使用 `#tick` 方法来刻处理，以执行一些客户端逻辑用于渲染目的。

```java
// 在某个 Screen 子类中
@Override
public void tick() {
    super.tick();

    // 每帧执行一些逻辑
}
```

### 输入处理 (Input Handling)

由于屏幕是 `GuiEventListener`s 的子类型，输入处理器也可以被重写，例如用于处理特定[按键映射 (key mapping)]上的逻辑。

### 渲染屏幕 (Rendering the Screen)

屏幕通过 `#renderWithTooltipAndSubtitles` 在三个不同的层级中提交其元素：背景层 (background stratum)、渲染层 (render stratum) 和可选的提示框层 (tooltip stratum)。

背景层元素首先通过 `#renderBackground` 提交，通常包含任何模糊或背景纹理。

:::warning
模糊处理 (blurring)，通过 `GuiGraphics#blurBeforeThisStratum` 处理，在任何给定帧上只能调用一次。尝试渲染第二个模糊会导致抛出异常。
:::

渲染层元素接下来通过 `#render` 方法提交，这是作为 `Renderable` 子类型提供的。这主要提交窗口部件和标签，并设置在下一层级渲染的提示框。

最后，提示框层提交设置的提示框。提示框在渲染层通过 `GuiGraphics#setTooltipForNextFrame` 或 `GuiGraphics#setComponentTooltipFromElementsForNextFrame` 提交，这些方法可以接受要提交的文本或提示框组件以及提示框应在屏幕上渲染的 XY 相对坐标。

```java
// 在某个 Screen 子类中

// mouseX 和 mouseY 指示光标在屏幕上的缩放坐标
@Override
public void renderBackground(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    // 在背景层提交内容
    this.renderTransparentBackground(graphics);
}

@Override
public void render(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    // 在窗口部件之前提交内容

    // 然后提交窗口部件（如果这是 Screen 的直接子类）
    super.render(graphics, mouseX, mouseY, partialTick);

    // 在窗口部件之后提交内容

    // 设置提示框，使其在此方法中的所有内容之上添加
    graphics.setTooltipForNextFrame(...);
}
```

### 关闭屏幕 (Closing the Screen)

当屏幕关闭时，有两个方法处理清理：`#onClose` 和 `#removed`。

`onClose` 在用户进行输入以关闭当前屏幕时调用。此方法通常用作回调，以销毁和保存屏幕本身的任何内部进程。这包括向服务器发送数据包。

`removed` 在屏幕更改之前并释放给垃圾回收器之前调用。这处理屏幕打开前尚未重置回其初始状态的任何内容。

```java
// 在某个 Screen 子类中

@Override
public void onClose() {
    // 在此停止任何处理器

    // 最后调用，以防它干扰重写
    super.onClose();
}

@Override
public void removed() {
    // 在此重置初始状态

    // 最后调用，以防它干扰重写
    super.removed()
;}
```

## `AbstractContainerScreen`

如果屏幕直接连接到[容器菜单 (menu)][menus]，则应继承 `AbstractContainerScreen`。`AbstractContainerScreen` 充当菜单的屏幕和输入处理器，并包含与槽位同步和交互的逻辑。因此，通常只需要重写或实现两个方法来拥有一个正常工作的容器屏幕。再次，为了更容易理解，将按照通常遇到容器屏幕组件的顺序进行说明。

`AbstractContainerScreen` 通常需要三个参数：正在打开的容器菜单（由泛型 `T` 表示）、玩家物品栏（仅用于显示名称）和屏幕本身的标题。在此，可以设置许多定位字段：

字段 (Field) | 描述 (Description)
:---: | :---
`imageWidth` | 用于背景的纹理宽度。这通常在 256 x 256 的 PNG 中，默认为 176。
`imageHeight` | 用于背景的纹理高度。这通常在 256 x 256 的 PNG 中，默认为 166。
`titleLabelX` | 屏幕标题将渲染的相对 x 坐标。
`titleLabelY` | 屏幕标题将渲染的相对 y 坐标。
`inventoryLabelX` | 玩家物品栏名称将渲染的相对 x 坐标。
`inventoryLabelY` | 玩家物品栏名称将渲染的相对 y 坐标。

:::caution
在上一节中，提到预计算的相对坐标应在 `#init` 方法中设置。这仍然是正确的，因为这里提到的值不是预计算的坐标，而是静态值和相对化坐标。

图像值是静态且不变的，因为它们代表背景纹理大小。为了使渲染更容易，`init` 方法中预计算了两个额外的值（`leftPos` 和 `topPos`），标记背景将渲染的左上角。标签坐标是相对于这些值的。

`leftPos` 和 `topPos` 也是一种渲染背景的便捷方式，因为它们已经代表要传递给 `GuiGraphics#blit` 的位置。
:::

```java
// 在某个 AbstractContainerScreen 子类中
public MyContainerScreen(MyMenu menu, Inventory playerInventory, Component title) {
    super(menu, playerInventory, title);

    this.titleLabelX = 10;
    this.inventoryLabelX = 10;

    // 如果更改了 `imageHeight`，则也必须更改 `inventoryLabelY`，
    // 因为该值依赖于 `imageHeight` 值。
}
```

### 菜单访问 (Menu Access)

由于菜单被传递到屏幕中，菜单中已同步的任何值（通过槽位、数据槽位或自定义系统）现在都可以通过 `menu` 字段访问。

### 容器刻处理 (Container Tick)

当玩家存活并查看屏幕时，容器屏幕在 `#tick` 方法内通过 `#containerTick` 进行刻处理。这基本上取代了容器屏幕中的 `tick`，其最常见的用途是刻处理配方书。

```java
// 在某个 AbstractContainerScreen 子类中
@Override
protected void containerTick() {
    super.containerTick();

    // 在此刻处理事物
}
```

### 渲染容器屏幕 (Rendering the Container Screen)

容器屏幕使用所有三个层级来提交其元素。首先，背景层通过 `#renderBg` 提交背景纹理。然后，渲染层通过 `#render` 像之前一样提交窗口部件，然后通过 `#renderLabels` 提交任何文本。最后，`AbstractContainerScreen` 还提供了辅助方法 `renderTooltip`，将提示框提交到提示框层。

从 `render` 开始，最常见的重写（通常是唯一情况）调用父类来提交容器屏幕元素，然后是 `renderTooltip`。

```java
// 在某个 AbstractContainerScreen 子类中
@Override
public void render(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    // 提交要渲染的窗口部件和标签
    super.render(graphics, mouseX, mouseY, partialTick);

    // 此方法由容器屏幕添加，用于在提示框层提交悬停槽位的提示框。
    this.renderTooltip(graphics, mouseX, mouseY);
}
```

`renderBg` 被调用来将屏幕的背景元素提交到背景层。

```java
// 在某个 AbstractContainerScreen 子类中

// 背景纹理的位置 (assets/<命名空间>/<路径>)
private static final ResourceLocation BACKGROUND_LOCATION = ResourceLocation.fromNamespaceAndPath(MOD_ID, "textures/gui/container/my_container_screen.png");

@Override
protected void renderBg(GuiGraphics graphics, float partialTick, int mouseX, int mouseY) {
    // 提交背景纹理。'leftPos' 和 'topPos' 应该
    // 已经代表纹理应渲染的左上角，因为它是从 'imageWidth'
    // 和 'imageHeight' 预计算出来的。两个零代表 PNG 文件内的
    // 整数 u/v 坐标，其大小由最后两个整数表示（通常为 256 x 256）。
    graphics.blit(
        RenderPipelines.GUI_TEXTURED,
        BACKGROUND_LOCATION,
        this.leftPos, this.topPos,
        0, 0,
        this.imageWidth, this.imageHeight,
        256, 256
    );
}
```

`renderLabels` 被调用来在渲染层中窗口部件之后提交任何文本。它使用屏幕字体调用 `drawString` 来提交关联的组件。

```java
// 在某个 AbstractContainerScreen 子类中
@Override
protected void renderLabels(GuiGraphics graphics, int mouseX, int mouseY) {
    super.renderLabels(graphics, mouseX, mouseY);

    // 假设我们有一个 Component 'label'
    // 'label' 绘制在 'labelX' 和 'labelY' 处
    // 颜色是 ARGB 值
    // 最后的布尔值在 true 时渲染阴影
    graphics.drawString(this.font, this.label, this.labelX, this.labelY, 0xFF404040, false);
}
```

:::note
提交标签时，你**不**需要指定 `leftPos` 和 `topPos` 偏移。这些已经在 `Matrix3x2fStack` 内平移过了，因此此方法中的所有内容都相对于这些坐标提交。
:::

## 注册 AbstractContainerScreen

要将 `AbstractContainerScreen` 与菜单一起使用，需要注册它。这可以通过在[**模组事件总线 (mod event bus)**][modbus]上的 `RegisterMenuScreensEvent` 中调用 `register` 来完成。

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerScreens(RegisterMenuScreensEvent event) {
    event.register(MY_MENU.get(), MyContainerScreen::new);
}
```

[menus]: ../inventories/menus.md
[network]: ../networking/index.md
[screen]: #the-screen-subtype
[argb]: https://en.wikipedia.org/wiki/RGBA_color_model#ARGB32
[component]: ../resources/client/i18n.md#components
[keymapping]: ../misc/keymappings.md#inside-a-gui
[modbus]: ../concepts/events.md#event-buses