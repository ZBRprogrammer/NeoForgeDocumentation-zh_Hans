# 自定义模型加载器(Custom Model Loaders)

模型只是一个形状。它可以是一个立方体、立方体的集合、三角形的集合或任何其他几何形状（或几何形状的集合）。对于大多数上下文，模型如何定义并不相关，因为最终一切都会烘焙成`QuadCollection`。因此，NeoForge添加了注册自定义模型加载器的能力，可以将您想要的任何模型转换为游戏使用的烘焙格式。

## 模型加载器(Model Loaders)

方块模型的入口点仍然是模型JSON文件。但是，您可以在JSON的根目录中指定一个`loader`字段，该字段将用您自己的加载器替换默认加载器。自定义模型加载器可能会忽略默认加载器需要的所有字段。

除了默认模型加载器，NeoForge还提供了几个内置加载器，每个加载器服务于不同的目的。

### 复合模型(Composite Model)

复合模型可用于在父模型中指定不同的模型部分，并且仅在子模型中应用其中一些部分。这最好通过示例来说明。考虑`examplemod:example_composite_model`处的以下父模型：

```json5
{
    "loader": "neoforge:composite",
    // 指定模型部分。
    "children": {
        // 这些可以是另一个模型的引用或模型本身。
        "part_1": {
            "parent": "examplemod:some_model_1"
        },
        "part_2": {
            "parent": "examplemod:some_model_2"
        }
    },
    "visibility": {
        // 默认禁用第2部分。
        "part_2": false
    }
}
```

然后，我们可以在`examplemod:example_composite_model`的子模型中禁用和启用各个部分：

```json5
{
    "parent": "examplemod:example_composite_model",
    // 覆盖可见性。如果缺少某个部分，它将使用父模型的可见性值。
    "visibility": {
        "part_1": false,
        "part_2": true
    }
}
```

要[数据生成][modeldatagen]此模型，请使用自定义加载器类`CompositeModelBuilder`。

:::warning
复合模型加载器不应用于[客户端物品][citems]使用的模型。相反，它们应使用定义本身中提供的[复合模型][itemcomposite]。
:::

### 空模型(Empty Model)

空模型什么都不渲染。

```json5
{
    "loader": "neoforge:empty"
}
```

### OBJ模型(OBJ Model)

OBJ模型加载器允许您在游戏中使用Wavefront `.obj` 3D模型，允许在模型中包含任意形状（包括三角形、圆形等）。`.obj`模型必须放在`models`文件夹（或其子文件夹）中，并且必须提供具有相同名称的`.mtl`文件（或手动设置），因此，例如，位于`models/block/example.obj`的OBJ模型必须有一个对应的MTL文件位于`models/block/example.mtl`。

```json5
{
    "loader": "neoforge:obj",
    // 必需。对模型文件的引用。请注意，这是相对于命名空间根目录，而不是模型文件夹。
    "model": "examplemod:models/example.obj",
    // 通常，.mtl文件必须放在与.obj文件相同的位置，只有文件扩展名不同。
    // 这将导致加载器自动拾取它们。但是，如果需要，您也可以手动设置
    // .mtl文件的位置。
    "mtl_override": "examplemod:models/example_other_name.mtl",
    // 这些纹理可以在.mtl文件中引用为#texture0、#particle等。
    // 这通常需要手动编辑.mtl文件。
    "textures": {
        "texture0": "minecraft:block/cobblestone",
        "particle": "minecraft:block/stone"
    },
    // 启用或禁用模型的自动剔除。可选，默认为true。
    "automatic_culling": false,
    // 是否对模型着色。可选，默认为true。
    "shade_quads": false,
    // 一些建模程序会假设V=0是底部而不是顶部。此属性将V上下翻转。
    // 可选，默认为false。
    "flip_v": true,
    // 是否启用自发光。可选，默认为true。
    "emissive_ambient": false
}
```

要[数据生成][modeldatagen]此模型，请使用自定义加载器类`ObjModelBuilder`。

### 创建自定义模型加载器(Creating Custom Model Loaders)

要创建您自己的模型加载器，您需要四个类，加上一个事件处理程序：

