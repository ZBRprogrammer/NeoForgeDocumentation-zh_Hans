---
sidebar_position: 2
---
# 编解码器 (Codecs)

编解码器是来自 Mojang [DataFixerUpper] 的序列化工具，用于描述对象如何在不同的格式之间转换，例如 JSON 的 `JsonElement` 和 NBT 的 `Tag`。

## 使用编解码器

编解码器主要用于将 Java 对象编码（或序列化）为某种数据格式类型，并将格式化数据对象解码（或反序列化）回其关联的 Java 类型。这通常通过 `Codec#encodeStart` 和 `Codec#parse` 分别完成。

### DynamicOps

为了确定编码和解码的中间文件格式，`#encodeStart` 和 `#parse` 都需要一个 `DynamicOps` 实例来定义该格式内的数据。

[DataFixerUpper] 库包含 `JsonOps`，用于编解码存储在 [`Gson`][gson] 的 `JsonElement` 实例中的 JSON 数据。`JsonOps` 支持两种 `JsonElement` 序列化版本：`JsonOps#INSTANCE` 定义标准 JSON 文件，`JsonOps#COMPRESSED` 允许数据被压缩成单个字符串。

``` java
// 让 exampleCodec 表示一个 Codec<ExampleJavaObject>
// 让 exampleObject 是一个 ExampleJavaObject
// 让 exampleJson 是一个 JsonElement

// 将 Java 对象编码为常规 JsonElement
exampleCodec.encodeStart(JsonOps.INSTANCE, exampleObject);

// 将 Java 对象编码为压缩的 JsonElement
exampleCodec.encodeStart(JsonOps.COMPRESSED, exampleObject);

// 将 JsonElement 解码为 Java 对象
// 假设 JsonElement 被正常解析
exampleCodec.parse(JsonOps.INSTANCE, exampleJson);
```

Minecraft 还提供了 `NbtOps` 来编解码存储在 `Tag` 实例中的 NBT 数据。这可以通过 `NbtOps#INSTANCE` 引用。

``` java
// 让 exampleCodec 表示一个 Codec<ExampleJavaObject>
// 让 exampleObject 是一个 ExampleJavaObject
// 让 exampleNbt 是一个 Tag

// 将 Java 对象编码为 Tag
exampleCodec.encodeStart(NbtOps.INSTANCE, exampleObject);

// 将 Tag 解码为 Java 对象
exampleCodec.parse(NbtOps.INSTANCE, exampleNbt);
```

为了处理注册表条目，Minecraft 提供了 `RegistryOps`，它包含一个查找提供器(lookup provider)来获取可用的注册表元素。这些可以通过 `RegistryOps#create` 创建，它接收要存储数据的具体类型的 `DynamicOps` 和包含可用注册表访问权限的查找提供器。NeoForge 扩展了 `RegistryOps` 以创建 `ConditionalOps`：一个可以处理[条目加载条件][conditions]的注册表编解码器查找器。

``` java
// 让 lookupProvider 是一个 HolderLookup.Provider
// 让 exampleCodec 表示一个 Codec<ExampleJavaObject>
// 让 exampleObject 是一个 ExampleJavaObject
// 让 exampleJson 是一个 JsonElement

// 获取 JsonElement 的注册表操作(RegistryOps)
RegistryOps<JsonElement> ops = RegistryOps.create(JsonOps.INSTANCE, lookupProvider);

// 将 Java 对象编码为 JsonElement
exampleCodec.encodeStart(ops, exampleObject);

// 将 JsonElement 解码为 Java 对象
exampleCodec.parse(ops, exampleJson);
```

#### 格式转换

`DynamicOps` 也可以单独用于在两种不同的编码格式之间进行转换。这可以使用 `#convertTo` 并提供 `DynamicOps` 格式和要转换的编码对象来完成。

``` java
// 将 Tag 转换为 JsonElement
// 让 exampleTag 是一个 Tag
JsonElement convertedJson = NbtOps.INSTANCE.convertTo(JsonOps.INSTANCE, exampleTag);
```

### DataResult

