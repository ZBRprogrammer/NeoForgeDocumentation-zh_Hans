import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 访问转换器(Access Transformers)

访问转换器(Access Transformers，简称ATs)允许拓宽类、方法和字段的可见性并修改其`final`标志。这使得模组开发者能够访问和修改其控制范围之外、原本不可访问的类成员。

相关[规范文档][specs]可以在NeoForged的GitHub上查看。

## 添加访问转换器(Access Transformers)

将访问转换器(Access Transformer)添加到您的模组项目非常简单，只需在`build.gradle`中添加一行配置即可。

访问转换器(Access Transformers)需要在`build.gradle`中声明。AT文件可以指定在任何位置，只要它们在编译时被复制到`resources`输出目录即可。

<Tabs defaultValue="mdg">
<TabItem value="mdg" label="ModDevGradle">

这里默认不需要做任何事情！

</TabItem>
<TabItem value="ng" label="NeoGradle">

```
// In build.gradle:
minecraft {
    accessTransformers {
        file 'src/main/resources/META-INF/accesstransformer.cfg'
    }
}
```

</TabItem>
</Tabs>

默认情况下，NeoForge会搜索`META-INF/accesstransformer.cfg`。如果`build.gradle`在其他位置指定了访问转换器(Access Transformers)，则需要在`neoforge.mods.toml`中定义它们的位置：

```
# In neoforge.mods.toml:
[[accessTransformers]]
## The file is relative to the output directory of the resources, or the root path inside the jar when compiled
## The 'resources' directory represents the root output directory of the resources
file="META-INF/accesstransformer.cfg"
```

此外，可以指定多个AT文件，它们将按顺序应用。这对于包含多个包的大型模组非常有用。

<Tabs defaultValue="mdg">
<TabItem value="mdg" label="ModDevGradle">

```
// In build.gradle:
neoForge {
    // ModDevGradle already tries to include 'src/main/resources/META-INF/accesstransformer.cfg' by default
    accessTransformers.from 'src/additions/resources/accesstransformer_additions.cfg'
}
```

</TabItem>
<TabItem value="ng" label="NeoGradle">

```
// In build.gradle:
minecraft {
    accessTransformers {
        file 'src/main/resources/META-INF/accesstransformer.cfg'
        file 'src/additions/resources/accesstransformer_additions.cfg'
    }
}
```

</TabItem>
</Tabs>

```
# In neoforge.mods.toml
[[accessTransformers]]
file="accesstransformer.cfg"

[[accessTransformers]]
file="accesstransformer_additions.cfg"
```

添加或修改任何访问转换器(Access Transformer)后，必须刷新Gradle项目才能使转换生效。

## 访问转换器(Access Transformer)规范

### 注释

`#`之后直到行尾的所有文本都将被视为注释，不会被解析。

### 访问修饰符(Access Modifiers)

访问修饰符(Access Modifiers)指定给定目标将被转换为何种新的成员可见性。按可见性递减顺序排列：

- `public` - 对其包内外的所有类可见
- `protected` - 仅对包内类和子类可见
- `default` - 仅对包内类可见
- `private` - 仅对类内部可见

可以在上述修饰符后附加特殊修饰符`+f`和`-f`，分别用于添加或移除`final`修饰符，该修饰符在应用时会阻止子类化、方法重写或字段修改。

:::danger
指令仅修改其直接引用的方法；任何重写方法都不会被访问转换器(Access Transformers)转换。建议确保被转换的方法没有限制可见性的、未转换的重写，否则JVM将抛出错误。

可以安全转换的方法示例包括`final`方法（或`final`类中的方法）和`static`方法。`private`方法通常也是安全的；但是，它们可能会导致任何子类型中的意外重写，因此应执行一些额外的手动验证。
:::

### 目标与指令(Targets and Directives)

#### 类

要针对类：

```
<access modifier> <fully qualified class name>
```

内部类通过将外部类的完全限定名称和内部类的名称用`$`作为分隔符连接来表示。

#### 字段

要针对字段：

```
<access modifier> <fully qualified class name> <field name>
```

#### 方法

针对方法需要一种特殊的语法来表示方法参数和返回类型：

```
<access modifier> <fully qualified class name> <method name>(<parameter types>)<return type>
```

##### 指定类型

也称为“描述符”：更多技术细节请参阅 [Java虚拟机规范, SE 21, 第4.3.2和4.3.3节][jvmdescriptors]。

- `B` - `byte`，有符号字节
- `C` - `char`，UTF-16中的Unicode码点
- `D` - `double`，双精度浮点值
- `F` - `float`，单精度浮点值
- `I` - `integer`，32位整数
- `J` - `long`，64位整数
- `S` - `short`，有符号短整数
- `Z` - `boolean`，`true`或`false`值
- `[` - 引用数组的一个维度
    - 示例：`[[S` 指代 `short[][]`
- `L<class name>;` - 引用一个引用类型
    - 示例：`Ljava/lang/String;` 指代 `java.lang.String` 引用类型 _(注意使用斜杠而不是点)_
- `(` - 引用方法描述符，参数应在此提供，如果没有参数则不提供
    - 示例：`<method>(I)Z` 指代一个需要一个整数参数并返回布尔值的方法
- `V` - 表示方法不返回值，只能在方法描述符的末尾使用
    - 示例：`<method>()V` 指代一个没有参数且不返回任何内容的方法

### 示例

```
# Makes public the ByteArrayToKeyFunction interface in Crypt
public net.minecraft.util.Crypt$ByteArrayToKeyFunction

# Makes protected and removes the final modifier from 'random' in MinecraftServer
protected-f net.minecraft.server.MinecraftServer random

# Makes public the 'makeExecutor' method in Util,
# accepting a String and returns a TracingExecutor
public net.minecraft.Util makeExecutor(Ljava/lang/String;)Lnet/minecraft/TracingExecutor;

# Makes public the 'leastMostToIntArray' method in UUIDUtil,
# accepting two longs and returning an int[]
public net.minecraft.core.UUIDUtil leastMostToIntArray(JJ)[I
```

[specs]: https://github.com/NeoForged/AccessTransformers/blob/main/FMLAT.md
[jvmdescriptors]: https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.3.2