- 一个`UnbakedModelLoader`类
- 一个`UnbakedGeometry`类，通常是`ExtendedUnbakedGeometry`实例
- 一个`UnbakedModel`类，通常是`AbstractUnbakedModel`实例
- 一个`QuadCollection`类来保存烘焙的四边形，通常是类本身
- 一个[客户端][sides]的[事件处理程序][event]，用于注册未烘焙模型加载器的`ModelEvent.RegisterLoaders`
- 可选：一个[客户端][sides]的[事件处理程序][event]，用于缓存正在加载的数据的模型加载器的`AddClientReloadListenersEvent`

为了说明这些类如何连接，我们将跟踪一个模型被加载的过程：

- 在模型加载期间，具有`loader`属性设置为您的加载器的模型JSON传递给您的未烘焙模型加载器。加载器然后读取模型JSON，并使用模型JSON的属性以及具有模型未烘焙四边形的`UnbakedGeometry`返回一个`UnbakedModel`对象。
- 在模型烘焙期间，调用`UnbakedGeometry#bake`，返回一个`QuadCollection`。
- 在模型渲染期间，使用`QuadCollection`以及[客户端物品][citems]或[方块状态定义][blockstatedefinition]所需的任何其他信息进行渲染。

:::note
如果您正在为物品或方块状态使用的模型创建自定义模型加载器，根据用例，创建新的`ItemModel`或`BlockStateModel`可能更好。例如，使用或生成`QuadCollection`s的模型更适合作为`ItemModel`或`BlockStateModel`，而解析不同数据格式（如`.obj`）的模型应使用新的模型加载器。
:::

让我们通过一个基本的类设置进一步说明这一点。加载器类名为`MyUnbakedModelLoader`，未烘焙类名为`MyUnbakedModel`，未烘焙几何体名为`MyUnbakedGeometry`。我们还将假设模型加载器需要一些缓存：

```java
// 这是用于将模型加载到其未烘焙格式的类
public class MyUnbakedModelLoader implements UnbakedModelLoader<MyUnbakedModel>, ResourceManagerReloadListener {
    // 强烈建议对未烘焙模型加载器使用单例模式，因为所有模型都可以通过一个加载器加载。
    public static final MyUnbakedModelLoader INSTANCE = new MyUnbakedModelLoader();
    // 我们将用于注册此加载器的ID。也用于加载器数据生成类。
    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_loader");

    // 根据单例模式，将构造函数设为私有。        
    private MyUnbakedModelLoader() {}

    @Override
    public void onResourceManagerReload(ResourceManager resourceManager) {
        // 处理任何缓存清除逻辑
    }

    @Override
    public MyUnbakedModel read(JsonObject obj, JsonDeserializationContext context) throws JsonParseException {
        // 使用给定的JsonObject，如果需要，使用JsonDeserializationContext从模型JSON获取属性。
        // MyUnbakedModel构造函数可能有构造函数参数（参见下文）。

        // 读取用于创建四边形的数据
        MyUnbakedGeometry geometry;

        // 对于由原版和NeoForge提供的基本参数，您可以使用StandardModelParameters
        StandardModelParameters params = StandardModelParameters.parse(obj, context);

        return new MyUnbakedModel(params, geometry);
    }
}

// 保存要渲染的未烘焙四边形
// 存储在未烘焙模型中的其他信息应传递到上下文映射
public class MyUnbakedGeometry implements ExtendedUnbakedGeometry {

    public MyUnbakedGeometry(...) {
        // 存储要烘焙的未烘焙四边形
    }

    // 负责模型烘焙的方法，返回四边形集合。此方法中的参数是：
    // - 纹理名称到其关联材质的映射。
    // - 模型烘焙器。可用于获取要烘焙的子模型并从纹理槽获取精灵。
    // - 模型状态。这保存来自方块状态文件的变换，通常来自旋转和uvlock。
    // - 模型名称。
    // - 由NeoForge和您的未烘焙模型提供的设置的ContextMap。有关所有可用属性，请参见'NeoForgeModelProperties'类。
    @Override
    public QuadCollection bake(TextureSlots textureSlots, ModelBaker baker, ModelState state, ModelDebugName debugName, ContextMap additionalProperties) {
        // 创建集合的构建器
        var builder = new QuadCollection.Builder();
        // 构建用于烘焙的四边形
        builder.addUnculledFace(...); // 或addCulledFace(Direction, BakedQuad)
        // 创建四边形集合
        return builder.build();
    }
}

// 未烘焙模型包含从JSON读取的所有信息。
// 它提供基本设置和几何体。
// 使用AbstractUnbakedModel设置原版和NeoForge属性方法
public class MyUnbakedModel extends AbstractUnbakedModel {

    private final MyUnbakedGeometry geometry;

    public MyUnbakedModel(StandardModelParameters params, MyUnbakedGeometry geometry) {
        super(params);
        this.geometry = geometry;
    }

    @Override
    public UnbakedGeometry geometry() {
        // 用于构造烘焙四边形的几何体
        return this.geometry;
    }

    @Override
    public void fillAdditionalProperties(ContextMap.Builder propertiesBuilder) {
        super.fillAdditionalProperties(propertiesBuilder);
        // 通过调用withParameter(ContextKey<T>, T)在下面添加其他属性
        // 然后可以在UnbakedGeometry#bake中提供的ContextMap中访问它们
    }
}
```