使用编解码器编码或解码的数据会返回一个 `DataResult`，它保存转换后的实例或一些错误数据，具体取决于转换是否成功。转换成功时，`#result` 提供的 `Optional` 将包含成功转换的对象。如果转换失败，`#error` 提供的 `Optional` 将包含 `PartialResult`，它根据编解码器保存错误消息和部分转换的对象。

此外，`DataResult` 上还有许多方法可用于将结果或错误转换为所需的格式。例如，`#resultOrPartial` 将返回一个 `Optional`，成功时包含结果，失败时包含部分转换的对象。该方法接收一个字符串消费者(string consumer)来确定如何报告错误消息（如果存在）。

``` java
// 让 exampleCodec 表示一个 Codec<ExampleJavaObject>
// 让 exampleJson 是一个 JsonElement

// 将 JsonElement 解码为 Java 对象
DataResult<ExampleJavaObject> result = exampleCodec.parse(JsonOps.INSTANCE, exampleJson);

result
    // 获取结果或错误时的部分结果，报告错误消息
    .resultOrPartial(errorMessage -> /* 对错误消息进行某些操作 */)
    // 如果结果或部分结果存在，进行某些操作
    .ifPresent(decodedObject -> /* 对解码后的对象进行某些操作 */);
```

## 现有的编解码器

### 基本类型 (Primitives)

`Codec` 类包含某些已定义基本类型的编解码器静态实例。

| 编解码器        | Java 类型     |
| :------------ | :------------ |
| `BOOL`        | `Boolean`     |
| `BYTE`        | `Byte`        |
| `SHORT`       | `Short`       |
| `INT`         | `Integer`     |
| `LONG`        | `Long`        |
| `FLOAT`       | `Float`       |
| `DOUBLE`      | `Double`      |
| `STRING`      | `String`*     |
| `BYTE_BUFFER` | `ByteBuffer`  |
| `INT_STREAM`  | `IntStream`   |
| `LONG_STREAM` | `LongStream`  |
| `PASSTHROUGH` | `Dynamic<?>`**|
| `EMPTY`       | `Unit`***     |

\* `String` 可以通过 `Codec#string` 或 `Codec#sizeLimitedString` 限制为特定数量的字符。

\*\* `Dynamic` 是一个对象，它以支持的 `DynamicOps` 格式保存编码值。这些通常用于将编码的对象格式转换为其他编码的对象格式。

\*\*\* `Unit` 是一个用于表示 `null` 对象的对象。

### 原版(Vanilla)和NeoForge

Minecraft 和 NeoForge 为经常编码和解码的对象定义了许多编解码器。一些例子包括 `ResourceLocation#CODEC` 用于 `ResourceLocation`，`ExtraCodecs#INSTANT_ISO8601` 用于 `DateTimeFormatter#ISO_INSTANT` 格式的 `Instant`，以及 `CompoundTag#CODEC` 用于 `CompoundTag`。

:::caution
`CompoundTag` 无法使用 `JsonOps` 从 JSON 解码数字列表。`JsonOps` 在转换时会将数字设置为其最窄的类型。`ListTag` 强制其数据为特定类型，因此具有不同类型的数字（例如 `64` 是 `byte`，`384` 是 `short`）会在转换时抛出错误。
:::

原版和 NeoForge 注册表也有针对注册表包含的对象类型的编解码器（例如 `BuiltInRegistries#BLOCK` 有一个 `Codec<Block>`）。`Registry#byNameCodec` 会将注册表对象编码为其注册名称。原版注册表还有一个 `Registry#holderByNameCodec`，它编码为注册名称并解码为包装在 `Holder` 中的注册表对象。

## 创建编解码器

可以为任何对象创建编解码器以进行编码和解码。为了便于理解，将显示等效的编码 JSON。

### 记录 (Records)

编解码器可以通过记录(records)来定义对象。每个记录编解码器使用显式命名字段定义任何对象。创建记录编解码器的方法有很多种，但最简单的是通过 `RecordCodecBuilder#create`。

