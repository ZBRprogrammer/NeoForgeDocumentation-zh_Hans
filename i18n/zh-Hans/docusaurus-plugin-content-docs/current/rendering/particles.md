# 客户端粒子 (Client Particles)

粒子 (particles) 是用于完善游戏和增加沉浸感的视觉特效。由于主要具有视觉性质，关键部分仅存在于物理（和逻辑）客户端[端 (side)]。

本文涵盖粒子与渲染相关的方面。有关粒子类型 (particle types) 的更多信息（通常用于生成粒子），以及粒子描述 (particle descriptions)（可以指定粒子的精灵图 (sprites)），请参阅配套的[粒子类型 (particle types)][particletype]文章。

## `Particle` 类

`Particle` 定义了在世界中生成并向玩家显示的客户端表示形式。大多数属性和基础物理由字段控制，如 `gravity`、`lifetime`、`hasPhysics`、`friction` 等。通常需要重写的两个方法是 `tick` 和 `move`，两者都如其名所示。因此，大多数自定义粒子通常都很简短，只包含一个设置所需字段的构造函数，偶尔重写这两个方法。

构造粒子最常用的两种方法是：通过继承 `SingleQuadParticle` 或其某个实现（例如 `SimpleAnimatedParticle`），它会在屏幕上绘制一个面向视口的纹理 (look-facing texture)；或者直接继承 `Particle`，这可以完全控制为渲染而提交的[特性 (features)]。

## 单四边形粒子 (A Single Quad)

继承 `SingleQuadParticle` 的粒子会向屏幕绘制一个带有某个图集精灵 (atlas sprite) 的四边形 (quad)。该类提供了许多辅助方法，从设置粒子大小（通过 `quadSize` 字段或 `scale` 方法），到为纹理着色（通过 `setColor` 和 `setAlpha`）。然而，关于四边形粒子最重要的两点是：用作纹理的 `TextureAtlasSprite`，以及通过 `SingleQuadParticle.Layer` 获取和渲染该精灵图的位置。

首先，`TextureAtlasSprite` 被传入构造函数，可以是其本身，或者更常见的是一个 `SpriteSet`，代表其生命周期中的纹理。初始时，精灵图被设置到受保护的 `sprite` 字段，但可以在 `tick` 期间通过调用 `setSprite` 或 `setSpriteFromAge` 来更新。

:::tip
如果在粒子构造函数中更新了 `age` 或 `lifetime` 字段，则应调用 `setSpriteFromAge` 以显示适当的纹理。
:::

然后，在[特性提交过程 (feature submission process)][features]中，`SingleQuadParticle.Layer` 确定使用哪个图集以及用于将四边形绘制到屏幕的渲染管线 (pipeline)。默认情况下，Vanilla 提供了三个层：

| 层 (Layer) | 纹理图集 (Texture Atlas) | 用于 (For) |
|:-------------:|:-------------:|:----------------------------------|
| `TERRAIN` | 方块 (Blocks) | 使用方块纹理的粒子 |
| `OPAQUE` | 粒子 (Particles) | 没有透明度的粒子 |
| `TRANSLUCENT` | 粒子 (Particles) | 有透明度的粒子 |

可以通过调用构造函数轻松创建自定义层。