完成后，不要忘记实际注册您的加载器：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerLoaders(ModelEvent.RegisterLoaders event) {
    event.register(MyUnbakedModelLoader.ID, MyUnbakedModelLoader.INSTANCE);
}

// 如果您在模型加载器中缓存数据：
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void addClientResourceListeners(AddClientReloadListenersEvent event) {
    // 使用我们的ID注册监听器
    event.addListener(MyUnbakedModelLoader.ID, MyUnbakedModelLoader.INSTANCE);
    // 为我们的模型加载器添加依赖项，以便在模型加载之前运行
    // 允许在新数据填充之前清除缓存
    event.addDependency(MyUnbakedModelLoader.ID, VanillaClientListeners.MODELS);
}
```

#### 模型加载器数据生成(Model Loader Datagen)

当然，我们也可以[数据生成]我们的模型。为此，我们需要一个扩展`CustomLoaderBuilder`的类：

```java
public class MyLoaderBuilder extends CustomLoaderBuilder {
    public MyLoaderBuilder() {
        super(
            // 您的模型加载器的ID。
            MyUnbakedModelLoader.ID,
            // 加载器是否允许内联原版元素作为加载器缺失时的回退。
            false
        );
    }
    
    // 在此处添加字段和设置器。然后可以在下面使用这些字段。

    @Override
    protected CustomLoaderBuilder copyInternal() {
        // 创建您的加载器构建器的新实例，并将属性从此构建器复制到新实例。
        MyLoaderBuilder builder = new MyLoaderBuilder();
        // builder.<field> = this.<field>;
        return builder;
    }
    
    // 将模型序列化为JSON。
    @Override
    public JsonObject toJson(JsonObject json) {
        // 将您的字段添加到给定的JsonObject。
        // 然后调用super，它添加加载器属性和其他一些东西。
        return super.toJson(json);
    }
}
```

要使用此加载器构建器，请在方块（或物品）[模型数据生成][modeldatagen]期间执行以下操作：

```java
// 这假设是ModelProvider的扩展和一个DeferredBlock<Block> EXAMPLE_BLOCK。
// customLoader()的参数是构造构建器的Supplier和设置相关属性的Consumer。
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    blockModels.createTrivialBlock(
        // 为其生成模型的方块
        EXAMPLE_BLOCK.get(),
        TexturedModel.createDefault(
            // 用于获取纹理的映射
            block -> new TextureMapping().put(
                TextureSlot.ALL, TextureMapping.getBlockTexture(block)
            ),
            // 用于创建JSON的模型模板构建器
            ExtendedModelTemplateBuilder.builder()
                // 假设我们使用自定义模型加载器
                .customLoader(MyLoaderBuilder::new, loader -> {
                    // 在此处设置任何必需的字段
                })
                // 模型所需的纹理
                .requiredTextureSlot(TextureSlot.ALL)
                // 完成后调用build
                .build()
        )
    );
}
```

#### 可见性(Visibility)

`CustomLoaderBuilder`的默认实现包含应用可见性的方法。您可以选择在模型加载器中使用或忽略`visibility`属性。目前，只有[复合模型加载器][composite]和[OBJ加载器][obj]使用此属性。

## 方块状态模型加载器(Block State Model Loaders)

由于方块状态模型被认为与模型JSON文件分开，因此也有自定义的NeoForge加载器，通过在变体或多部分中指定`type`来处理。自定义方块状态模型加载器可能会忽略加载器需要的所有字段。

### 复合方块状态模型(Composite Block State Model)

复合方块状态模型可用于一起渲染多个`BlockStateModel`s。

```json5
{
    "variants": {
        "": {
            "type": "neoforge:composite",
            // 指定模型部分。
            "models": [
                // 这些必须是内联的方块状态模型
                {
                    "variants": {
                        // ...
                    }
                },
                {
                    "multipart": [
                        // ...
                    ]
                }
                // ...
            ]
        }
    }
}
```

要[数据生成][modeldatagen]此方块状态模型，请使用自定义加载器类`CompositeBlockStateModelBuilder`。

### 重用默认模型加载器(Reusing the Default Model Loader)

在某些上下文中，重用原版模型加载器并在其基础上构建模型逻辑，而不是完全替换它，是有意义的。我们可以使用一个巧妙的技巧：在模型加载器中，我们只需删除`loader`属性并将其发送回模型反序列化器，欺骗它认为它是一个常规的未烘焙模型。然后，我们可以在烘焙过程之前修改模型或其几何体，在那里我们可以做任何我们想要的方式。

```java
public class MyUnbakedModelLoader implements UnbakedModelLoader<MyUnbakedModel> {
    public static final MyUnbakedModelLoader INSTANCE = new MyUnbakedModelLoader();
    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_loader");
    
