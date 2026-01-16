# 配置(Configuration)

配置(Configurations)定义了可应用于模组(Mod)实例的设置和用户偏好。NeoForge使用一个基于[TOML][toml]文件并通过[NightConfig][nightconfig]读取的配置系统。

## 创建配置(Creating a Configuration)

可以通过创建`IConfigSpec`的子类型来创建配置(Configuration)。NeoForge通过`ModConfigSpec`实现了该类型，并允许通过`ModConfigSpec.Builder`来构建它。构建器(Builder)可以通过`Builder#push`创建配置节(section)以及通过`Builder#pop`离开配置节，从而将配置值分隔到不同的节中。之后，可以使用以下两种方法之一来构建配置(Configuration)：

 方法(Method)     | 描述(Description)
 :---       | :---
`build`     | 创建`ModConfigSpec`。
`configure` | 创建一个包含配置值持有类(Class)和`ModConfigSpec`的对(Pair)。

:::note
`ModConfigSpec.Builder#configure`通常在`static`代码块中使用，并结合一个以`ModConfigSpec.Builder`作为其构造函数参数的类，用于关联并持有配置值：

``` java
//定义一个字段来保存配置和规范(Spec)以供后续使用
public static final ExampleConfig CONFIG;
public static final ModConfigSpec CONFIG_SPEC;

private ExampleConfig(ModConfigSpec.Builder builder) {
    // 定义配置使用的属性(properties)
    // ...
}

//CONFIG 和 CONFIG_SPEC 由同一个构建器(Builder)构建，因此我们使用静态(static)代码块来分隔属性
static {
    Pair<ExampleConfig, ModConfigSpec> pair =
            new ModConfigSpec.Builder().configure(ExampleConfig::new);
        
    //存储结果值
    CONFIG = pair.getLeft();
    CONFIG_SPEC = pair.getRight();
}
```
:::

每个配置值都可以提供额外的上下文(Context)以提供附加行为。上下文必须在配置值完全构建之前定义：

| 方法(Method)         | 描述(Description)                                                                                                 |
|:---------------|:------------------------------------------------------------------------------------------------------------|
| `comment`      | 提供配置值作用的描述。可以提供多个字符串以形成多行注释。 |
| `translation`  | 提供配置值名称的翻译键(translation key)。                                                |
| `worldRestart` | 必须在世界(World)重启后，此配置值才能被更改。                                         |
| `gameRestart`  | 必须在游戏(Game)重启后，此配置值才能被更改。                                          |

### 配置值(ConfigValue)

配置值可以在定义任何上下文后（如果定义了的话），使用任意`#define`方法构建。

所有配置值方法至少接收两个组件：

- 表示变量名的路径(Path)：一个用`.`分隔的字符串，表示配置值所在的节(sections)
- 当不存在有效配置时的默认值

特定的`ConfigValue`方法接收两个额外的组件：

- 一个验证器(validator)，用于确保反序列化的对象是有效的
- 一个表示配置值数据类型的类(Class)

``` java
//将配置属性存储为公共(public)最终(final)字段
public final ModConfigSpec.ConfigValue<String> welcomeMessage;

private ExampleConfig(ModConfigSpec.Builder builder) {
    //定义每个属性
    //一个属性可以是在游戏初始化时记录到控制台的消息
    welcomeMessage = builder.define("welcome_message", "Hello from the config!");
}
```

配置值本身可以通过`ConfigValue#get`获取。这些值会被缓存，以防止多次从文件读取。

#### 其他配置值类型(Additional Config Value Types)

- **范围值(Range Values)**
    - 描述：值必须在定义的边界内
    - 类类型：`Comparable<T>`
    - 方法名：`#defineInRange`
    - 额外组件：
        - 配置值允许的最小值和最大值
        - 表示配置值数据类型的类

:::note
`DoubleValue`、`IntValue`和`LongValue`是范围值，它们分别指定类为`Double`、`Integer`和`Long`。
:::

- **白名单值(Whitelisted Values)**
    - 描述：值必须在提供的集合中
    - 类类型：`T`
    - 方法名：`#defineInList`
    - 额外组件：
        - 一个允许的配置值集合

- **列表值(List Values)**
    - 描述：值是条目的列表
    - 类类型：`List<T>`
    - 方法名：`#defineList`，如果列表允许为空，则使用`#defineListAllowEmpty`
    - 额外组件：
        - 一个提供器(supplier)，用于在配置屏幕中添加新条目时返回要使用的默认值。
        - 一个验证器，用于确保列表中反序列化的元素是有效的
        - （可选）一个验证器，用于确保列表不会包含太少或太多的条目

- **枚举值(Enum Values)**
    - 描述：一个在提供的集合中的枚举值
    - 类类型：`Enum<T>`
    - 方法名：`#defineEnum`
    - 额外组件：
        - 一个获取器(getter)，用于将字符串或整数转换为枚举
        - 一个允许的配置值集合

- **布尔值(Boolean Values)**
    - 描述：一个`boolean`值
    - 类类型：`Boolean`
    - 方法名：`#define`

## 注册配置(Registering a Configuration)

一旦`ModConfigSpec`构建完成，必须注册它，以允许NeoForge根据需要加载、跟踪和同步配置设置。配置应该在模组构造函数中通过`ModContainer#registerConfig`注册。配置可以以[给定的类型][configtype]注册，该类型代表配置所属的端(side)，此外还需要提供`ModConfigSpec`，并可选择性地指定配置的文件名。

