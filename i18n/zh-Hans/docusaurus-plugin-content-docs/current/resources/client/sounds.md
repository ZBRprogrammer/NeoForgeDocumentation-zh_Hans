# 声音 (Sounds)

声音虽然对任何东西都不是必需的，但可以使模组感觉更加细致和生动。Minecraft 为您提供了各种注册和播放声音的方法，本文将对此进行说明。

## 术语 (Terminology)

Minecraft 声音引擎使用各种术语来指代不同的事物：

- **声音事件 (Sound event)**：声音事件是代码中的触发器，告诉声音引擎播放某个声音。`SoundEvent`s 也是您向游戏注册的东西。
- **声音类别 (Sound category)** 或**声音源 (sound source)**：声音类别是声音的粗略分组，可以单独切换。声音选项 GUI 中的滑块代表这些类别：`master`、`block`、`player` 等。在代码中，它们可以在 `SoundSource` 枚举中找到。
- **声音定义 (Sound definition)**：声音事件到一个或多个声音对象的映射，加上一些可选的元数据。声音定义位于命名空间的 [`sounds.json` 文件][soundsjson]中。
- **声音对象 (Sound object)**：一个由声音文件位置加上一些可选元数据组成的 JSON 对象。
- **声音文件 (Sound file)**：磁盘上的声音文件。Minecraft 仅支持 `.ogg` 声音文件。

:::danger
由于 OpenAL（Minecraft 的音频库）的实现，为了使您的声音具有衰减 (attenuation)——即根据玩家距离的远近而变弱或变强——您的声音文件必须是单声道 (mono)（单通道）。立体声 (stereo)（多通道）声音文件将不受衰减影响，并且始终在玩家位置播放，这使它们成为环境声音和背景音乐的理想选择。另请参阅 [MC-146721][bug]。
:::

## 创建 `SoundEvent`s

`SoundEvent`s 是[已注册对象 (registered objects)][registration]，这意味着它们必须通过 `DeferredRegister` 注册到游戏并且是单例的：

```java
public class MySoundsClass {
    // 假设您的模组 ID 是 examplemod
    public static final DeferredRegister<SoundEvent> SOUND_EVENTS =
            DeferredRegister.create(BuiltInRegistries.SOUND_EVENT, "examplemod");
    
    // 所有 Vanilla 声音都使用可变范围事件。
    public static final Holder<SoundEvent> MY_SOUND = SOUND_EVENTS.register(
            "my_sound",
            // 接受注册名
            SoundEvent::createVariableRangeEvent
    );
    
    // 还有一个目前未使用的方法来注册固定范围（= 无衰减）事件：
    public static final Holder<SoundEvent> MY_FIXED_SOUND = SOUND_EVENTS.register(
            "my_fixed_sound",
            // 16 是声音的默认范围。请注意，由于 OpenAL 的限制，
            // 大于 16 的值没有效果，将被限制为 16。
            registryName -> SoundEvent.createFixedRangeEvent(registryName, 16f)
    );
}
```

当然，不要忘记在[模组构造函数 (mod constructor)][modctor]中将您的注册表添加到[模组事件总线 (mod event bus)][modbus]：

```java
public ExampleMod(IEventBus modBus) {
    MySoundsClass.SOUND_EVENTS.register(modBus);
    // 其他内容在此
}
```

瞧，您有了一个声音事件！

## `sounds.json`

_另请参阅：[Minecraft Wiki][mcwiki] 上的 [sounds.json][mcwikisounds]_

现在，要将您的声音事件连接到实际的声音文件，我们需要创建声音定义。一个命名空间的所有声音定义都存储在名为 `sounds.json` 的单个文件中，也称为声音定义文件，直接位于命名空间的根目录中。每个声音定义都是声音事件 ID（例如 `my_sound`）到 JSON 声音对象的映射。请注意，声音事件 ID 不指定命名空间，因为那已经由声音定义文件所在的命名空间决定。示例 `sounds.json` 可能如下所示：