    private MyUnbakedModelLoader() {}

    @Override
    public MyUnbakedModel read(JsonObject jsonObject, JsonDeserializationContext context) throws JsonParseException {
        // 通过移除加载器字段来欺骗反序列化器认为这是一个普通模型
        // 然后将其传递给反序列化器。
        jsonObject.remove("loader");
        UnbakedModel model = context.deserialize(jsonObject, UnbakedModel.class);
        return new MyUnbakedModel(model, /* 其他参数在这里 */);
    }
}

// 我们扩展委托类，因为它存储了包装的模型
public class MyUnbakedModel extends DelegateUnbakedModel {

    // 存储模型供下文使用
    public MyUnbakedModel(UnbakedModel model, /* 其他参数在这里 */) {
       super(model);
    }
}
```

### 创建自定义方块状态模型加载器(Creating Custom Block State Model Loaders)

要创建您自己的方块状态模型加载器，您需要五个类，加上一个事件处理程序：

- 一个`CustomUnbakedBlockStateModel`类来加载方块状态模型
- 一个`BlockStateModel`类来烘焙模型，通常是`DynamicBlockStateModel`实例
- 一个`BlockModelPart.Unbaked`来加载模型JSON
- 一个`ModelState`来将任何变换应用于给定面或模型
- 一个`BlockModelPart`来保存四边形、环境光遮蔽和粒子纹理，通常是`SimpleModelWrapper`
- 一个[客户端][sides]的[事件处理程序][event]，用于注册未烘焙方块状态模型加载器编解码器的`RegisterBlockStateModels`

为了说明这些类如何连接，我们将跟踪一个方块状态模型被加载的过程：

- 在定义加载期间，变体、多部分或[自定义定义][customdefinition]内具有`type`属性设置为您的加载器的方块状态模型被解码到您的`CustomUnbakedBlockStateModel`。
- 在模型烘焙期间，为每个`BlockState`调用`CustomUnbakedBlockStateModel#bake`。
- 在模型烘焙期间，为给定`BlockState`调用`CustomUnbakedBlockStateModel#bake`，创建一个`BlockStateModel`。
- 在模型烘焙期间，为`BlockStateModel`内的模型部分调用`BlockModelPart.Unbaked#bake`，将`ResolvedModel`内联为`QuadCollection`，同时获取环境光遮蔽设置、粒子图标和默认渲染类型。

最重要的方法在`BlockStateModel`内是`collectParts`，它负责返回要渲染的`BlockModelPart`s列表。请记住，每个`BlockModelPart`通过`BlockModelPart#getQuads`包含其`BakedQuad`s列表，然后上传到顶点消费者并渲染。`collectParts`有四个参数：

- 一个`BlockAndTintGetter`：表示`BlockState`渲染所在的维度。
- 一个`BlockPos`：方块渲染的位置。
- 一个`BlockState`：正在渲染的[方块状态]。可能为null，表示正在渲染物品。
- 一个`RandomSource`：您可以用于随机化的客户端绑定随机源。

让我们通过一个基本的类设置进一步说明这一点。烘焙模型名为`MyBlockStateModel`，未烘焙类是一个内部记录`MyBlockStateModel.Unbaked`，模型部分名为`MyBlockModelPart`，未烘焙部分类是一个内部记录`MyBlockModelPart.Unbaked`，`ModelState`名为`MyModelState`：