``` java
// 在主模组文件中，假设有一个 ModConfigSpec CONFIG
public ExampleMod(ModContainer container) {
    ...
    //注册配置
    container.registerConfig(ModConfig.Type.COMMON, ExampleConfig.CONFIG);
    ...
}
```

### 配置类型(Configuration Types)

配置类型决定了配置文件的位置、加载时间以及文件是否通过网络同步。默认情况下，所有配置都从物理客户端(physical client)上的`.minecraft/config`或物理服务器(physical server)上的`<server_folder>/config`加载。每个配置类型之间的一些细微差别可以在以下小节中找到。

:::tip
NeoForge在其代码库中记录了[配置类型][type]。
:::

- `STARTUP`
    - 从配置文件夹(config folder)加载到物理客户端和物理服务器上
    - 注册时立即读取
    - **不会**通过网络同步
    - 默认后缀为`-startup`

:::warning
注册为`STARTUP`类型的配置可能导致客户端和服务器之间的不同步(desyncs)，例如，如果配置用于禁用内容(content)的注册。因此，强烈建议不要在`STARTUP`中使用任何用于启用或禁用可能改变模组内容的特性的配置。
:::

- `CLIENT`
    - **仅**从配置文件夹加载到物理客户端上
        - 此配置类型没有服务器位置
    - 在`FMLCommonSetupEvent`触发之前立即读取
    - **不会**通过网络同步
    - 默认后缀为`-client`
- `COMMON`
    - 从配置文件夹加载到物理客户端和物理服务器上
    - 在`FMLCommonSetupEvent`触发之前立即读取
    - **不会**通过网络同步
    - 默认后缀为`-common`
- `SERVER`
    - 从配置文件夹加载到物理客户端和物理服务器上
        - 可以通过在以下位置添加配置来为每个世界(world)覆盖：
            - 客户端：`.minecraft/saves/<world_name>/serverconfig`
            - 服务器：`<server_folder>/world/serverconfig`
    - 在`ServerAboutToStartEvent`触发之前立即读取
    - 通过网络同步到客户端
    - 默认后缀为`-server`

## 配置事件(Configuration Events)

每当配置加载、重新加载或卸载时发生的操作，可以使用`ModConfigEvent.Loading`、`ModConfigEvent.Reloading`和`ModConfigEvent.Unloading`事件来完成。这些事件必须[注册][events]到模组事件总线(Mod Event Bus)上。

:::caution
这些事件是针对模组的所有配置调用的；应该使用提供的`ModConfig`对象来区分正在加载或重新加载的是哪个配置。
:::

## 配置屏幕(Configuration Screen)

配置屏幕允许用户在游戏内编辑模组的配置值，而无需打开任何文件。该屏幕将自动解析你注册的配置文件并填充屏幕。

模组可以使用NeoForge提供的内置配置屏幕。模组可以扩展`ConfigurationScreen`来更改默认屏幕的行为，或者创建自己的配置屏幕。模组也可以从头开始创建自己的屏幕，并通过下面的扩展点将该自定义屏幕提供给NeoForge。

可以通过在模组构造期间在[客户端][client]注册一个`IConfigScreenFactory`扩展点来为模组注册配置屏幕：

``` java
// 在主客户端模组文件中
public ExampleModClient(ModContainer container) {
    ...
    // 这将使用 NeoForge 的 ConfigurationScreen 来显示此模组的配置
    container.registerExtensionPoint(IConfigScreenFactory.class, ConfigurationScreen::new);
    ...
}
```

在游戏中，可以通过进入“模组(Mods)”页面，从侧边栏选择模组，然后点击“配置(Config)”按钮来访问配置屏幕。启动时(Startup)、通用(Common)和客户端(Client)配置选项随时可以编辑。服务器(Server)配置仅在本地玩游戏世界(world)时才可在屏幕中编辑。如果连接到服务器或其他人的局域网(LAN)世界，服务器配置选项将在屏幕中被禁用。模组配置屏幕的第一页将显示每个已注册的配置文件，供玩家选择要编辑哪个。

:::warning
如果你要创建屏幕，应该为所有配置条目添加翻译键(Translation keys)并在语言(lang) JSON中定义文本。

你可以使用`ModConfigSpec$Builder#translation`方法为配置指定翻译键，因此我们可以扩展之前的代码为：
``` java
ConfigValue<T> value = builder.comment("此值名为'config_value_name'，如果没有现有配置，则设置为defaultValue")
    .translation("modid.config.config_value_name")
    .define("config_value_name", defaultValue);
```

为了方便翻译，请打开配置屏幕并访问所有配置及其子节。然后退出到模组列表屏幕。此时，所有遇到过的未翻译配置条目将被打印到控制台。这使得更容易知道要翻译什么以及翻译键是什么。
:::

[toml]: https://toml.io/
[nightconfig]: https://github.com/TheElectronWill/night-config
[configtype]: #configuration-types
[type]: https://github.com/neoforged/FancyModLoader/blob/aafe4660ae6eff2702ec786dba8e83c69c0d9e91/loader/src/main/java/net/neoforged/fml/config/ModConfig.java#L88-L121
[events]: ../concepts/events.md#registering-an-event-handler
[client]: ../concepts/sides.md#mod