`RecordCodecBuilder#create` 接收一个函数，该函数定义一个 `Instance` 并返回对象的应用（`App`）。可以将其类比为创建类*实例*和用于将该类*应用*于构造对象的构造函数。

``` java
// 要为其创建编解码器的某个对象
public class SomeObject {

    public SomeObject(String s, int i, boolean b) { /* ... */ }

    public String s() { /* ... */ }

    public int i() { /* ... */ }

    public boolean b() { /* ... */ }
}
```

#### 字段 (Fields)

`Instance` 最多可以使用 `#group` 定义 16 个字段。每个字段必须是一个应用(application)，定义正在为其创建对象的实例以及对象的类型。满足此要求的最简单方法是获取一个 `Codec`，设置要解码的字段名称，并设置用于编码该字段的获取器(getter)。

可以使用 `#fieldOf`（如果字段是必需的）或 `#optionalFieldOf`（如果字段包装在 `Optional` 中或具有默认值）从 `Codec` 创建字段。这两种方法都需要一个字符串，其中包含编码对象中的字段名称。然后，可以使用 `#forGetter` 设置用于编码字段的获取器，该获取器接收一个函数，该函数在给定对象时返回字段数据。

:::warning
如果存在解析时抛出错误的元素，`#optionalFieldOf` 将抛出错误。如果错误应该被吞掉，请改用 `#lenientOptionalFieldOf`。
:::

然后，可以通过 `#apply` 应用结果产品，以定义实例应如何为应用程序构造对象。为了方便起见，分组的字段应按它们在构造函数中出现的相同顺序列出，以便该函数可以简单地是构造函数方法引用。

``` java
public static final Codec<SomeObject> RECORD_CODEC = RecordCodecBuilder.create(instance -> // 给定一个实例
    instance.group( // 定义实例内的字段
        Codec.STRING.fieldOf("s").forGetter(SomeObject::s), // 字符串
        Codec.INT.optionalFieldOf("i", 0).forGetter(SomeObject::i), // 整数，如果字段不存在则默认为0
        Codec.BOOL.fieldOf("b").forGetter(SomeObject::b) // 布尔值
    ).apply(instance, SomeObject::new) // 定义如何创建对象
);
```

```json5
// 编码的 SomeObject
{
    "s": "value",
    "i": 5,
    "b": false
}

// 另一个编码的 SomeObject
{
    "s": "value2",
    // i 被省略，默认为 0
    "b": true
}

// 另一个编码的 SomeObject
{
    "s": "value2",
    // 将抛出错误，因为未使用 lenientOptionalFieldOf
    "i": "bad_value",
    "b": true
}
```

### 转换器 (Transformers)

编解码器可以通过映射方法转换为等效的或部分等效的表示形式。每个映射方法接收两个函数：一个将当前类型转换为新类型，另一个将新类型转换回当前类型。这是通过 `#xmap` 函数完成的。

``` java
// 一个类
public class ClassA {

    public ClassB toB() { /* ... */ }
}

// 另一个等效的类
public class ClassB {

    public ClassA toA() { /* ... */ }
}

// 假设存在某个编解码器 A_CODEC
public static final Codec<ClassB> B_CODEC = A_CODEC.xmap(ClassA::toB, ClassB::toA);
```

如果某个类型是部分等效的，意味着在转换过程中存在一些限制，那么有一些映射函数会返回 `DataResult`，该结果可用于在遇到异常或无效状态时返回错误状态。

| A 是否完全等价于 B | B 是否完全等价于 A | 转换方法         |
| :----------------: | :----------------: | :--------------- |
| 是                 | 是                 | `#xmap`          |
| 是                 | 否                 | `#flatComapMap`  |
| 否                 | 是                 | `#comapFlatMap`  |
| 否                 | 否                 | `#flatXMap`      |

``` java
// 给定一个字符串编解码器要转换为整数
// 并非所有字符串都能变成整数（A 不完全等价于 B）
// 所有整数都能变成字符串（B 完全等价于 A）
public static final Codec<Integer> INT_CODEC = Codec.STRING.comapFlatMap(
    s -> { // 失败时返回包含错误的 DataResult
        try {
            return DataResult.success(Integer.valueOf(s));
        } catch (NumberFormatException e) {
            return DataResult.error(s + " is not an integer.");
        }
    },
    Integer::toString // 常规函数
);
```

