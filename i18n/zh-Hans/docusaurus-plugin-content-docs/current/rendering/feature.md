---
sidebar_position: 1
---

# 特性 (Features)

渲染特性 (rendering feature) 定义了一组未烘焙到关卡几何体中的对象，例如实体 (entities)、文本 (text) 和粒子 (particles)。这些对象通常具有动态位置，因此像掉落或手持的方块 (blocks) 和物品 (items) 也属于此类。特性渲染器 (feature renderer) 的目的是为了更好地批处理和排序要渲染到屏幕上的对象。特性渲染器分为两个阶段：提交阶段 (submission phase)，收集所有特性；以及渲染阶段 (rendering phases)，渲染收集到的特性。

## 提交特性 (Submitting Features)

特性提交通常由负责这些对象的底层子系统处理：[`EntityRenderer` 用于实体 (entities)][entities]、[`BlockEntityRenderer` 用于方块实体 (block entities)][blockentities]、[`ParticleGroupRenderState` 用于粒子 (particles)`][particles] 等。每个子系统都提供自己的 `submit` 方法，通常传入对象的一些通用渲染状态 (render state)。然后通过 `SubmitNodeCollector` 提交必要的元素，并存储在 `SubmitNodeCollection` 树状映射 (tree map) 中以供渲染。

通过收集器 (collector) 可以使用以下方法，它们将按最终渲染的顺序执行：

| 方法 (Method) | 描述 (Description) |
|:----------------------:|:-----------------------------------------------------------------------------------------------------------|
| `submitHitbox` | 表示包围盒 (bounding box) 的线框和表示视向向量 (view vector) 的线，通常用于实体。 |
| `submitShadow` | 根据所需半径、位置和不透明度绘制的一系列黑色椭圆。 |
| `submitNameTag` | 文本，按透明度排序。 |
| `submitText` | 文本。 |
| `submitFlame` | 应用于实体的火焰覆盖层。 |
| `submitLeash` | 一个 24 段的平面。 |
| `submitModel` | 带有渲染状态的 `Model`s，按透明度排序。 |
| `submitModelPart` | `ModelPart`s。 |
| `submitBlock` | 带有烘焙光照 (baked lighting) 的 `BlockState`。 |
| `submitMovingBlock` | 带有动态光照 (dynamic lighting) 的 `BlockState`。 |
| `submitBlockModel` | 带有烘焙光照的 `BlockStateModel`。 |
| `submitItem` | 一个解构的 `ItemStackRenderState`。 |
| `submitCustomGeometry` | 一个自定义方法，用于定义上传到给定 `RenderType` 缓冲区的顶点。 |
| `submitParticleGroup` | 用于缓存和写入一批粒子四边形 (quads) 的渲染器。 |

:::warning
提交给收集器的每个元素在方法调用后都应被视为不可变。像 `PoseStack` 这样的一些元素会在那时进行快照 (snapshotted)，以防止任何进一步的修改。
:::

从技术上讲，上面列出的所有方法都是超接口 `OrderedSubmitNodeCollector` 的一部分。这是因为收集器可以将特性分组到“渲染顺序 (orders)”中，每个顺序代表渲染器的一次绘制过程 (pass)。默认情况下，所有特性都在顺序 0 上渲染，这意味着它们将根据下面定义的渲染顺序进行绘制。数字较小的顺序会先渲染，数字较大的顺序后渲染。可以使用 `SubmitNodeCollector#order` 来指定元素绘制的顺序：

```java
// 假设我们有一个 SubmitNodeCollector 收集器

// 这将在顺序 0 上渲染。
collector.submitModel(...);

// 这将在模型之前渲染。
collector.order(-1).submitBlock(...);

// 这将在模型之后渲染。
collector.order(1).submitParticleGroup(...);
```

## 渲染特性 (Rendering Features)

特性渲染通过 `FeatureRenderDispatcher` 的 `renderAllFeatures` 方法处理，该方法渲染保存在包含 `SubmitNodeCollection` 树状映射的 `SubmitNodeStorage` 中的已提交对象。分发器包含一个渲染器类列表，这些类负责渲染给定类型的已提交对象。对于给定的“顺序”，特性按以下顺序渲染：

* 阴影 (Shadows)
* 不透明模型 (Opaque models)，然后是已排序的透明模型 (sorted transparent models)
* 模型部件 (Model parts)
* 实体火焰覆盖层 (Entity flame overlays)
* 已排序的透明名称标签 (Sorted transparent name tags)，然后是不透明名称标签 (opaque name tags)
* 文本 (Text)
* 碰撞箱 (Hitboxes)
* 栓绳 (Leashes)
* 物品 (Items)
* 移动方块、方块，然后是方块模型 (Moving blocks, blocks, and then block models)
* 自定义几何体 (Custom geometry)
* 粒子 (Particles)

特性渲染完成后，`SubmitNodeStorage` 将被清空以供下次使用。特性渲染器每帧可能被调用多次，因为它不仅用于关卡，还用于手持物品和[画中画 GUI 渲染器 (picture-in-picture gui renderer)][gui]。请注意，在这些情况下，调用 `renderAllFeatures` 后会接着调用 `MultiBufferSource.BufferSource#endBatch` 来构建网格 (mesh) 并将其绘制到缓冲区。

[blockentities]: ../blockentities/ber.md#blockentityrenderer
[entities]: ../entities/renderer.md#entity-renderers
[gui]: screens.md#picture-in-picture
[particles]: #TODO