```java
// 用于应用必要变换的模型状态
// 如果您使用中间对象来保存模型状态，它必须可转换为ModelState
public class MyModelState implements ModelState {

    // 用于未烘焙方块模型部分
    public static final Codec<MyModelState> CODEC = Codec.unit(new MyModelState());

    public MyModelState() {}

    @Override
    public Transformation transformation() {
        // 返回应用于烘焙顶点的模型旋转
        return Transformation.identity();
    }

    @Override
    public Matrix4fc faceTransformation(Direction direction) {
        // 返回在变换之后应用于模型上给定面的矩阵
        // 目前原版中未使用
        return NO_TRANSFORM;
    }

    @Override
    public Matrix4fc inverseFaceTransformation(Direction direction) {
        // 返回faceTransformation的逆矩阵，应用于模型上的给定面
        // 这会传递给FaceBakery
        return NO_TRANSFORM;
    }
}

// 表示烘焙模型的模型部分
// useAmbientOcclusion和particleIcon作为记录的一部分实现
public record MyBlockModelPart(QuadCollection quads, boolean useAmbientOcclusion, TextureAtlasSprite particleIcon) implements BlockModelPart {

    // 获取要渲染的烘焙四边形
    @Override
    List<BakedQuad> getQuads(@Nullable Direction direction) {
        return this.quads.getQuads(direction);
    }

    // 从方块状态json读取的未烘焙模型
    public record Unbaked(ResourceLocation modelLocation, MyModelState modelState) implements BlockModelPart.Unbaked {

        // 用于未烘焙方块状态模型
        public static final MapCodec<MyBlockModelPart.Unbaked> CODEC = RecordCodecBuilder.mapCodec(
            instance -> instance.group(
                ResourceLocation.CODEC.fieldOf("model").forGetter(MyBlockModelPart.Unbaked::modelLocation),
                MyModelState.CODEC.fieldOf("state").forGetter(MyBlockModelPart.Unbaked::modelState)
            ).apply(instance, MyBlockModelPart.Unbaked::new)
        );

        @Override
        public void resolveDependencies(ResolvableModel.Resolver resolver) {
            // 标记模型部分使用的任何模型
            resolver.markDependency(this.modelLocation);
        }

        @Override
        public BlockModelPart bake(ModelBaker baker) {
            // 获取要烘焙的模型
            ResolvedModel resolvedModel = baker.getModel(this.modelLocation);

            // 获取模型部分的必要设置
            TextureSlots slots = resolvedModel.getTopTextureSlots();
            boolean ao = resolvedModel.getTopAmbientOcclusion();
            TextureAtlasSprite particle = resolvedModel.resolveParticleSprite(slots, baker);
            QuadCollection quads = resolvedModel.bakeTopGeometry(slots, baker, this.modelState);
            
            // 返回烘焙部分
            return new MyBlockModelPart(quads, ao, particle);
        }
    }
}

// 表示烘焙方块状态的状态模型
public record MyBlockStateModel(MyBlockModelPart model) implements DynamicBlockStateModel {

    // 设置粒子图标
    // 虽然需要实现，但任何实际逻辑应委托给感知维度的版本
    @Override
    public TextureAtlasSprite particleIcon() {
        return this.model.particleIcon();
    }

    // 这有效地充当重用先前生成的几何体的键。这通常应尽可能确定。
    @Override
    public Object createGeometryKey(BlockAndTintGetter level, BlockPos pos, BlockState state, RandomSource random) {
        return this;
    }

    // 负责收集要渲染的部分的方法。此方法中的参数是：
    // - 方块和着色的获取器，通常是维度。
    // - 要渲染的方块位置。
    // - 方块的状态。
    // - 随机实例。
    // - 要渲染的模型部分列表。在此处添加您的模型部分。
    @Override
    public void collectParts(BlockAndTintGetter level, BlockPos pos, BlockState state, RandomSource random, List<BlockModelPart> parts) {
        // 如果您希望渲染的方块依赖于方块实体（例如，您的方块实体实现`BlockEntity#getModelData`）
        // 您可以使用方块位置调用`BlockAndTintGetter#getModelData`
        // 您可以使用`ModelProperty`键的`get`读取属性
        // 请记住，您的方块实体应调用`BlockEntity#requestModelDataUpdate`以将模型数据同步到客户端
        ModelData data = level.getModelData(pos);

        // 添加要渲染的模型
        parts.add(this.model);
    }

    @Override
    public TextureAtlasSprite particleIcon(BlockAndTintGetter level, BlockPos pos, BlockState state) {
        // 如果您想使用维度来确定渲染什么粒子，请重写此方法
        return self().particleIcon();
    }

    // 从方块状态json读取的未烘焙模型
    public record Unbaked(MyBlockModelPart.Unbaked model) implements CustomUnbakedBlockStateModel {

        // 要注册的编解码器
        public static final MapCodec<MyBlockStateModel.Unbaked> CODEC = MyBlockModelPart.Unbaked.CODEC.xmap(
            MyBlockStateModel.Unbaked::new, MyBlockStateModel.Unbaked::model
        );
        public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_model_loader");

        @Override
        public void resolveDependencies(ResolvableModel.Resolver resolver) {
            // 标记状态模型使用的任何模型
            this.model.resolveDependencies(resolver);
        }

        @Override
        public BlockStateModel bake(ModelBaker baker) {
            // 烘焙模型部分并传递到方块状态模型
            return new MyBlockStateModel(this.model.bake(baker));
        }
    }
}
```

完成后，不要忘记实际注册您的加载器：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerDefinitions(RegisterBlockStateModels event) {
    event.registerModel(MyBlockStateModel.Unbaked.ID, MyBlockStateModel.Unbaked.CODEC);
}
```

