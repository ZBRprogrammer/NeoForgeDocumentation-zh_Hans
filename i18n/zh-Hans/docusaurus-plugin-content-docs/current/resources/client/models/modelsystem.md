# 理解模型系统(Understanding the Model System)

Minecraft内的模型仅仅是具有附加纹理的四边形列表。建模过程的每个部分都有自己单独的实现，底层模型JSON被反序列化为`UnbakedModel`。最终，管道的每个部分都接收一些`List<BakedQuad>`及其管道所需的属性。一些[方块实体渲染器][ber]也使用这些模型。模型的复杂程度没有限制。

模型存储在`ModelManager`中，可以通过`Minecraft.getInstance().getModelManager()`访问。对于物品管道，您可以通过传递一个[`ResourceLocation`][rl]通过`ModelManager#getItemModel`获取关联的[`ItemModel`][itemmodels]。对于方块状态管道，您可以通过传递一个`BlockState`通过`ModelManager.getBlockModelShaper().getBlockModel()`获取关联的`BlockStateModel`。模组基本上总是重用先前自动加载和烘焙的模型。

## 通用模型和几何体(Common Models and Geometry)

基本模型JSON（在`assets/<namespace>/models`中）被反序列化为`UnbakedModel`。`UnbakedModel`通常距离其烘焙输出仅一步之遥，包含一些一般属性的通用形式。它包含的最重要的东西是通过`UnbakedModel#geometry`的`UnbakedGeometry`，它表示将成为`BakedQuad`s的数据。这些四边形通过（最终）调用`UnbakedGeometry#bake`内联到物品和方块状态模型中。这通常构造一个`QuadCollection`，其中包含可以在任何时候渲染或仅在给定方向未被剔除时渲染的`BakedQuad`s列表。现在，四边形与建模程序（以及大多数其他游戏）中的三角形相比，然而由于Minecraft通常专注于正方形，开发人员选择在Minecraft中使用四边形（4个顶点）而不是三角形（3个顶点）进行渲染。

`UnbakedModel`包含由[方块状态定义][bsd]、[物品模型][itemmodelsection]或两者使用的信息。例如，`useAmbientOcclusion`仅由方块状态定义使用，`guiLight`和`transforms`仅由物品模型使用，而`textureSlots`和`parent`由两者使用。

在烘焙过程中，每个`UnbakedModel`都被包装在一个`ResolvedModel`中，该模型由物品或方块状态的`ModelBaker`获取。顾名思义，`ResolvedModel`是一个`UnbakedModel`，其所有悬而未决的引用都已解析。然后可以从`getTop*`方法获取关联的数据，这些方法从当前模型及其父级计算属性和几何体。通过调用`ResolvedModel#bakeTopGeometry`将`ResolvedModel`烘焙到其`QuadCollection`通常在这里完成。

## 方块状态定义(Block State Definitions)

方块状态定义JSON（在`assets/<namespace>/blockstates`中）被编译并烘焙成每个`BlockState`的`BlockStateModel`。创建`BlockStateModel`的过程如下：

- 在加载过程中：
    - 方块状态定义JSON被加载到`BlockStateModel.UnbakedRoot`中。根是一个通用的共享缓存系统，用于将`BlockState`链接到某些`BlockStateModel`s集合。
    - `BlockStateModel.UnbakedRoot`加载`BlockStateModel.Unbaked`并准备将它们链接到适当的`BlockState`。
    - `BlockStateModel.Unbaked`加载其`BlockModelPart.Unbaked`，用于获取通用的`UnbakedModel`（更具体地说是`ResolvedModel`）。
- 在烘焙过程中：
    - 为每个`BlockState`调用`BlockStateModel.UnbakedRoot#bake`。
    - 为给定的`BlockState`调用`BlockStateModel.Unbaked#bake`，创建`BlockStateModel`。
    - 为`BlockStateModel`内的模型部分调用`BlockModelPart.Unbaked#bake`，将`ResolvedModel`内联为`QuadCollection`，同时默认获取环境光遮蔽设置、粒子图标和渲染类型。

`BlockStateModel`内最重要的方法是`collectParts`，它负责返回要渲染的`BlockModelPart`s列表。请记住，每个`BlockModelPart`通过`BlockModelPart#getQuads`包含其`BakedQuad`s列表，然后上传到顶点消费者并渲染。`collectParts`有四个参数：

- 一个`BlockAndTintGetter`：表示`BlockState`渲染所在的维度。
- 一个`BlockPos`：方块渲染的位置。
- 一个`BlockState`：正在渲染的[方块状态]。可能为null，表示正在渲染物品。
- 一个`RandomSource`：您可用于随机化的客户端绑定随机源。

### 模型数据(Model Data)

有时，`BlockStateModel`可能依赖`BlockEntity`来确定在`collectParts`中选择哪些`BlockModelPart`s。NeoForge提供`ModelData`系统来同步和传递来自`BlockEntity`的数据。为此，`BlockEntity`必须实现`getModelData`并返回要同步的数据。然后可以通过调用`BlockEntity#requestModelDataUpdate`将数据发送到客户端。然后，在`collectParts`内，可以使用`BlockPos`在`BlockAndTintGetter`上调用`getModelData`以获取数据。

## 物品模型(Item Models)

[客户端物品][clientitem] JSON（在`assets/<namespace>/items`中）被编译并烘焙成给定`Item`的`ItemModel`，供`ItemStack`使用。创建`ItemModel`的过程如下：

- 在加载过程中：
    - 客户端物品JSON被加载到`ClientItem`中。这包含物品模型和一些关于应如何渲染的一般属性。
    - `ClientItem`加载`ItemModel.Unbaked`。