```json5
// 将返回 5
"5"

// 将错误，不是整数
"value"
```

#### 范围编解码器 (Range Codecs)

范围编解码器是 `#flatXMap` 的一个实现，如果值不在设定的最小值和最大值之间（包含边界），则返回错误的 `DataResult`。如果超出范围，该值仍会作为部分结果提供。通过 `#intRange`、`#floatRange` 和 `#doubleRange` 分别为整数、浮点数和双精度数提供了实现。

``` java
public static final Codec<Integer> RANGE_CODEC = Codec.intRange(0, 4);
```

```json5
// 将有效，在 [0, 4] 内
4

// 将错误，在 [0, 4] 外
5
```

#### 字符串解析器 (String Resolver)

`Codec#stringResolver` 是 `flatXmap` 的一个实现，它将字符串映射到某种对象。

``` java
public record StringResolverObject(String name) { /* ... */ }

// 假设存在某个 Map<String, StringResolverObject> OBJECT_MAP
public static final Codec<StringResolverObject> STRING_RESOLVER_CODEC = Codec.stringResolver(StringResolverObject::name, OBJECT_MAP::get);
```

```json5
// 将此字符串映射到其关联的对象
"example_name"
```

### 默认值 (Defaults)

如果编码或解码的结果失败，可以通过 `Codec#orElse` 或 `Codec#orElseGet` 提供默认值。

``` java
public static final Codec<Integer> DEFAULT_CODEC = Codec.INT.orElse(
    errorMessage -> /* 对错误消息进行某些操作 */,
    0 // 也可以通过 #orElseGet 提供供应值
);
```

```json5
// 不是整数，默认为 0
"value"
```

### 单位 (Unit)

一个在代码中提供值但编码为空值的编解码器可以使用 `Codec#unit` 表示。如果一个编解码器在数据对象中使用不可编码的条目，这很有用。

``` java
public static final Codec<IEventBus> UNIT_CODEC = Codec.unit(
    () -> NeoForge.EVENT_BUS // 也可以是原始值
);
```

```json5
// 这里什么都没有，将返回 NeoForge 事件总线
```

### 延迟初始化 (Lazy Initialized)

有时，一个编解码器可能依赖于构造时不存在的数据。在这种情况下，可以使用 `Codec#lazyInitialized` 让编解码器在首次编码/解码时构造自身。该方法接收一个供应的编解码器。

``` java
public static final Codec<IEventBus> LAZY_CODEC = Codec.lazyInitialized(
    () -> Codec.unit(NeoForge.EVENT_BUS)
);
```

```json5
// 这里什么都没有，将返回 NeoForge 事件总线
// 编码/解码方式与普通编解码器相同
```

### 列表 (List)

可以通过 `Codec#listOf` 从对象编解码器生成对象列表的编解码器。`listOf` 还可以接收表示列表最小和最大大小的整数。`sizeLimitedListOf` 功能相同，但只指定最大限制。

``` java
// BlockPos#CODEC 是一个 Codec<BlockPos>
public static final Codec<List<BlockPos>> LIST_CODEC = BlockPos.CODEC.listOf();
```

```json5
// 编码的 List<BlockPos>
[
    [1, 2, 3], // BlockPos(1, 2, 3)
    [4, 5, 6], // BlockPos(4, 5, 6)
    [7, 8, 9]  // BlockPos(7, 8, 9)
]
```

使用列表编解码器解码的列表对象存储在**不可变**列表中。如果需要可变列表，应对列表编解码器应用[转换器]。

### 映射 (Map)

可以通过 `Codec#unboundedMap` 从两个编解码器生成键和值对象的映射的编解码器。无限制映射可以指定任何基于字符串或经过字符串转换的值作为键。

``` java
// BlockPos#CODEC 是一个 Codec<BlockPos>
public static final Codec<Map<String, BlockPos>> MAP_CODEC = Codec.unboundedMap(Codec.STRING, BlockPos.CODEC);
```

