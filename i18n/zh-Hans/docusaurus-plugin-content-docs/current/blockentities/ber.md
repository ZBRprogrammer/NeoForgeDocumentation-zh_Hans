# 方块实体渲染器(BlockEntityRenderer)

`BlockEntityRenderer`，通常缩写为BER，用于以[静态烘焙模型][model]（JSON、OBJ等）无法表示的方式“渲染”[方块][block]。例如，这可用于动态渲染类似箱子的方块的容器内容。方块实体渲染器要求方块具有[`BlockEntity`][blockentity]，即使该方块不存储任何其他数据。

BER直接实现`BlockEntityRenderer`接口，并提交其[特性(features)]进行渲染：

``` java
// The generic type in the superinterface should be set to what block entity
// you are trying to render, along with its extracted render state. More on this below.
public class MyBlockEntityRenderer implements BlockEntityRenderer<MyBlockEntity, MyBlockEntityRenderState> {

    public MyBlockEntityRenderer(BlockEntityRendererProvider.Context context) {
        // Get whatever is necessary from the context
    }

    // Tell the renderer how to create a new render state.
    @Override
    public MyBlockEntityRenderState createRenderState() {
        return new MyBlockEntityRenderState();
    }

    // Update the render state by copying the needed values from the passed block entity
    // to the passed render state.
    // The block entity and render state are the generic types passed to the renderer
    @Override
    public void extractRenderState(MyBlockEntity blockEntity, MyBlockEntityRenderState renderState, float partialTick, Vec3 cameraPos, @Nullable ModelFeatureRenderer.CrumblingOverlay crumblingOverlay) {
        // Always call super or `BlockEntityRenderState#extractBase`
        super.extractRenderState(blockEntity, renderState, partialTick, cameraPos, crumblingOverlay);

        // Extract and store any additional values in the state here.
        renderState.value = blockEntity.getValue();
    }

    // Actually submit the features of the block entity to render.
    // The first parameter matches the render state's generic type.
    @Override
    public void submit(MyBlockEntityRenderState renderState, PoseStack poseStack, SubmitNodeCollector collector, CameraRenderState cameraState) {
        // Submit using the collector here.
    }
}
```

现在我们有了BER，还需要将其注册并连接到其所属的方块实体。这是在[`EntityRenderersEvent.RegisterRenderers`][event]中完成的，如下所示：

``` java
@SubscribeEvent // on the mod event bus only on the physical client
public static void registerEntityRenderers(EntityRenderersEvent.RegisterRenderers event) {
    event.registerBlockEntityRenderer(
            // The block entity type to register the renderer for.
            MyBlockEntities.MY_BLOCK_ENTITY.get(),
            // A function of BlockEntityRendererProvider.Context to BlockEntityRenderer.
            MyBlockEntityRenderer::new
    );
}
```

:::note

如果您在BER中不需要提供者上下文，也可以移除构造函数：

``` java
public class MyBlockEntityRenderer implements BlockEntityRenderer<MyBlockEntity, MyBlockEntityRenderState> {
    
    // ...
}

// In some event handler class
@SubscribeEvent // on the mod event bus only on the physical client
public static void registerEntityRenderers(EntityRenderersEvent.RegisterRenderers event) {
    event.registerBlockEntityRenderer(MyBlockEntities.MY_BLOCK_ENTITY.get(),
            // Pass the context to an empty (default) constructor call
            context -> new MyBlockEntityRenderer()
    );
}
```

:::

## 方块实体渲染状态(Block Entity Render States)

如上例所述，方块实体渲染状态用于从实际方块实体的值中提取用于渲染的值。它们在功能上是可变的、从`BlockEntityRenderState`扩展而来的数据存储对象：

``` java
public class MyBlockEntityRenderState extends BlockEntityRenderState {
    public boolean value;
}
```

然后应在`BlockEntityRenderer#extractRenderState`中从`BlockEntity`子类填充这些值。

## 物品方块渲染(Item Block Rendering)

由于并非所有带有渲染器的方块实体都可以用静态物品模型表示，因此可以创建特殊的渲染器来更动态地控制该过程。这是使用[`SpecialModelRenderer`s][special]完成的。在这些情况下，必须创建一个特殊模型渲染器来提交所需的[特性(features)]，并为方块本身被提交渲染而不是物品变体（例如，末影人携带方块）的场景注册相应的特殊方块模型渲染器。

请参阅[客户端物品文档][special]以获取更多信息。

[block]: ../blocks/index.md
[blockentity]: index.md
[event]: ../concepts/events.md#registering-an-event-handler
[features]: ../rendering/feature.md
[item]: ../items/index.md
[model]: ../resources/client/models/index.md
[special]: ../resources/client/models/items.md#special-models