# 模型数据生成(Model Datagen)

与大多数JSON数据一样，方块和物品模型及其必要的方块状态文件和[客户端物品][citems]都可以进行[数据生成][datagen]。这一切都由原版的`ModelProvider`处理，NeoForge通过`ExtendedModelTemplateBuilder`提供扩展。由于方块和物品模型的模型JSON本身相似，数据生成代码也相对类似。

## 模型模板(Model Templates)

每个模型都从一个`ModelTemplate`开始。对于原版，`ModelTemplate`作为某些预生成模型文件的父级，定义父模型、必需的纹理槽以及要应用的文件后缀。在NeoForge的情况下，`ExtendedModelTemplate`通过`ExtendedModelTemplateBuilder`构建，允许用户生成模型到底层元素和面，以及任何NeoForge添加的功能。

`ModelTemplate`通过使用`ModelTemplates`中的方法之一或调用构造函数来创建。对于构造函数，它接受可选的父模型`ResourceLocation`（相对于`models`目录）、要应用于文件路径末尾的可选字符串（例如，对于按下的按钮，后缀为`_pressed`）以及必须在数据生成时定义以防止崩溃的`TextureSlot`可变参数。`TextureSlot`只是一个定义`textures`映射中纹理“键”的字符串。每个键还可以有一个父`TextureSlot`，如果未为特定槽指定纹理，则将解析到该父槽。例如，`TextureSlot#PARTICLE`将首先查找已定义的`particle`纹理，然后检查已定义的`texture`值，最后检查`all`。如果槽或其父级未定义，则会在数据生成期间抛出崩溃。

```java
// 假设存在引用为'#base'的纹理
// 可以通过指定'base'或'all'来解析
public static final TextureSlot BASE = TextureSlot.create("base", TextureSlot.ALL);

// 假设存在模型'examplemod:block/example_template'
public static final ModelTemplate EXAMPLE_TEMPLATE = new ModelTemplate(
    // 父模型位置
    Optional.of(
        ModelLocationUtils.decorateBlockModelLocation("examplemod:example_template")
    ),
    // 应用于使用此模板的任何模型末尾的后缀
    Optional.of("_example"),
    // 必须定义的所有纹理槽
    // 应根据父模型中未定义的内容尽可能具体
    TextureSlot.PARTICLE,
    BASE
);
```

NeoForge添加的`ExtendedModelTemplate`可以通过`ExtendedModelTemplateBuilder#builder`或现有原版模板的`ModelTemplate#extend`构建。然后，构建器可以使用`#build`解析为模板。构建器的方法提供对模型JSON构建的完全控制：

| 方法                                           | 效果                                                                                                                                                                                                                                                                                                                                                  |
|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `#parent(ResourceLocation parent)`                                         | 设置相对于`models`目录的父模型位置。 |
| `#suffix(String suffix)`                                                   | 将字符串附加到模型文件路径的末尾。 |
| `#requiredTextureSlot(TextureSlot slot)`                                   | 添加必须在`TextureMapping`中定义以供生成的纹理槽。 |
| `#renderType(ResourceLocation renderType)`                                 | 设置渲染类型。有一个参数为`String`的重载。有关有效值列表，请参阅`RenderType`类。                                                                                                                                                                                                                        |
| `transform(ItemDisplayContext type, Consumer<TransformVecBuilder> action)` | 添加通过消费者配置的`TransformVecBuilder`，用于在模型上设置`display`。 |
| `#ambientOcclusion(boolean ambientOcclusion)`                              | 设置是否使用[环境光遮蔽][ao]。                                                                                                                                                                                                                                                                                                     |
| `#guiLight(UnbakedModel.GuiLight light)`                                   | 设置GUI光照。可以是`GuiLight.FRONT`或`GuiLight.SIDE`。                                                                                                                                                                                                                                                                                         |
| `#element(Consumer<ElementBuilder> action)`                                | 添加一个新的`ElementBuilder`（相当于向模型添加新的[元素][elements]），通过消费者配置。                                                                                                                                                                                                 |                                                                                                                                                                                                                                                            |
| `#customLoader(Supplier customLoaderFactory, Consumer action)`            | 使用给定的工厂，使此模型使用[自定义加载器][custommodelloader]，从而使用通过消费者配置的自定义加载器构建器。这会更改构建器类型，因此可能使用不同的方法，具体取决于加载器的实现。NeoForge默认提供了一些自定义加载器，有关更多信息（包括数据生成），请参阅链接文章。 |
| `#rootTransforms(Consumer<RootTransformsBuilder> action)`                  | 通过消费者配置模型的变换，以在物品显示和方块状态变换之前应用。 |

