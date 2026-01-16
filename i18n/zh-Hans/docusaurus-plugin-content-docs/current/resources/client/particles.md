# 粒子 (Particles)

粒子 (particles) 是通常使用其关联的粒子类型生成的视觉特效。它们可以在客户端和服务器[端 (side)]生成，但由于主要具有视觉性质，关键部分仅存在于物理（和逻辑）客户端端。

本文涵盖粒子类型和粒子描述的构建和使用。有关更多与渲染相关的信息，请参阅配套的[客户端粒子 (client particles)][clientparticle]文章。

## 注册 `ParticleType`s

粒子使用 `ParticleType`s 注册。它们的工作方式类似于 `EntityType`s 或 `BlockEntityType`s，即有一个 `Particle` 类——每个生成的粒子都是该类的实例——然后是 `ParticleType` 类，保存一些通用信息，用于注册。`ParticleType`s 是一个[注册表 (registry)]，这意味着我们想使用 `DeferredRegister` 来注册它们，就像所有其他注册对象一样：

```java
public class MyParticleTypes {
    // 假设您的模组 ID 是 examplemod
    public static final DeferredRegister<ParticleType<?>> PARTICLE_TYPES =
        DeferredRegister.create(BuiltInRegistries.PARTICLE_TYPE, "examplemod");
    
    // 添加新粒子类型的最简单方法是重用 Vanilla 的 SimpleParticleType。
    // 也可以实现自定义 ParticleType，见下文。
    public static final Supplier<SimpleParticleType> MY_QUAD_PARTICLE = PARTICLE_TYPES.register(
        // 粒子类型的名称。
        "my_quad_particle",
        // 供应器。布尔参数表示在视频设置中将粒子选项设置为“最少 (Minimal)”时是否会影响此粒子类型；
        // 对于大多数 Vanilla 粒子，这是 false，但对于例如爆炸、篝火烟雾或墨鱼墨汁，这是 true。
        () -> new SimpleParticleType(false)
    );
}
```

:::info
仅在需要在服务器端处理粒子时才需要 `ParticleType`。客户端也可以直接使用 `Particle`s。
:::

## 自定义 `ParticleType`s

虽然对于大多数情况 `SimpleParticleType` 足够，但有时需要在服务器端为粒子附加额外数据。这就是需要自定义 `ParticleType` 和关联的自定义 `ParticleOptions` 的地方。让我们从 `ParticleOptions` 开始，因为这是实际存储信息的地方：

```java
public class MyParticleOptions implements ParticleOptions {
    
    // 定义粒子的额外信息的映射编解码器 (map codec)，用于例如命令中。
    // 由于我们的类型中没有信息，使用单位映射编解码器 (unit map codec)；
    // 这对应于在命令中使用空字符串。
    public static final MapCodec<MyParticleOptions> CODEC = MapCodec.unit(new MyParticleOptions());

    // 向网络缓冲区读取和写入信息。
    public static final StreamCodec<ByteBuf, MyParticleOptions> STREAM_CODEC = StreamCodec.unit(new MyParticleOptions());

    // 不需要任何参数，但可以定义粒子工作所需的任何字段。
    public MyParticleOptions() {}

    @Override
    public ParticleType<?> getType() {
        // 返回注册的粒子类型
    }
}
```

然后我们在自定义的 `ParticleType` 中使用这个 `ParticleOptions` 实现...

```java
public class MyParticleType extends ParticleType<MyParticleOptions> {
    // 布尔参数再次决定是否在较低的粒子设置下限制粒子。
    // 有关更多信息，请参阅本文顶部附近的 MyParticleTypes 类的实现。
    public MyParticleType(boolean overrideLimiter) {
        // 将反序列化器传递给父类。
        super(overrideLimiter);
    }

    @Override
    public MapCodec<MyParticleOptions> codec() {
        return MyParticleOptions.CODEC;
    }

    @Override
    public StreamCodec<? super RegistryFriendlyByteBuf, MyParticleOptions> streamCodec() {
        return MyParticleOptions.STREAM_CODEC;
    }
}
```

... 并在[注册 (registration)][registry]期间引用它：

```java
public static final Supplier<MyParticleType> MY_CUSTOM_PARTICLE = PARTICLE_TYPES.register(
    "my_custom_particle",
    () -> new MyParticleType(false)
);
```

然后将注册的粒子传递给 `ParticleOptions#getType`：

```java
public class MyParticleOptions implements ParticleOptions {
    
    // ...

    @Override
    public ParticleType<?> getType() {
        return MY_CUSTOM_PARTICLE.get();
    }
}
```

## 粒子描述 (Particle Descriptions)

粒子描述是位于 `assets/<namespace>/particles` 目录中的 JSON 文件。粒子描述的名称与其关联的[粒子类型 (particle type)][particletype] 相同，并包含一个相对于 `assets/<namespace>/textures/particles` 的纹理列表。

粒子描述看起来像这样：

