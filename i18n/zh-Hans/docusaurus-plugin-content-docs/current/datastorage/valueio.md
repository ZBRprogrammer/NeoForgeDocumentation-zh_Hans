---
sidebar_position: 3
---
# 值输入/输出 (Value I/O)

值输入/输出系统是一种标准化的序列化方法，用于操作某些支持对象的数据，例如[NBT 的 `CompoundTag`s][nbt]。

## 输入和输出

值输入/输出系统由两部分组成：在序列化期间写入对象的 `ValueOutput`，以及在反序列化期间从对象读取的 `ValueInput`。实现方法通常将 `ValueOutput` 或 `ValueInput` 作为其唯一参数，不返回任何内容。值输入/输出期望支持对象是字符串键到对象值的字典。使用提供的方法，值输入/输出然后读取或写入信息到支持对象。

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);
    // 将数据写入输出
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);
    // 从输入读取数据
}

// 对于某个 Entity 子类
@Override
protected void addAdditionalSaveData(ValueOutput output) {
    super.addAdditionalSaveData(output);
    // 将数据写入输出
}

@Override
protected void readAdditionalSaveData(ValueInput input) {
    super.readAdditionalSaveData(input);
    // 从输入读取数据
}
```

### 基本类型 (Primitives)

值输入/输出包含用于读取和写入某些基本类型的方法。`ValueOutput` 方法以 `put*` 为前缀，接收键和基本类型值。`ValueInput` 方法命名为 `get*Or`，接收键和一个默认值（如果不存在）。

| Java 类型   | `ValueOutput` | `ValueInput`           |
|:----------:|:-------------:|:----------------------:|
| `boolean`  | `putBoolean`  | `getBooleanOr`         |
| `byte`     | `putByte`     | `getByteOr`            |
| `short`    | `putShort`    | `getShortOr`           |
| `int`      | `putInt`      | `getInt`\*, `getIntOr` |
| `long`     | `putLong`     | `getLong`\*, `getLongOr`|
| `float`    | `putFloat`    | `getFloatOr`           |
| `double`   | `putDouble`   | `getDoubleOr`          |
| `String`   | `putString`   | `getString`\*, `getStringOr`|
| `int[]`    | `putIntArray` | `getIntArray`\*        |

\* 这些 `ValueInput` 方法返回用 `Optional` 包装的基本类型，而不是接收并传回某些回退值。

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);

    // 将数据写入输出
    output.putBoolean(
        // 字符串键
        "boolValue",
        // 与此键关联的值
        true
    );
    output.putString("stringValue", "Hello world!");
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // 从输入读取数据

    // 如果不存在，默认为 false
    boolean boolValue = input.getBooleanOr(
        // 要检索的字符串键
        "boolValue",
        // 如果键不存在则返回的默认值
        false
    );

    // 如果不存在，默认为 'Dummy!'
    String stringValue = input.getStringOr("stringValue", "Dummy!");
    // 返回用 Optional 包装的值
    Optional<String> stringValueOpt = input.getString("stringValue");
}
```

### 编解码器 (Codecs)

[`Codec`s][codec] 也可以用于从值输入/输出存储和读取值。在原版中，所有 `Codec` 都使用 `RegistryOps` 处理，允许存储数据包条目。`ValueOutput#store` 和 `storeNullable` 接收键、用于写入对象的编解码器以及对象本身。如果对象为 `null`，`storeNullable` 将不写入任何内容。`ValueInput#read` 可以通过接收键和编解码器来读取对象，返回用 `Optional` 包装的对象。

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);

    // 将数据写入输出
    output.storeNullable("codecValue", Rarity.CODEC, Rarity.EPIC);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // 从输入读取数据
    Optional<Rarity> codecValue = input.read("codecValue", Rarity.CODEC);
}
```

`ValueOutput` 和 `ValueInput` 也为 `MapCodec` 提供了 `store` / `read` 方法。与 `Codec` 相比，`MapCodec` 变体将值合并到当前根上。

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);

    // 将数据写入输出
    output.store(
        SingleFile.MAP_CODEC,
        new SingleFile(ResourceLocation.fromNamespaceAndPath("examplemod", "example"))
    );
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // 从输入读取数据

    // 不需要键，因为它们存储在根值访问上
    Optional<SingleFile> file = input.read(SingleFile.MAP_CODEC);
    // 这是存在的，因为 `SingleFile` 写入了 `resource` 参数
    String resource = input.getStringOr("resource", "Not present!");
}
```

:::warning
`MapCodec` 会将任何键写入值访问器，可能覆盖现有数据。确保 `MapCodec` 内的任何键与其他键不同。
:::

### 列表 (Lists)

可以通过两种方法之一创建和读取列表：子值输入/输出或[编解码器]。

通过 `ValueOutput#childrenList` 创建一个列表，接收某个键。这将返回一个 `ValueOutput.ValueOutputList`，它充当值的只写列表。可以通过 `ValueOutputList#addChild` 将新的值对象添加到列表中。这将返回一个 `ValueOutput` 来写入值对象数据。然后可以使用 `ValueInput#childrenList` 或 `childrenListOrEmpty`（如果不存在则默认为空列表）读取该列表。这些方法返回一个 `ValueInput.ValueInputList`，它充当只读可迭代对象或流（通过 `stream`）。

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);

    // 将数据写入输出

    // 创建列表
    ValueOutput.ValueOutputList listValue = output.childrenList("listValue");
    // 添加元素
    ValueOutput childIdx0 = listValue.addChild();
    childIdx0.putBoolean("boolChild", false);
    ValueOutput childIdx1 = listValue.addChild();
    childIdx1.putInt("boolChild", true);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // 从输入读取数据

    // 读取列表的值
    for (ValueInput childInput : input.childrenListOrEmpty("listValue")) {
        boolean boolChild = childInput.getBooleanOr("boolChild", false);
    }
}
```

`Codec` 通过 `ValueOutput#list` 为数据对象提供了一个列表变体。这接收一个键和某个 `Codec`，返回一个 `ValueOutput.TypedOutputList`。`TypedOutputList` 与 `ValueOutputList` 相同，只是它操作数据对象而不是使用另一个值输入/输出。可以通过 `TypedOutputList#add` 将元素添加到列表中。然后，类似地，可以使用 `ValueInput#list` 或 `listOrEmpty` 读取该列表，返回一个 `TypedValueInput`。