:::tip
虽然可以通过数据生成创建精细复杂的模型，但建议使用建模软件（如[Blockbench][blockbench]）创建更复杂的模型，然后使用导出的模型，直接使用或作为其他模型的父级。
:::

### 创建模型实例

现在我们有了`ModelTemplate`，我们可以通过调用`ModelTemplate#create*`方法之一来生成模型本身。尽管每个创建方法接受不同的参数，但核心上，它们都接受表示文件名的`ResourceLocation`、将`TextureSlot`映射到相对于`textures`目录的某个`ResourceLocation`的`TextureMapping`，以及作为`BiConsumer<ResourceLocation, ModelInstance>`的模型输出。然后，该方法实质上创建用于生成模型的`JsonObject`，如果提供任何重复项则抛出错误。

:::note
调用基础的`create`方法不会应用存储的后缀。只有接受方块或物品的`create*`方法才会这样做。
:::

```java
// 给定某个BiConsumer<ResourceLocation, ModelInstance> modelOutput
// 假设有一个DeferredBlock<Block> EXAMPLE_BLOCK
EXAMPLE_TEMPLATE.create(
    // 在'assets/minecraft/models/block/example_block_example.json'处创建模型
    EXAMPLE_BLOCK.get(),
    // 在槽中定义纹理
    new TextureMapping()
        // "particle": "examplemod:item/example_block"
        .put(TextureSlot.PARTICLE, TextureMapping.getBlockTexture(EXAMPLE_BLOCK.get()))
        // "base": "examplemod:item/example_block_base"
        .put(TextureSlot.BASE, TextureMapping.getBlockTexture(EXAMPLE_BLOCK.get(), "_base")),
    // 生成的模型json的消费者
    modelOutput
);
```

有时，生成的模型对其纹理使用相似的模型模板和命名模式（例如，普通方块的纹理只是方块的名称）。在这些情况下，可以创建`TexturedModel.Provider`来帮助消除冗余。提供器实际上是一个函数式接口，它接受某个`Block`并返回`TexturedModel`（一个`ModelTemplate`/`TextureMapping`对）来生成模型。该接口通过`TexturedModel#createDefault`构造，该函数接受将`Block`映射到其`TextureMapping`的函数以及要使用的`ModelTemplate`。然后，可以通过使用要生成的`Block`调用`TexturedModel.Provider#create`来生成模型。

```java
public static final TexturedModel.Provider EXAMPLE_TEMPLATE_PROVIDER = TexturedModel.createDefault(
    // 方块到纹理映射
    block -> new TextureMapping()
        .put(TextureSlot.PARTICLE, TextureMapping.getBlockTexture(block))
        .put(TextureSlot.BASE, TextureMapping.getBlockTexture(block, "_base")),
    // 用于生成的模板
    EXAMPLE_TEMPLATE
);

// 给定某个BiConsumer<ResourceLocation, ModelInstance> modelOutput
// 假设有一个DeferredBlock<Block> EXAMPLE_BLOCK
EXAMPLE_TEMPLATE_PROVIDER.create(
    // 在'assets/minecraft/models/block/example_block_example.json'处创建模型
    EXAMPLE_BLOCK.get(),
    // 生成的模型json的消费者
    modelOutput
);
```

## `模型提供器(ModelProvider)`

方块和物品模型数据生成都利用`registerModels`提供的生成器，分别命名为`BlockModelGenerators`和`ItemModelGenerators`。每个生成器都会生成模型JSON以及任何其他必需的文件（方块状态、客户端物品）。每个生成器包含各种辅助方法，这些方法将所有文件的构建批处理为单个易于使用的方法，例如`ItemModelGenerators#generateFlatItem`创建基本的`item/generated`模型，或`BlockModelGenerators#createTrivialCube`创建基本的`block/cube_all`模型。

```java
public class ExampleModelProvider extends ModelProvider {

    public ExampleModelProvider(PackOutput output) {
        // 将"examplemod"替换为您自己的模组ID。
        super(output, "examplemod");
    }

    @Override
    protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
        // 在此生成模型和相关文件
    }
}
```