```json5
// 编码的 Map<String, BlockPos>
{
    "key1": [1, 2, 3], // key1 -> BlockPos(1, 2, 3)
    "key2": [4, 5, 6], // key2 -> BlockPos(4, 5, 6)
    "key3": [7, 8, 9]  // key3 -> BlockPos(7, 8, 9)
}
```

使用无限制映射编解码器解码的映射对象存储在**不可变**映射中。如果需要可变映射，应对映射编解码器应用[转换器]。

:::caution
无限制映射仅支持编码/解码为字符串的键。可以使用键值[对]列表编解码器来绕过此限制。
:::

### 对 (Pair)

可以通过 `Codec#pair` 从两个编解码器生成对象对的编解码器。

对编解码器通过首先解码对中的左对象，然后获取编码对象的剩余部分并从该部分解码右对象来解码对象。因此，编解码器必须在解码后表达有关编码对象的一些信息（例如[记录]），或者它们必须被增强为 `MapCodec` 并通过 `#codec` 转换为常规编解码器。这通常可以通过使编解码器成为某个对象的[字段]来完成。

``` java
public static final Codec<Pair<Integer, String>> PAIR_CODEC = Codec.pair(
    Codec.INT.fieldOf("left").codec(),
    Codec.STRING.fieldOf("right").codec()
);
```

```json5
// 编码的 Pair<Integer, String>
{
    "left": 5,       // fieldOf 查找 'left' 键以获取左对象
    "right": "value" // fieldOf 查找 'right' 键以获取右对象
}
```

:::tip
具有非字符串键的映射编解码器可以使用键值对列表并应用[转换器]来编码/解码。
:::

### 任一 (Either)

可以通过 `Codec#either` 从两个编解码器生成两种不同编码/解码对象数据方法的编解码器。

任一(either)编解码器尝试使用第一个编解码器解码对象。如果失败，则尝试使用第二个编解码器解码。如果再次失败，则 `DataResult` 将仅包含来自第二次编解码器失败的错误。

``` java
public static final Codec<Either<Integer, String>> EITHER_CODEC = Codec.either(
    Codec.INT,
    Codec.STRING
);
```

```json5
// 编码的 Either.Left<Integer, String>
5

// 编码的 Either.Right<Integer, String>
"value"
```

:::tip
这可以与[转换器]结合使用，以从两种不同的编码方法中获取特定对象。
:::

#### 异或 (Xor)

`Codec#xor` 是[任一]编解码器的一个特例，只有当两种方法之一成功处理时，结果才算成功。如果两个编解码器都可以处理，则会抛出错误。

``` java
public static final Codec<Either<Integer, String>> XOR_CODEC = Codec.xor(
    Codec.INT.fieldOf("number").codec(),
    Codec.STRING.fieldOf("text").codec()
);
```

```json5
// 编码的 Either.Left<Integer, String>
{
    "number": 4
}

// 编码的 Either.Right<Integer, String>
{
    "text": "value"
}

// 抛出错误，因为两者都可以解码
{
    "number": 4,
    "text": "value"
}
```

#### 替代 (Alternative)

`Codec#withAlternative` 是[任一]编解码器的一个特例，其中两个编解码器都尝试解码同一对象，但以不同格式存储。第一个（或主要）编解码器将尝试解码对象。如果失败，则将使用第二个编解码器。编码将始终使用主要编解码器。

``` java
public static final Codec<BlockPos> ALTERNATIVE_CODEC = Codec.withAlternative(
    BlockPos.CODEC,
    RecordCodecBuilder.create(instance -> instance.group(
        Codec.INT.fieldOf("x").forGetter(BlockPos::getX),
        Codec.INT.fieldOf("y").forGetter(BlockPos::getY),
        Codec.INT.fieldOf("z").forGetter(BlockPos::getZ)
    ), BlockPos::new)
);
```

```json5
// 解码 BlockPos 的常规方法
[ 1, 2, 3 ]

// 解码 BlockPos 的替代方法
{
    "x": 1,
    "y": 2,
    "z": 3
}
```