```java
public class MyQuadParticle extends SingleQuadParticle {

    public static final SingleQuadParticle.Layer EXAMPLE_LAYER = new SingleQuadParticle.Layer(
        // 粒子是否使用非完全不透明的纹理。
        true,
        // 用于获取精灵图的纹理图集。
        // 这应该与 `TextureAtlasSprite#atlasLocation` 匹配。
        TextureAtlas.LOCATION_PARTICLES,
        // 用于绘制粒子的渲染管线。
        // 自定义渲染管线应基于 `RenderPipelines#PARTICLE_SNIPPET`
        // 来指定可用的 uniforms 和 samplers。
        RenderPipelines.WEATHER_DEPTH_WRITE
    );

    private final SpriteSet spriteSet;

    // 前四个参数是不言自明的。
    // 精灵图集或图集精灵通常通过提供者 (provider) 给出，见下文。
    // 可以根据需要添加其他参数，例如 xSpeed/ySpeed/zSpeed。
    public MyQuadParticle(ClientLevel level, double x, double y, double z, SpriteSet spriteSet) {
        // 在构造函数中初始设置精灵图集
        super(level, x, y, z, spriteSet.first());
        this.spriteSet = spriteSet;
        this.gravity = 0; // 我们的粒子现在漂浮在半空中，为什么不呢。
    }

    @Override
    public void tick() {
        // 让父类处理移动。
        // 如果需要，你可以用你自己的移动逻辑替换它。
        // 如果只想修改内置的移动逻辑，你也可以重写 move()。
        super.tick();

        // 为当前粒子年龄设置精灵图，即推进动画。
        this.setSpriteFromAge(this.spriteSet);
    }

    @Override
    protected abstract SingleQuadParticle.Layer getLayer() {
        // 设置用于获取和提交纹理的层。
        return EXAMPLE_LAYER;
    }
}
```

:::warning
其 `SingleQuadParticle.Layer` 使用 `TextureAtlas#LOCATION_PARTICLES` 的粒子必须有关联的[粒子描述 (particle description)][description]。否则，粒子所需的纹理将不会被添加到图集中。
:::

## `ParticleProvider`

为某个粒子类型创建好粒子后，必须通过 `ParticleProvider` 将粒子类型链接起来。`ParticleProvider` 是一个仅客户端的类，负责通过 `createParticle` 方法从 `ParticleEngine` 实际创建我们的 `Particle`s。虽然这里可以包含更复杂的代码，但许多粒子提供者都像这样简单：

```java
// ParticleProvider 的泛型类型必须与此提供者所针对的粒子类型相匹配。
public class MyQuadParticleProvider implements ParticleProvider<SimpleParticleType> {

    // 一组粒子精灵图。
    private final SpriteSet spriteSet;

    // 注册函数传递一个 SpriteSet，所以我们接收并存储它以供进一步使用。
    // 如果你的粒子不需要 SpriteSet，则可以省略此构造函数。
    public MyParticleProvider(SpriteSet spriteSet) {
        this.spriteSet = spriteSet;
    }

    // 魔法发生的地方。每次调用此方法时，我们都返回一个新粒子！
    // 第一个参数的类型与传递给超接口的泛型类型匹配。
    @Override
    @Nullable
    public Particle createParticle(SimpleParticleType particleType, ClientLevel level, double x, double y, double z, double xd, double yd, double zd, RandomSource random
    ) {
        // 我们不使用类型、速度增量或引擎随机数。
        return new MyQuadParticle(level, x, y, z, this.spriteSet);
    }
}
```

然后，你的粒子提供者必须在[客户端 (client-side)][side]的[模组总线 (mod bus)][modbus][事件 (event)] `RegisterParticleProvidersEvent` 中与粒子类型关联：

```java
@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerParticleProviders(RegisterParticleProvidersEvent event) {
    // 有多种注册提供者的方式，它们的不同之处在于第二个参数提供的函数类型。
    // 例如，#registerSpriteSet 表示一个 Function<SpriteSet, ParticleProvider<?>>：
    event.registerSpriteSet(MyParticleTypes.MY_QUAD_PARTICLE.get(), MyQuadParticleProvider::new);

    // 另一方面，#registerSpecial 映射到一个 ParticleProvider<?>。
    // 如果精灵图不是从粒子描述中获取的，则应使用此方法。
}
```

:::warning
如果使用了 `registerSpriteSet`，则粒子类型也必须有关联的[粒子描述 (particle description)][description]。否则，将抛出异常，指出“无法加载描述 (Failed to load description)”。
:::

[description]: ../resources/client/particles.md
[event]: ../concepts/events.md
[features]: feature.md
[modbus]: ../concepts/events.md#event-buses
[particletype]: ../resources/client/particles.md
[registry]: ../concepts/registries.md#methods-for-registering
[side]: ../concepts/sides.md