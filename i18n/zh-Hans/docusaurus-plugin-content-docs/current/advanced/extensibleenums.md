# 可扩展枚举(Extensible Enums)

可扩展枚举(Extensible Enums)是对特定原版(vanilla)枚举的增强，允许添加新的条目。这是通过运行时修改已编译的枚举字节码来添加元素实现的。

## `IExtensibleEnum`

所有可以添加新条目的枚举都实现了`IExtensibleEnum`接口。此接口作为标记，允许`RuntimeEnumExtender`启动插件服务知道哪些枚举应该被转换。

:::warning
您**不应**在自己的枚举上实现此接口。根据您的用例，请使用映射(Map)或注册表(Registry)替代。
由于转换器运行的顺序问题，未打补丁以实现该接口的枚举无法通过混入(Mixin)或核心模组(Coremods)添加该接口。
:::

### 创建枚举条目

要创建新的枚举条目，需要创建一个JSON文件，并在`neoforge.mods.toml`中通过`[[mods]]`块的`enumExtensions`条目引用。指定的路径必须相对于`resources`目录：

``` java
# In neoforge.mods.toml:
[[mods]]
## The file is relative to the output directory of the resources, or the root path inside the jar when compiled
## The 'resources' directory represents the root output directory of the resources
enumExtensions="META-INF/enumextensions.json"
```

条目的定义包括目标枚举的类名、新字段的名称（必须以模组ID为前缀）、用于构造条目的构造函数的描述符以及传递给该构造函数的参数。

``` java
{
    "entries": [
        {
            // The enum class the entry should be added to
            "enum": "net/minecraft/world/item/ItemDisplayContext",
            // The field name of the new entry, must be prefixed with the mod ID
            "name": "EXAMPLEMOD_STANDING",
            // The constructor to be used
            "constructor": "(ILjava/lang/String;Ljava/lang/String;)V",
            // Constant parameters provided directly.
            "parameters": [ -1, "examplemod:standing", null ]
        },
        {
            "enum": "net/minecraft/world/item/Rarity",
            "name": "EXAMPLEMOD_CUSTOM",
            "constructor": "(ILjava/lang/String;Ljava/util/function/UnaryOperator;)V",
            // The parameters to be used, provided as a reference to an EnumProxy<Rarity> field in the given class
            "parameters": {
                "class": "example/examplemod/MyEnumParams",
                "field": "CUSTOM_RARITY_ENUM_PROXY"
            }
        },
        {
            "enum": "net/minecraft/world/damagesource/DamageEffects",
            "name": "EXAMPLEMOD_TEST",
            "constructor": "(Ljava/lang/String;Ljava/util/function/Supplier;)V",
            // The parameters to be used, provided as a reference to a method in the given class
            "parameters": {
                "class": "example/examplemod/MyEnumParams",
                "method": "getTestDamageEffectsParameter"
            }
        }
    ]
}
```

``` java
public class MyEnumParams {
    public static final EnumProxy<Rarity> CUSTOM_RARITY_ENUM_PROXY = new EnumProxy<>(
            Rarity.class, -1, "examplemod:custom", (UnaryOperator<Style>) style -> style.withItalic(true)
    );
    
    public static Object getTestDamageEffectsParameter(int idx, Class<?> type) {
        return type.cast(switch (idx) {
            case 0 -> "examplemod:test";
            case 1 -> (Supplier<SoundEvent>) () -> SoundEvents.DONKEY_ANGRY;
            default -> throw new IllegalArgumentException("Unexpected parameter index: " + idx);
        });
    }
}
```

#### 构造函数(Constructor)

构造函数必须指定为[方法描述符][jvmdescriptors]，并且只能包含源代码中可见的参数，忽略隐藏的常量名称和序数参数。
如果构造函数标记了`@ReservedConstructor`注解，则不能用于模组化的枚举常量。

#### 参数(Parameters)

参数可以通过三种方式指定，具体取决于参数类型：