```json5
{
    // 声音事件 "examplemod:my_sound" 的声音定义
    "my_sound": {
        // 声音对象列表。如果包含多个元素，将随机选择一个元素。
        "sounds": [
            // 只有 name 是必需的，所有其他属性都是可选的。
            {
                // 声音文件的位置，相对于命名空间的 sounds 文件夹。
                // 此示例引用位于 assets/examplemod/sounds/sound_1.ogg 的声音。
                "name": "examplemod:sound_1",
                // 可以是 "sound" 或 "event"。"sound" 使 name 引用声音文件。
                // "event" 使 name 引用另一个声音事件。默认为 "sound"。
                "type": "sound",
                // 播放此声音的音量。必须在 0.0 和 1.0 之间（默认）。
                "volume": 0.8,
                // 播放声音的音调值。
                // 必须在 0.0 和 2.0 之间。默认为 1.0。
                "pitch": 1.1,
                // 从声音列表中选择声音时，此声音的权重。默认为 1。
                "weight": 3,
                // 如果为 true，声音将从文件流式传输，而不是一次性加载。
                // 推荐用于长度超过几秒的声音文件。默认为 false。
                "stream": true,
                // 衰减距离的手动覆盖。默认为 16。固定范围声音事件忽略此项。
                "attenuation_distance": 8,
                // 如果为 true，声音将在包加载时加载到内存中，而不是在播放声音时加载。
                // Vanilla 将此用于水下环境音。默认为 false。
                "preload": true
            },
            // { "name": "examplemod:sound_2" } 的快捷方式
            "examplemod:sound_2"
        ]
    },
    "my_fixed_sound": {
        // 可选的。如果为 true，则替换其他资源包中的声音，而不是添加到它们。
        // 有关更多信息，请参阅下面的合并章节。
        "replace": true,
        // 触发此声音事件时显示的字幕的翻译键。
        "subtitle": "examplemod.my_fixed_sound",
        "sounds": [
            "examplemod:sound_1",
            "examplemod:sound_2"
        ]
    }
}
```

### 合并 (Merging)

与大多数其他资源文件不同，`sounds.json` 不会覆盖其下方包中的值。相反，它们被合并在一起，然后作为一个组合的 `sounds.json` 文件进行解释。考虑在两个不同资源包 RP1 和 RP2 的两个 `sounds.json` 文件中定义的声音 `sound_1`、`sound_2`、`sound_3` 和 `sound_4`，其中 RP2 位于 RP1 下方：

RP1 中的 `sounds.json`：

```json5
{
    "sound_1": {
        "sounds": [
            "sound_1"
        ]
    },
    "sound_2": {
        "replace": true,
        "sounds": [
            "sound_2"
        ]
    },
    "sound_3": {
        "sounds": [
            "sound_3"
        ]
    },
    "sound_4": {
        "replace": true,
        "sounds": [
            "sound_4"
        ]
    }
}
```

RP2 中的 `sounds.json`：

```json5
{
    "sound_1": {
        "sounds": [
            "sound_5"
        ]
    },
    "sound_2": {
        "sounds": [
            "sound_6"
        ]
    },
    "sound_3": {
        "replace": true,
        "sounds": [
            "sound_7"
        ]
    },
    "sound_4": {
        "replace": true,
        "sounds": [
            "sound_8"
        ]
    }
}
```

然后游戏将使用来加载声音的组合（合并）`sounds.json` 文件将如下所示（仅在内存中，此文件从未写入任何地方）：

```json5
{
    "sound_1": {
        // 上层包中 replace false 和下层包中 false：先添加下层包，然后添加上层包
        "sounds": [
            "sound_5",
            "sound_1"
        ]
    },
    "sound_2": {
        // 上层包中 replace true 和下层包中 false：仅添加上层包
        "sounds": [
            "sound_2"
        ]
    },
    "sound_3": {
        // 上层包中 replace false 和下层包中 true：先添加下层包，然后添加上层包
        // 仍会丢弃位于 RP2 下方的第三个资源包中的值
        "sounds": [
            "sound_7",
            "sound_3"
        ]
    },
    "sound_4": {
        // 上层包中 replace true 和下层包中 true：仅添加上层包
        "sounds": [
            "sound_8"
        ]
    }
}
```

## 播放声音 (Playing Sounds)

Minecraft 提供了各种播放声音的方法，有时不清楚应该使用哪一种。所有方法都接受一个 `SoundEvent`，它可以是您自己的，也可以是 Vanilla 的（Vanilla 声音事件在 `SoundEvents` 类中）。对于以下方法描述，客户端和服务器分别指[逻辑客户端和逻辑服务器 (logical client and logical server)][sides]。

### `Level`

- `playSeededSound(Entity entity, double x, double y, double z, Holder<SoundEvent> soundEvent, SoundSource soundSource, float volume, float pitch, long seed)`
    - 客户端行为：如果传入的玩家是本地玩家，则在给定位置向玩家播放声音事件，否则无操作。
    - 服务器行为：向除传入玩家之外的所有玩家发送数据包，指示客户端在给定位置向玩家播放声音事件。
    - 用法：从将在两端运行的客户端启动代码调用。服务器不向启动玩家播放可以防止向他们播放两次声音事件。或者，从服务器启动的代码（例如[方块实体 (block entity)][be]）调用，传入 `null` 玩家以向所有人播放声音。
- `playSound(Entity entity, double x, double y, double z, SoundEvent soundEvent, SoundSource soundSource, float volume, float pitch)`
    - 转发到 `playSeededSound`，使用随机选择的种子并将持有器包装在 `SoundEvent` 周围