:::note
`TypedValueOutput` / `TypedValueInput` 和 `Codec#listOf` 之间的主要区别在于错误的处理方式。对于 `Codec#listOf`，失败的条目将导致整个对象被标记为错误的 `DataResult`。同时，类型化的值输入/输出通常通过 `ProblemReporter` 处理错误。在原版中，`Codec#listOf` 提供了更大的灵活性，因为 `ProblemReporter` 是在创建值输入/输出时指定的。但是，自定义的值输入/输出用法可以根据用例实现任何一种。
:::

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);

    // 将数据写入输出

    // 创建列表
    ValueOutput.TypedInputList<Rarity> listValue = output.list("listValue", Rarity.CODEC);
    // 添加元素
    listValue.add(Rarity.COMMON);
    listValue.add(Rarity.EPIC);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // 从输入读取数据

    // 读取列表的值
    for (Rarity rarity : input.listOrEmpty("listValue", Rarity.CODEC)) {
        // ...
    }
}
```

:::warning
即使列表为空，也会被写入 `ValueOutput`。如果你不想写入列表，那么 `TypedOutputList` 或 `ValueOutputList` 应该检查它是否 `isEmpty`，然后使用列表键调用 `discard`。

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);

    // 将数据写入输出

    // 创建列表
    ValueOutput.TypedInputList<Rarity> listValue = output.list("listValue", Rarity.CODEC);

    // 检查列表是否为空
    if (listValue.isEmpty()) {
        // 从输出中丢弃
        output.discard("listValue");
    }
}
```
:::

### 对象 (Objects)

可以通过子对象创建和读取对象。`ValueOutput#child` 创建一个新的 `ValueObject`，给定一个键。然后，可以使用 `ValueInput#child` 读取该对象，或者使用 `childOrEmpty`（如果它应该默认为具有空支持值的 `ValueInput`）。

``` java
// 对于某个 BlockEntity 子类
@Override
protected void saveAdditional(ValueOutput output) {
    super.saveAdditional(output);

    // 将数据写入输出

    // 创建对象
    ValueOutput objectValue = output.child("objectValue");
    // 向对象添加数据
    objectValue.putBoolean("boolChild", true);
    objectValue.putInt("intChild", 20);
}

@Override
protected void loadAdditional(ValueInput input) {
    super.loadAdditional(input);

    // 从输入读取数据

    // 读取对象
    ValueInput objectValue = input.childOrEmpty("objectValue");
    // 从对象获取数据
    boolean boolChild = objectValue.getBooleanOr("boolChild", false);
    int intChild = objectValue.getIntOr("intChild", 0);
}
```

## ValueIOSerializable

`ValueIOSerializable` 是一个 NeoForge 添加的接口，用于可以使用值输入/输出进行序列化和反序列化的对象。NeoForge 使用此 API 来处理[数据附件][attachments]。该接口提供了两个方法：`serialize` 用于将对象写入 `ValueOutput`，`deserialize` 用于从 `ValueInput` 读取对象。

``` java
public class ExampleObject implements ValueIOSerializable {

    @Override
    public void serialize(ValueOutput output) {
        // 在此处写入对象数据
    }

    @Override
    public void deserialize(ValueInput input) {
        // 在此处读取对象数据
    }
}
```

## 实现 (Implementations)

### NBT

[NBTs][nbt] 的值输入/输出通过 `TagValueOutput` 和 `TagValueInput` 处理。

`TagValueOutput` 可以通过 `createWithContext` 或 `createWithoutContext` 创建，`createWithContext` 意味着输出可以访问 `HolderLookup.Provider`，它提供所有注册表条目（静态和数据包），而 `createWithoutContext` 不提供任何数据包访问。原版只使用 `createWithContext`。一旦使用了 `ValueOutput`，可以通过 `TagValueOutput#buildResult` 检索 `CompoundTag`。另一方面，可以通过 `create` 创建 `TagValueInput`，接收 `HolderLookup.Provider` 和输入正在访问的 `CompoundTag`。

这两个值输入/输出还接收一个 `ProblemReporter`。`ProblemReporter` 用于收集读取/写入过程中的所有内部错误。目前，这只跟踪 `Codec` 错误。如何处理这些错误取决于模组开发者。原版实现会在 `ProblemReporter` 不为空时抛出错误。

``` java
// 假设我们可以访问 HolderLookup.Provider lookupProvider

TagValueOutput output = TagValueOutput.createWithContext(
    ProblemReporter.DISCARDING, // 选择丢弃所有错误
    lookupProvider
);

// 写入输出...

CompoundTag tag = output.buildResult();

// 收集错误
ProblemReporter.Collector reporter = new ProblemReporter.Collector(
    // 可选地接收根路径元素
    // 某些对象（例如，方块实体，实体）有一个 #problemPath() 方法可以提供
    new RootFieldPathElement("example_object")
);

TagValueInput input = TagValueInput.create(
    reporter,
    lookupProvider,
    tag
);

// 从输入读取...
```

[attachments]: attachments.md
[codec]: codecs.md
[nbt]: nbt.md