### 递归 (Recursive)

有时，一个对象可能将相同类型的对象作为字段引用。例如，`EntityPredicate` 为载具、乘客和目标实体接收一个 `EntityPredicate`。在这种情况下，可以使用 `Codec#recursive` 来提供作为创建编解码器函数一部分的编解码器。

``` java
// 定义我们的递归对象
public record RecursiveObject(Optional<RecursiveObject> inner) { /* ... */ }

public static final Codec<RecursiveObject> RECURSIVE_CODEC = Codec.recursive(
    RecursiveObject.class.getSimpleName(), // 这是用于 toString 方法的
    recursedCodec -> RecordCodecBuilder.create(instance -> instance.group(
        recursedCodec.optionalFieldOf("inner").forGetter(RecursiveObject::inner)
    ).apply(instance, RecursiveObject::new))
);
```

```json5
// 一个编码的递归对象
{
    "inner": {
        "inner": {}
    }
}
```

### 分发 (Dispatch)

编解码器可以有子编解码器，这些子编解码器可以根据指定的类型解码特定对象，这是通过 `Codec#dispatch` 实现的。这通常用于包含编解码器的注册表中，例如规则测试或方块放置器。

分发(dispatch)编解码器首先尝试从某个字符串键（通常是 `type`）获取编码的类型。然后，解码该类型，调用用于解码实际对象的特定编解码器的获取器。如果用于解码对象的 `DynamicOps` 压缩了其映射，或者对象编解码器本身没有被增强为 `MapCodec`（例如记录或具有字段的基本类型），那么对象需要存储在 `value` 键中。否则，对象在与其余数据相同的层级被解码。

``` java
// 定义我们的对象
public abstract class ExampleObject {

    // 定义用于指定编码对象类型的方法
    public abstract MapCodec<? extends ExampleObject> type();
}

// 创建存储字符串的简单对象
public class StringObject extends ExampleObject {

    public StringObject(String s) { /* ... */ }

    public String s() { /* ... */ }

    public MapCodec<? extends ExampleObject> type() {
        // 一个已注册的注册表对象
        // "string":
        //   Codec.STRING.xmap(StringObject::new, StringObject::s).fieldOf("string")
        return STRING_OBJECT_CODEC.get();
    }
}

// 创建存储字符串和整数的复杂对象
public class ComplexObject extends ExampleObject {

    public ComplexObject(String s, int i) { /* ... */ }

    public String s() { /* ... */ }

    public int i() { /* ... */ }

    public MapCodec<? extends ExampleObject> type() {
        // 一个已注册的注册表对象
        // "complex":
        //   RecordCodecBuilder.mapCodec(instance ->
        //     instance.group(
        //       Codec.STRING.fieldOf("s").forGetter(ComplexObject::s),
        //       Codec.INT.fieldOf("i").forGetter(ComplexObject::i)
        //     ).apply(instance, ComplexObject::new)
        //   )
        return COMPLEX_OBJECT_CODEC.get();
    }
}

// 假设存在 Registry<MapCodec<? extends ExampleObject>> DISPATCH
public static final Codec<ExampleObject> = DISPATCH.byNameCodec() // 获取 Codec<MapCodec<? extends ExampleObject>>
    .dispatch(
        ExampleObject::type, // 从特定对象获取编解码器
        Function.identity() // 从注册表获取编解码器
    );
```

```json5
// 简单对象
{
    "type": "string", // 对应 StringObject
    "value": "value" // 编解码器类型未从 MapCodec 增强，需要字段
}

// 复杂对象
{
    "type": "complex", // 对应 ComplexObject

    // 编解码器类型从 MapCodec 增强，可以内联
    "s": "value",
    "i": 0
}
```

[DataFixerUpper]: https://github.com/Mojang/DataFixerUpper
[gson]: https://github.com/google/gson
[conditions]: ../resources/server/conditions.md
[transformer]: #transformer-codecs
[pair]: #pair
[records]: #records
[field]: #fields
[either]: #either