- `playSound(Entity entity, BlockPos pos, SoundEvent soundEvent, SoundSource soundSource, float volume, float pitch)`
    - 转发到上述方法，`x`、`y` 和 `z` 分别取 `pos.getX() + 0.5`、`pos.getY() + 0.5` 和 `pos.getZ() + 0.5` 的值。
- `playLocalSound(double x, double y, double z, SoundEvent soundEvent, SoundSource soundSource, float volume, float pitch, boolean distanceDelay)`
    - 客户端行为：在给定位置向玩家播放声音。不向服务器发送任何内容。如果 `distanceDelay` 为 `true`，则根据到玩家的距离延迟声音。
    - 服务器行为：无操作。
    - 用法：从服务器发送的自定义数据包中调用。Vanilla 将此用于雷声。
- `playPlayerSound(SoundEvent soundEvent, SoundSource soundSource, float volume, float pitch)`
    - 客户端行为：播放绑定到玩家位置的声音。不向服务器发送任何内容。
    - 服务器行为：无操作。
    - 用法：Vanilla 将此用于环境方块声音。

### `ClientLevel`

- `playLocalSound(BlockPos pos, SoundEvent soundEvent, SoundSource soundSource, float volume, float pitch, boolean distanceDelay)`
    - 转发到 `Level#playLocalSound`，`x`、`y` 和 `z` 分别取 `pos.getX() + 0.5`、`pos.getY() + 0.5` 和 `pos.getZ() + 0.5` 的值。

### `Entity`

- `playSound(SoundEvent soundEvent, float volume, float pitch)`
    - 转发到 `Level#playSound`，玩家为 `null`，声音源为 `Entity#getSoundSource`，x/y/z 为实体的位置，其他参数传入。

### `Player`

- `playSound(SoundEvent soundEvent, float volume, float pitch)`（重写 `Entity` 中的方法）
    - 转发到 `Level#playSound`，玩家为 `this`，声音源为 `SoundSource.PLAYER`，x/y/z 为玩家的位置，其他参数传入。因此，客户端/服务器行为模仿 `Level#playSound` 的行为：
        - 客户端行为：在给定位置向客户端玩家播放声音事件。
        - 服务器行为：在给定位置向除调用此方法的玩家之外的所有人播放声音事件。

## 数据生成 (Datagen)

声音文件本身当然不能[数据生成 (datagenned)][datagen]，但 `sounds.json` 文件可以。为此，我们扩展 `SoundDefinitionsProvider` 并重写 `registerSounds()` 方法：

```java
public class MySoundDefinitionsProvider extends SoundDefinitionsProvider {
    // 可以从 `GatherDataEvent.Client` 获取参数。
    public MySoundDefinitionsProvider(PackOutput output) {
        // 使用您实际的模组 ID 而不是 "examplemod"。
        super(output, "examplemod");
    }

    @Override
    public void registerSounds() {
        // 第一个参数接受 Holder<SoundEvent>、SoundEvent 或 ResourceLocation。
        add(MySoundsClass.MY_SOUND, SoundDefinition.definition()
            // 将声音对象添加到声音定义。参数是可变参数。
            .with(
                // 第一个参数接受字符串或 ResourceLocation。
                // 第二个参数可以是 SOUND 或 EVENT，如果是前者可以省略。
                sound("examplemod:sound_1", SoundDefinition.SoundType.SOUND)
                    // 设置音量。也有 double 对应方法。
                    .volume(0.8f)
                    // 设置音调。也有 double 对应方法。
                    .pitch(1.2f)
                    // 设置权重。
                    .weight(2)
                    // 设置衰减距离。
                    .attenuationDistance(8)
                    // 启用流式传输。
                    // 也有无参数重载，默认为 stream(true)。
                    .stream(true)
                    // 启用预加载。
                    // 也有无参数重载，默认为 preload(true)。
                    .preload(true),
                // 我们能得到的最短形式。
                sound("examplemod:sound_2")
            )
            // 设置字幕。
            .subtitle("sound.examplemod.sound_1")
            // 启用替换。
            .replace(true)
        );
    }
}
```

与每个数据提供者一样，不要忘记向事件注册提供者：

```java
@SubscribeEvent // 在模组事件总线上
public static void gatherData(GatherDataEvent.Client event) {
    event.createProvider(MySoundDefinitionsProvider::new);
}
```

[bug]: https://bugs.mojang.com/browse/MC-146721
[datagen]: ../index.md#data-generation
[mcwiki]: https://minecraft.wiki
[mcwikisounds]: https://minecraft.wiki/w/Sounds.json
[modbus]: ../../concepts/events.md#event-buses
[modctor]: ../../gettingstarted/modfiles.md#javafml-and-mod
[registration]: ../../concepts/registries.md
[sides]: ../../concepts/sides.md#the-logical-side
[soundsjson]: #soundsjson