与所有数据提供器一样，不要忘记将您的提供器注册到事件：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(ExampleModelProvider::new);
}
```

### 方块模型数据生成

现在，要实际生成方块状态和方块模型文件，您可以在`ModelProvider#registerModels`内调用`BlockModelGenerators`中的众多公共方法之一，或将生成的文件自己传递给方块状态文件的`blockStateOutput`、非普通客户端物品的`itemModelOutput`以及模型JSON的`modelOutput`。

:::note
如果您为方块注册了关联的`BlockItem`且没有生成的客户端物品，`ModelProvider`将使用默认方块模型位置`assets/<namespace>/models/block/<path>.json`作为其模型自动生成客户端物品。
:::

```java
public class ExampleModelProvider extends ModelProvider {

    public ExampleModelProvider(PackOutput output) {
        // 将"examplemod"替换为您自己的模组ID。
        super(output, "examplemod");
    }

    @Override
    protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
        // 占位符，其用法应替换为实际值。有关如何使用模型构建器，请参见上文；
        // 有关模型构建器提供的辅助方法，请参见下文。
        Block block = MyBlocksClass.EXAMPLE_BLOCK.get();

        // 创建一个每面纹理相同的简单方块模型。
        // 纹理必须位于assets/<namespace>/textures/block/<path>.png，其中
        // <namespace>和<path>分别是方块注册名称的命名空间和路径。
        // 大多数（完整）方块使用此方法，例如木板、圆石或砖块。
        blockModels.createTrivialCube(block);

        // 接受要使用的`TexturedModel.Provider`的重载。
        blockModels.createTrivialBlock(block, EXAMPLE_TEMPLATE_PROVIDER);

        // 方块物品会自动生成模型
        // 但假设您想生成不同的物品，例如平面物品
        blockModels.registerSimpleFlatItemModel(block);

        // 添加一个原木方块模型。需要在assets/<namespace>/textures/block/<path>.png和
        // assets/<namespace>/textures/block/<path>_top.png处有两个纹理，分别引用侧面和顶部纹理。
        // 请注意，此处的方块输入限制为RotatedPillarBlock，这是原版原木使用的类。
        blockModels.woodProvider(block).log(block);
        
        // 类似于WoodProvider#logWithHorizontal。用于石英柱和类似方块。
        blockModels.createRotatedPillarWithHorizontalVariant(block, TexturedModel.COLUMN_ALT, TexturedModel.COLUMN_HORIZONTAL_ALT);

        // 使用`ExtendedModelTemplate`指定要使用的渲染类型。
        blockModels.createRotatedPillarWithHorizontalVariant(block,
            TexturedModel.COLUMN_ALT.updateTemplate(template ->
                template.extend().renderType("minecraft:cutout").build()
            ),
            TexturedModel.COLUMN_HORIZONTAL_ALT.updateTemplate(template ->
                template.extend().renderType(this.mcLocation("cutout_mipped")).build()
            )
        );

        // 指定一个具有侧面纹理、前面纹理和顶部纹理的可水平旋转的方块模型。
        // 底部也将使用侧面纹理。如果您不需要前面或顶部纹理，
        // 只需传入侧面纹理两次。用于例如熔炉和类似方块。
        blockModels.createHorizontallyRotatedBlock(
            block,
            TexturedModel.Provider.ORIENTABLE_ONLY_TOP.updateTexture(mapping ->
                mapping.put(TextureSlot.SIDE, this.modLocation("block/example_texture_side"))
                .put(TextureSlot.FRONT, this.modLocation("block/example_texture_front"))
                .put(TextureSlot.TOP, this.modLocation("block/example_texture_top"))
            )
        );

        // 指定一个附着在面上的可水平旋转方块模型，例如用于按钮。
        // 考虑将方块放置在地面和天花板上，并相应旋转它们。
        blockModels.familyWithExistingFullBlock(block).button(block);

        // 创建一个用于方块状态文件的模型
        ResourceLocation modelLoc = TexturedModel.CUBE.create(block, blockModels.modelOutput);

        // 创建一个通用变体进行变换
        Variant variant = new Variant(modelLoc);

        // 基本单变体模型
        blockModels.blockStateOutput.accept(
            MultiVariantGenerator.dispatch(
                block,
                new MultiVariant(
                    WeightedList.of(
                        new Weighted<>(
                            // 设置模型
                            variant
                                // 设置绕x轴和y轴的旋转
                                .with(VariantMutator.X_ROT.withValue(Quadrant.R90))
                                .with(VariantMutator.Y_ROT.withValue(Quadrant.R180))
                                // 设置uvlock
                                .with(VariantMutator.UV_LOCK.withValue(true)),
                            // 设置权重
                            5
                        )
                    )
                )
            )
        );

        // 基于方块状态属性添加一个或多个模型
        blockModels.blockStateOutput.accept(
            MultiVariantGenerator.dispatch(
                block,
                // 创建基本多变体
                BlockModelGenerators.variant(variant)
            ).with(
                // 应用属性分发
                // 将基于提供的变换器改变变体
                PropertyDispatch.modify(BlockStateProperties.AXIS)
                    .select(Direction.Axis.Y, BlockModelGenerators.NOP)
                    .select(Direction.Axis.Z, BlockModelGenerators.X_ROT_90)
                    .select(Direction.Axis.X, BlockModelGenerators.X_ROT_90.then(BlockModelGenerators.Y_ROT_90))
            )
        );

        // 生成一个多部分
        blockModels.blockStateOutput.accept(
            MultiPartGenerator.multiPart(block)
                // 提供基础模型
                .with(BlockModelGenerators.variant(variant))
                // 添加变体出现的条件
                .with(
                    // 添加要应用的条件
                    new CombinedCondition(
                        CombinedCondition.Operation.OR,
                        List.of(
                            // 其中至少一个条件为真
                            BlockModelGenerators.condition().term(BlockStateProperties.FACING, Direction.NORTH, Direction.SOUTH)
                            // 可以根据需要嵌套任意数量的条件或组
                            new CombinedCondition(
                                CombinedCondition.Operation.AND,
                                List.of(
                                    BlockModelGenerators.condition().term(BlockStateProperties.FACING, Direction.NORTH)
                                )
                            )
                        )
                    ),
                    // 提供要改变的变体
                    BlockModelGenerators.variant(variant)
                )
        );
    }
}
```

