# 按键映射(Key Mappings)

按键映射(Key Mapping)或键绑定(Key Binding)定义了应绑定到输入（鼠标点击、按键按下等）的特定操作。每当客户端能够接受输入时，都可以检查由按键映射(Key Mapping)定义的每个操作。此外，每个按键映射(Key Mapping)都可以通过[控制选项菜单(Controls option menu)][controls]分配给任何输入。

## 注册 `KeyMapping`

可以通过在仅物理客户端([physical client])上监听[模组事件总线(mod event bus)][eventbus]上的 `RegisterKeyMappingsEvent` 并调用 `register` 来注册 `KeyMapping`。

```java
// 在某个仅物理客户端的类中

// KeyMapping 是延迟初始化的，因此在注册之前不存在
public static final Lazy<KeyMapping> EXAMPLE_MAPPING = Lazy.of(() -> /*...*/);

@SubscribeEvent // 仅在物理客户端上，在模组事件总线(mod event bus)上
public static void registerBindings(RegisterKeyMappingsEvent event) {
    event.register(EXAMPLE_MAPPING.get());
}
```

## 创建 `KeyMapping`

可以使用其构造函数创建 `KeyMapping`。`KeyMapping` 接收一个定义映射名称的[翻译键(translation key)][tk]、映射的默认输入以及一个 `KeyMapping.Category`，用于定义映射在[控制选项菜单(Controls option menu)][controls]中将被归入的类别。

:::提示
可以通过使用 `ResourceLocation` 创建一个新的 `KeyMapping.Category`，并在仅[物理客户端(physical client)][sides]的[模组事件总线(mod event bus)][eventbus]上通过 `RegisterKeyMappingsEvent.registerCategory` 注册它，将 `KeyMapping` 添加到自定义类别。该类别的关联[翻译键(translation key)][tk]是 `key.category.<命名空间>.<路径>`。

```java
public static final KeyMapping.Category EXAMPLE_CATEGORY = new KeyMapping.Category(ResourceLocation.fromNamespaceAndPath("examplemod", "category"));

@SubscribeEvent // 仅在物理客户端上，在模组事件总线(mod event bus)上
public static void registerBindings(RegisterKeyMappingsEvent event) {
    // 注册类别
    event.registerCategory(EXAMPLE_CATEGORY);

    // 注册使用该类别的绑定
    event.register(EXAMPLE_MAPPING.get());
}
```

:::

### 默认输入(Default Inputs)

每个按键映射都有一个与之关联的默认输入。这是通过 `InputConstants.Key` 提供的。每个输入由一个 `InputConstants.Type`（定义提供输入的设备）和一个整数（定义设备上输入的关联标识符）组成。

原版提供三种输入类型：`KEYSYM`，通过提供的 `GLFW` 键标记定义键盘；`SCANCODE`，通过平台特定的扫描码定义键盘；以及 `MOUSE`，定义鼠标。

:::注意
强烈建议对键盘使用 `KEYSYM` 而不是 `SCANCODE`，因为 `GLFW` 键标记不绑定到任何特定系统。您可以在 [GLFW 文档][keyinput] 上阅读更多信息。
:::

整数取决于提供的类型。所有输入代码都在 `GLFW` 中定义：`KEYSYM` 标记以 `GLFW_KEY_*` 为前缀，而 `MOUSE` 代码以 `GLFW_MOUSE_*` 为前缀。

```java
new KeyMapping(
    "key.examplemod.example1", // 将使用此翻译键(translation key)进行本地化
    InputConstants.Type.KEYSYM, // 默认映射在键盘上
    GLFW.GLFW_KEY_P, // 默认键是 P
    KeyMapping.Category.MISC // 映射将在杂项(misc)类别中
)
```

:::注意
如果按键映射不应映射到默认值，则输入应设置为 `InputConstants.UNKNOWN`。原版构造函数将要求您通过 `InputConstants$Key.getValue` 提取输入代码，而 NeoForge 构造函数可以提供原始输入字段。
:::

### `IKeyConflictContext`

并非所有映射都在每个上下文中使用。有些映射仅在图形用户界面(GUI)中使用，而其他映射则纯粹在游戏中使用。为了避免不同上下文中相同按键的映射相互冲突，可以分配一个 `IKeyConflictContext`。

每个冲突上下文包含两个方法：`isActive`，定义映射是否可以在当前游戏状态下使用；以及 `conflicts`，定义映射是否与相同或不同冲突上下文中的按键冲突。

目前，NeoForge 通过 `KeyConflictContext` 定义了三个基本上下文：`UNIVERSAL`，这是默认值，意味着该按键可以在所有上下文中使用；`GUI`，意味着映射只能在打开 `Screen` 时使用；以及 `IN_GAME`，意味着映射只能在未打开 `Screen` 时使用。可以通过实现 `IKeyConflictContext` 来创建新的冲突上下文。