```json5
{
    // 将按顺序播放的纹理列表。如果需要会循环。
    // 纹理位置相对于 textures/particle 文件夹。
    "textures": [
        // 指向 `assets/examplemod/textures/particle/my_particle_0.png`
        "examplemod:my_particle_0",
        "examplemod:my_particle_1",
        "examplemod:my_particle_2",
        "examplemod:my_particle_3"
    ]
}
```

在资源重载期间，`ParticleResources` 加载所有粒子描述并将纹理拼接到 `TextureAtlas#LOCATION_PARTICLES` 图集中。然后，为每个描述创建一个 `SpriteSet`，其中包含指定的 `TextureAtlasSprite`s 列表。

### 使用描述

为了让[粒子 (particle)]能够使用其描述，必须通过[客户端 (client-side)][side]的[模组总线 (mod bus)][modbus][事件 (event)] `RegisterParticleProvidersEvent` 将 `ParticleType` 与一个接受 `SpriteSet` 的 [`ParticleProvider`][provider] 关联：

```java
public class MyParticleProvider implements ParticleProvider<SimpleParticleType> {

    private final SpriteSet spriteSet;

    // 接受 `ParticleResources` 提供的精灵图集。
    public MyParticleProvider(SpriteSet spriteSet) {
        this.spriteSet = spriteSet;
    }

    // ...
}

// 在某个仅客户端的事件处理器中

@SubscribeEvent // 仅在物理客户端的模组事件总线上
public static void registerParticleProviders(RegisterParticleProvidersEvent event) {
    // 当处理粒子描述时，必须使用 #registerSpriteSet。
    event.registerSpriteSet(MyParticleTypes.MY_PARTICLE.get(), MyParticleProvider::new);
}
```

:::warning
如果为没有通过 `RegisterParticleProvidersEvent#registerSpriteSet` 关联 `ParticleProvider` 的粒子类型创建了粒子描述，则会记录“冗余纹理列表 (Redundant texture list)”消息。
:::

### 数据生成 (Datagen)

粒子定义文件也可以[数据生成 (datagenned)][datagen]，通过扩展 `ParticleDescriptionProvider` 并重写 `#addDescriptions()` 方法：

```java
public class MyParticleDescriptionProvider extends ParticleDescriptionProvider {
    // 从 `GatherDataEvent.Client` 获取参数。
    public MyParticleDescriptionProvider(PackOutput output) {
        super(output);
    }

    // 假设所有引用的粒子实际上都存在。将 "examplemod" 替换为您的模组 ID。
    @Override
    protected void addDescriptions() {
        // 添加一个单精灵图粒子定义，文件位于
        // assets/examplemod/textures/particle/my_single_particle.png。
        spriteSet(MyParticleTypes.MY_SINGLE_PARTICLE.get(), ResourceLocation.fromNamespaceAndPath("examplemod", "my_single_particle"));
        // 添加一个多精灵图粒子定义，带有可变参数。也可以接受可迭代对象。
        spriteSet(MyParticleTypes.MY_MULTI_PARTICLE.get(),
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle_0"),
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle_1"),
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle_2")
        );
        // 上述的替代方法，为给定数量的纹理将 "_<索引>" 附加到给定的基本名称。
        spriteSet(MyParticleTypes.MY_ALT_MULTI_PARTICLE.get(),
            // 基本名称。
            ResourceLocation.fromNamespaceAndPath("examplemod", "my_multi_particle"),
            // 纹理数量。
            3,
            // 是否反转列表，即从最后一个元素开始而不是第一个。
            false
        );
    }
}
```

不要忘记将提供者添加到 `GatherDataEvent.Client`：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(MyParticleDescriptionProvider::new);
}
```

## 生成粒子 (Spawning Particles)

提醒一下，服务器只知道 [`ParticleType`s][particletype] 和 [`ParticleOption`s][options]，而客户端直接使用由与 `ParticleType` 关联的 `ParticleProvider`s 提供的 `Particle`s。因此，生成粒子的方式在您所在的端有很大不同。

- **通用代码 (Common code)**：调用 `Level#addParticle` 或 `Level#addAlwaysVisibleParticle`。这是创建对所有人可见的粒子的首选方法。
- **客户端代码 (Client code)**：使用通用代码方式。或者，使用您选择的粒子类创建 `new Particle()` 并用该粒子调用 `Minecraft.getInstance().particleEngine#add(Particle)`。请注意，以这种方式添加的粒子仅显示给客户端，因此其他玩家不可见。
- **服务器代码 (Server code)**：调用 `ServerLevel#sendParticles`。Vanilla 中由 `/particle` 命令使用。

[clientparticle]: ../../rendering/particles.md
[datagen]: ../index.md#data-generation
[event]: ../../concepts/events.md
[modbus]: ../../concepts/events.md#event-buses
[options]: #custom-particletypes
[particle]: ../../rendering/particles.md
[particletype]: #registering-particletypes
[provider]: ../../rendering/particles.md#particleprovider
[side]: ../../concepts/sides.md