#### 状态模型加载器数据生成(State Model Loader Datagen)

当然，我们也可以[数据生成]我们的模型。为此，我们需要一个扩展`CustomBlockStateModelBuilder`的类：

```java
// 用于构造方块状态JSON的构建器
public class MyBlockStateModelBuilder extends CustomBlockStateModelBuilder {

    private MyBlockModelPart.Unbaked model;

    public MyBlockStateModelBuilder() {}
    
    // 在此处添加字段和设置器。然后可以在下面使用这些字段。

    @Override
    public MyBlockStateModelBuilder with(VariantMutator variantMutator) {
        // 如果您想应用任何假设您的未烘焙模型部分是`Variant`的变换器
        // 如果不是，这应该什么都不做
        return this;
    }

    // 这是用于通用的未烘焙方块状态模型
    @Override
    public MyBlockStateModelBuilder with(UnbakedMutator unbakedMutator) {
        var result = new MyBlockStateModelBuilder();

        if (this.model != null) {
            result.model = unbakedMutator.apply(this.model);
        }

        return result;
    }

    // 将构建器转换为其未烘焙变体以进行编码
    @Override
    public CustomUnbakedBlockStateModel toUnbaked() {
        return new MyBlockStateModel.Unbaked(this.model);
    }
}
```

要使用此状态定义加载器构建器，请在方块（或物品）[模型数据生成][modeldatagen]期间执行以下操作：

```java
// 这假设是ModelProvider的扩展和一个DeferredBlock<Block> EXAMPLE_BLOCK。
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    blockModels.blockStateOutput.accept(
        MultiVariantGenerator.dispatch(
            // 为其生成模型的方块
            EXAMPLE_BLOCK.get(),
            // 我们的自定义方块状态构建器
            MultiVariant.of(new CustomBlockStateModelBuilder().with(...))
        )
    );
}
```

这将生成如下模型：

```json5
{
  "variants": {
    "": {
        "type": "examplemod:my_custom_model_loader"
        // 其他字段
    }
  }
}
```

## 方块状态定义加载器(Block State Definition Loaders)

虽然单个方块状态模型处理单个方块状态的加载，但方块状态定义加载器处理整个方块状态文件的加载，通过在`neoforge:definition_type`中指定来处理。自定义方块状态定义加载器可能会忽略加载器需要的所有字段。

### 创建自定义方块状态定义加载器(Creating Custom Block State Definition Loaders)

要创建您自己的方块状态定义加载器，您需要两个类，加上一个事件处理程序：

- 一个`CustomBlockModelDefinition`类来加载方块状态定义
- 一个`BlockStateModel.UnbakedRoot`类来将方块状态烘焙到其`BlockStateModel`
- 一个[客户端][sides]的[事件处理程序][event]，用于注册未烘焙方块状态模型加载器编解码器的`RegisterBlockStateModels`

为了说明这些类如何连接，我们将跟踪一个方块状态模型被加载的过程：