- 在烘焙过程中：
    - 为每个`Item`调用`ItemModel.Unbaked#bake`，将`ResolvedModel`内联为`List<BakedQuad>`，以及一些通用的`ModelRenderProperties`，如果`Item`是`BlockItem`，还包括渲染类型。

有关物品渲染的信息，请参见[手动渲染物品][itemmodels]部分。

### 视角(Perspectives)

Minecraft的渲染引擎识别总共8种视角类型（如果包括代码中的回退，则为9种）用于物品渲染。这些在模型JSON的`display`块中使用，并在代码中通过`ItemDisplayContext`枚举表示。这些通常从`UnbakedModel`传递到`ItemModel`中的`ModelRenderProperties`，然后通过`ModelRenderProperties#applyToLayer`应用于`ItemStackRenderState`。

| 枚举值                | JSON键                  | 用途                                                                                                            |
|---------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------|
| `THIRD_PERSON_RIGHT_HAND` | `"thirdperson_righthand"` | 第三人称中的右手（F5视角，或其他玩家）                                                        |
| `THIRD_PERSON_LEFT_HAND`  | `"thirdperson_lefthand"`  | 第三人称中的左手（F5视角，或其他玩家）                                                         |
| `FIRST_PERSON_RIGHT_HAND` | `"firstperson_righthand"` | 第一人称中的右手                                                                                       |
| `FIRST_PERSON_LEFT_HAND`  | `"firstperson_lefthand"`  | 第一人称中的左手                                                                                        |
| `HEAD`                    | `"head"`                  | 在玩家的头部盔甲槽中时（通常只能通过命令实现）                                          |
| `GUI`                     | `"gui"`                   | 库存，玩家快捷栏                                                                                       |
| `GROUND`                  | `"ground"`                | 掉落物品；请注意，掉落物品的旋转由掉落物品渲染器处理，而不是模型 |
| `FIXED`                   | `"fixed"`                 | 物品展示框                                                                                                      |
| `ON_SHELF`                | `"on_shelf"`              | 在书架方块上                                                                                                      |
| `NONE`                    | `"none"`                  | 代码中的回退目的，不应在JSON中使用                                                            |

NeoForge允许[扩展]`ItemDisplayContext`以用于自定义渲染调用。模组的`ItemDisplayContext`s可以指定在模型中未指定时使用的回退变换。否则，行为将与原版相同。

## 修改烘焙结果(Modifying a Baking Result)

在代码中修改现有的方块状态模型或物品堆栈模型通常可以通过将模型包装在某种委托中来完成。方块状态模型有`DelegateBlockStateModel`，而物品堆栈模型没有现有的实现。您的实现可以仅覆盖选定的方法，如下所示：

```java
// 对于方块状态
public class MyDelegateBlockStateModel extends DelegateBlockStateModel {
    // 将原始模型传递给super。
    public MyDelegateBlockStateModel(BlockStateModel originalModel) {
        super(originalModel);
    }
    
    // 在此处覆盖您想要的任何方法。如果需要，您也可以访问originalModel。
}

// 对于物品模型
public class MyDelegateItemModel implements ItemModel {

    private final ItemModel originalModel;

    public MyDelegateItemModel(ItemModel originalModel) {
        this.originalModel = originalModel;
    }

    // 在此处覆盖您想要的任何方法。如果需要，您也可以访问originalModel。
    @Override
    public void update(ItemStackRenderState renderState, ItemStack stack, ItemModelResolver resolver, ItemDisplayContext displayContext, @Nullable ClientLevel level, @Nullable ItemOwner owner, int seed
    ) {
        this.originalModel.update(renderState, stack, resolver, displayContext, level, owner, seed);
    }
}
```

编写模型包装类后，必须将包装器应用于它应影响的模型。在[客户端][sides]的[事件处理程序][event]中执行此操作，用于[**模组事件总线**][modbus]上的`ModelEvent.ModifyBakingResult`：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void modifyBakingResult(ModelEvent.ModifyBakingResult event) {
    // 对于方块状态模型
    event.getBakingResult().blockStateModels().computeIfPresent(
        // 要修改的模型的方块状态。
        MyBlocksClass.EXAMPLE_BLOCK.get().defaultBlockState(),
        // 具有位置和原始模型作为参数的BiFunction，返回新模型。
        (location, model) -> new MyDelegateBakedModel(model);
    );

    // 对于物品模型
    event.getBakingResult().itemStackModels().computeIfPresent(
        // 要修改的模型的资源位置。
        // 通常是物品注册名称；但是，由于ITEM_MODEL数据组件，可以是任何内容
        MyItemsClass.EXAMPLE_ITEM.getKey().location(),
        // 具有位置和原始模型作为参数的BiFunction，返回新模型。
        (location, model) -> new MyDelegateItemModel(model);
    );
}
```

:::warning
通常建议尽可能使用[自定义模型加载器][modelloader]而不是在`ModelEvent.ModifyBakingResult`中包装烘焙模型。如果需要，自定义模型加载器也可以使用委托模型。
:::

[ao]: https://en.wikipedia.org/wiki/Ambient_occlusion
[ber]: ../../../blockentities/ber.md
[blockstate]: ../../../blocks/states.md
[bsd]: #block-state-definitions
[clientitem]: items.md
[event]: ../../../concepts/events.md
[extended]: ../../../advanced/extensibleenums.md#creating-an-enum-entry
[itemmodels]: items.md#manually-rendering-an-item
[itemmodelsection]: #item-models
[livingentity]: ../../../entities/livingentity.md
[modbus]: ../../../concepts/events.md#event-buses
[modelloader]: modelloaders.md
[rl]: ../../../misc/resourcelocation.md
[perspective]: #perspectives
[rendertype]: index.md#render-types
[sides]: ../../../concepts/sides.md