- 在JSON文件中作为常量数组内联指定（仅允许基本类型值、字符串和为任何引用类型传递null）
- 作为对模组类中`EnumProxy<TheEnum>`类型字段的引用（参见上面的`EnumProxy`示例）
    - 第一个参数指定目标枚举，后续参数是传递给枚举构造函数的参数
- 作为对返回`Object`的方法的引用，其中返回值是要使用的参数值。该方法必须恰好有两个参数，类型为`int`（参数的索引）和`Class<?>`（参数的预期类型）
    - `Class<?>`对象应用于转换（`Class#cast()`）返回值，以便在模组代码中保留`ClassCastException`。

:::warning
用作参数值来源的字段和/或方法应放在单独的类中，以避免过早意外加载模组类。
:::

某些参数有额外的规则：

- 如果参数是与枚举上的`@IndexedEnum`注解相关的int ID参数，则它被忽略并替换为条目的序数。如果在JSON中内联指定了该参数，则必须指定为`-1`，否则将抛出异常。
- 如果参数是与枚举上的`@NamedEnum`注解相关的字符串名称参数，则必须按照`ResourceLocation`的`namespace:path`格式以模组ID为前缀，否则将抛出异常。

#### 获取生成的常量(Retrieving the Generated Constant)

可以通过`TheEnum.valueOf(String)`获取生成的枚举常量。如果使用字段引用提供参数，则也可以通过`EnumProxy#getValue()`从`EnumProxy`对象获取常量。

## 为NeoForge做贡献

要将新的可扩展枚举(Extensible Enum)添加到NeoForge，至少需要做两件事：

- 使枚举实现`IExtensibleEnum`，以标记该枚举应通过`RuntimeEnumExtender`进行转换。
- 添加一个`getExtensionInfo`方法，返回`ExtensionInfo.nonExtended(TheEnum.class)`。

根据枚举的具体细节，可能需要进一步的操作：

- 如果枚举具有int ID参数，且该参数应与条目的序数匹配，则如果它不是第一个参数，则应用`@IndexedEnum`注解，并将ID的参数索引作为注解的值
- 如果枚举具有用于序列化的字符串名称参数，因此应该被命名空间化，则如果它不是第一个参数，则应用`@NamedEnum`注解，并将名称的参数索引作为注解的值
- 如果枚举通过网络发送，则应使用`@NetworkedEnum`注解，注解参数指定值可能被发送的方向（客户端绑定、服务器绑定或双向）
- 如果枚举具有模组无法使用的构造函数（例如，因为它们需要在模组注册运行之前初始化的枚举上需要注册表对象），则应使用`@ReservedConstructor`注解

:::note
如果枚举实际添加了任何条目，`getExtensionInfo`方法将在运行时被转换以提供动态生成的`ExtensionInfo`。
:::

``` java
// This is an example, not an actual enum within Vanilla

// The first argument must match the enum constant's ordinal
@net.neoforged.fml.common.asm.enumextension.IndexedEnum
// The second argument is a string that must be prefixed with the mod id
@net.neoforged.fml.common.asm.enumextension.NamedEnum(1)
// This enum is used in networking and must be checked for mismatches between the client and server
@net.neoforged.fml.common.asm.enumextension.NetworkedEnum(net.neoforged.fml.common.asm.enumextension.NetworkedEnum.NetworkCheck.BIDIRECTIONAL)
public enum ExampleEnum implements net.neoforged.fml.common.asm.enumextension.IExtensibleEnum {
    // VALUE_1 represents the name parameter here
    VALUE_1(0, "value_1", false),
    VALUE_2(1, "value_2", true),
    VALUE_3(2, "value_3");

    ExampleEnum(int arg1, String arg2, boolean arg3) {
        // ...
    }

    ExampleEnum(int arg1, String arg2) {
        this(arg1, arg2, false);
    }

    public static net.neoforged.fml.common.asm.enumextension.ExtensionInfo getExtensionInfo() {
        return net.neoforged.fml.common.asm.enumextension.ExtensionInfo.nonExtended(ExampleEnum.class);
    }
}
```

[jvmdescriptors]: https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.3.2