- 在定义加载期间，具有`neoforge:definition_type`属性设置为您的加载器的方块状态定义被解码为`CustomBlockModelDefinition`。
- 然后，调用`CustomBlockModelDefinition#instantiate`以将所有可能的方块状态映射到其`BlockStateModel.UnbakedRoot`。对于简单情况，这是通过`BlockStateModel.Unbaked#asRoot`构造的。复杂情况创建自己的`BlockStateModel.UnbakedRoot`。
- 在模型烘焙期间，为某个`BlockState`调用`BlockStateModel.UnbakedRoot#bake`，返回一个`BlockStateModel`。

让我们通过一个基本的类设置进一步说明这一点。方块模型定义名为`MyBlockModelDefinition`，我们将重用`BlockStateModel.Unbaked#asRoot`来构造`BlockStateModel.UnbakedRoot`：

```java
public record MyBlockModelDefinition(MyBlockStateModel.Unbaked model) implements CustomBlockModelDefinition {

    // 要注册的编解码器
    public static final MapCodec<MyBlockModelDefinition> CODEC = MyBlockStateModel.Unbaked.CODEC.xmap(
        MyBlockModelDefinition::new, MyBlockModelDefinition::model
    );
    public static final ResourceLocation ID = ResourceLocation.fromNamespaceAndPath("examplemod", "my_custom_definition_loader");

    // 此方法将所有可能的状态映射到某个未烘焙根
    // 由于根通常会共享方块状态模型，它们通常使用`ModelBaker.SharedOperationKey`来缓存加载模型
    @Override
    public Map<BlockState, BlockStateModel.UnbakedRoot> instantiate(StateDefinition<Block, BlockState> states, Supplier<String> sourceSupplier) {
        Map<BlockState, BlockStateModel.UnbakedRoot> result = new HashMap<>();

        // 处理所有可能的状态
        var unbakedRoot = this.model.asRoot();
        states.getPossibleStates().forEach(state -> result.put(state, unbakedRoot));

        return result;
    }

    @Override
    public MapCodec<? extends CustomBlockModelDefinition> codec() {
        return CODEC;
    }
}
```

完成后，不要忘记实际注册您的加载器：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerDefinitions(RegisterBlockStateModels event) {
    event.registerDefinition(MyBlockModelDefinition.ID, MyBlockModelDefinition.CODEC);
}
```

#### 状态定义加载器数据生成(State Definition Loader Datagen)

当然，我们也可以[数据生成]我们的定义。为此，我们需要一个实现`BlockModelDefinitionGenerator`的类：

```java
public class MyBlockModelDefinitionGenerator implements BlockModelDefinitionGenerator {

    private final Block block;
    private final MyBlockStateModelBuilder builder;

    private MyBlockModelDefinitionGenerator(Block block, MyBlockStateModelBuilder builder) {
        this.block = block;
        this.builder = builder;
    }

    public static MyBlockModelDefinitionGenerator dispatch(Block block, MyBlockStateModelBuilder builder) {
        return new MyBlockModelDefinitionGenerator(block, builder);
    }

    @Override
    public Block block() {
        // 返回您为其生成定义文件的方块
        return this.block;
    }

    @Override
    public BlockModelDefinition create() {
        // 创建用于编码和解码文件的方块模型定义
        return new MyBlockModelDefinition(this.builder.toUnbaked());
    }
} 
```

要使用此状态定义加载器构建器，请在方块（或物品）[模型数据生成][modeldatagen]期间执行以下操作：

```java
// 这假设一个DeferredBlock<Block> EXAMPLE_BLOCK。
@Override
protected void registerModels(BlockModelGenerators blockModels, ItemModelGenerators itemModels) {
    blockModels.blockStateOutput.accept(
        MyBlockModelDefinitionGenerator.dispatch(
            // 为其生成模型的方块
            EXAMPLE_BLOCK.get(),
            new CustomBlockStateModelBuilder(...)
        )
    );
}
```

这将生成如下模型：

```json5
{
    "neoforge:definition_type": "examplemod:my_custom_definition_loader"
    // 其他字段
}
```

[citems]: items.md
[composite]: #composite-model
[customdefinition]: #block-state-definition-loaders
[datagen]: ../../index.md#data-generation
[event]: ../../../concepts/events.md#registering-an-event-handler
[itemcomposite]: items.md#composite-models
[modeldatagen]: datagen.md
[obj]: #obj-model
[sides]: ../../../concepts/sides.md