```java
new KeyMapping(
    "key.examplemod.example2",
    KeyConflictContext.GUI, // 映射只能在打开屏幕(Screen)时使用
    InputConstants.Type.MOUSE, // 默认映射在鼠标上
    GLFW.GLFW_MOUSE_BUTTON_LEFT, // 默认鼠标输入是左键
    EXAMPLE_CATEGORY // 映射将在新的示例类别中
)
```

### `KeyModifier`

模组开发者可能不希望映射在同时按住修饰键时具有相同的行为（例如 `G` 与 `CTRL + G`）。为了解决这个问题，NeoForge 向构造函数添加了一个额外参数，接受一个 `KeyModifier`，该修饰符可以将控制 (`KeyModifier.CONTROL`)、Shift (`KeyModifier.SHIFT`) 或 Alt (`KeyModifier.ALT`) 应用于任何输入。`KeyModifier.NONE` 是默认值，不应用任何修饰符。

可以通过按住修饰键和相关输入，在[控制选项菜单(Controls option menu)][controls]中添加修饰符。

```java
new KeyMapping(
    "key.examplemod.example3",
    KeyConflictContext.UNIVERSAL,
    KeyModifier.SHIFT, // 默认映射需要按住 Shift
    InputConstants.Type.KEYSYM, // 默认映射在键盘上
    GLFW.GLFW_KEY_G, // 默认键是 G
    KeyMapping.Category.MISC
)
```

## 检查 `KeyMapping`

可以检查 `KeyMapping` 以查看是否已被点击。根据时间，可以在条件语句中使用该映射来应用相关逻辑。

### 在游戏内(Within the Game)

在游戏内，应通过监听[事件总线(event bus)][eventbus]上的 `ClientTickEvent.Post` 并在 while 循环中检查 `KeyMapping.consumeClick` 来检查映射。`consumeClick` 仅在输入已执行且尚未被先前处理的次数内返回 `true`，因此不会无限期地阻塞游戏。

```java
@SubscribeEvent // 仅在物理客户端上，在游戏事件总线(game event bus)上
public static void onClientTick(ClientTickEvent.Post event) {
    while (EXAMPLE_MAPPING.get().consumeClick()) {
        // 在此处执行点击时要执行的逻辑
    }
}
```

:::警告
不要使用 `InputEvent` 作为 `ClientTickEvent.Post` 的替代品。键盘和鼠标输入有单独的事件，因此它们不会处理任何额外的输入。
:::

### 在图形用户界面(GUI)内部(Inside a GUI)

在图形用户界面(GUI)内部，可以在 `GuiEventListener` 方法之一中使用 `IKeyMappingExtension.isActiveAndMatches` 检查映射。最常见的可检查方法是 `keyPressed` 和 `mouseClicked`。

`keyPressed` 接收 `GLFW` 键标记、平台特定的扫描码以及按下的修饰符的位字段。可以通过使用 `InputConstants.getKey` 创建输入，将按键与映射进行检查。修饰符已在映射方法本身内部检查。

```java
// 在某个 Screen 子类中
@Override
public boolean keyPressed(int key, int scancode, int mods) {
    if (EXAMPLE_MAPPING.get().isActiveAndMatches(InputConstants.getKey(key, scancode))) {
        // 在此处执行按键按下时要执行的逻辑
        return true;
    }
    return super.keyPressed(x, y, button);
}
```

:::注意
如果您不拥有要检查**按键**的屏幕(Screen)，可以改为在[游戏事件总线(game event bus)][eventbus]上监听 `ScreenEvent.KeyPressed` 的 `Pre` 或 `Post` 事件。
:::

`mouseClicked` 接收鼠标的 x 位置、y 位置和点击的按钮。可以通过使用 `InputConstants.Type.getOrCreate` 和 `MOUSE` 输入创建输入，将鼠标按钮与映射进行检查。

```java
// 在某个 Screen 子类中
@Override
public boolean mouseClicked(double x, double y, int button) {
    if (EXAMPLE_MAPPING.get().isActiveAndMatches(InputConstants.TYPE.MOUSE.getOrCreate(button))) {
        // 在此处执行鼠标点击时要执行的逻辑
        return true;
    }
    return super.mouseClicked(x, y, button);
}
```

:::注意
如果您不拥有要检查**鼠标**的屏幕(Screen)，可以改为在[游戏事件总线(game event bus)][eventbus]上监听 `ScreenEvent.MouseButtonPressed` 的 `Pre` 或 `Post` 事件。
:::

[eventbus]: ../concepts/events.md#registering-an-event-handler
[controls]: https://minecraft.wiki/w/Options#Controls
[tk]: ../resources/client/i18n.md#components
[keyinput]: https://www.glfw.org/docs/3.3/input_guide.html#input_key
[sides]: ../concepts/sides.md#the-physical-side