## 物品模型数据生成

生成物品模型要简单得多，这主要归功于`ItemModelGenerators`和`ItemModelUtils`中的所有辅助方法用于属性信息。与上面类似，您可以在`ModelProvider#registerModels`内调用`ItemModelGenerators`中的众多公共方法之一，或将生成的文件自己传递给非普通客户端物品的`itemModelOutput`以及模型JSON的`modelOutput`。

```java
public class ExampleModelProvider extends ModelProvider {

    public ExampleModelProvider(PackOutput output) {
        // 将"examplemod"替换为您自己的模组ID。
        super(output, "examplemod");
    }

    @Override
    protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
        // 最常见的物品
        // 具有layer0纹理作为物品名称的item/generated
        itemModels.generateFlatItem(MyItemsClass.EXAMPLE_ITEM.get(), ModelTemplates.FLAT_ITEM);

        // 类似弓的物品
        ItemModel.Unbaked bow = ItemModelUtils.plainModel(ModelLocationUtils.getModelLocation(MyItemsClass.EXAMPLE_ITEM.get()));
        ItemModel.Unbaked pullingBow0 = ItemModelUtils.plainModel(this.createFlatItemModel(MyItemsClass.EXAMPLE_ITEM.get(), "_pulling_0", ModelTemplates.BOW));
        ItemModel.Unbaked pullingBow1 = ItemModelUtils.plainModel(this.createFlatItemModel(MyItemsClass.EXAMPLE_ITEM.get(), "_pulling_1", ModelTemplates.BOW));
        ItemModel.Unbaked pullingBow2 = ItemModelUtils.plainModel(this.createFlatItemModel(MyItemsClass.EXAMPLE_ITEM.get(), "_pulling_2", ModelTemplates.BOW));
        this.itemModelOutput.accept(
            MyItemsClass.EXAMPLE_ITEM.get(),
            // 物品的条件模型
            ItemModelUtils.conditional(
                // 检查物品是否正在使用
                ItemModelUtils.isUsingItem(),
                // 当为真时，基于使用持续时间选择模型
                ItemModelUtils.rangeSelect(
                    new UseDuration(false),
                    // 应用于阈值的标量
                    0.05F,
                    pullingBow0,
                    // 阈值0.65
                    ItemModelUtils.override(pullingBow1, 0.65F),
                    // 阈值0.9
                    ItemModelUtils.override(pullingBow2, 0.9F)
                ),
                // 当为假时，使用基础弓模型
                bow
            )
        );
    }
}
```

[ao]: https://en.wikipedia.org/wiki/Ambient_occlusion
[blockbench]: https://www.blockbench.net
[citems]: items.md
[custommodelloader]: modelloaders.md#datagen
[datagen]: ../../index.md#data-generation
[elements]: index.md#elements