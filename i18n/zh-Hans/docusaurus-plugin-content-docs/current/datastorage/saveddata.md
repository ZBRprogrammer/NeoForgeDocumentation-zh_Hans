---
sidebar_position: 5
---
# 保存的数据 (Saved Data)

保存的数据 (Saved Data, SD) 系统可用于在维度(levels)上保存额外的数据。

_如果数据特定于某些方块实体(block entities)、区块(chunks)或实体(entities)，请考虑使用[数据附件](attachments)替代。_

## `SavedData`

每个 SD 实现都必须是 `SavedData` 类的子类型。这可以像任何其他对象一样实现，有自己的字段和方法，但如果你想将数据存储到磁盘或更改，则必须调用 `setDirty`。`setDirty` 通知游戏有需要写入的更改。如果不调用，数据将仅在当前维度（或对于主世界(Overworld)维度的情况，是世界(world)）加载时持续存在。

``` java
// 对于某个保存数据的实现
public class ExampleSavedData extends SavedData {

    public void foo() {
        // 更改保存数据中的数据
        // 如果数据更改，调用 setDirty
        this.setDirty();
    }
}
```

## `SavedDataType`

由于 `SavedData` 只是一个对象，需要某种关联的标识符。此外，我们还需要将数据读写到磁盘。这就是 `SavedDataType` 的用武之地。它接收保存数据的标识符、当没有数据存在时的默认构造函数，以及用于编码和解码数据的[编解码器(codec)]。该标识符被视为相关世界文件夹和维度级别内的路径位置，如下所示：`./<world_folder>/<level_name>/data/<identifier>.dat`。任何缺失的目录都会被创建，包括作为标识符一部分的目录。

:::note
`SavedDataType` 构造函数的第四个参数是 `DataFixTypes`，但由于 NeoForge 不支持数据修复器(data fixers)，所有原版用例都已打补丁以允许空值。
:::

`SavedDataType` 构造函数有两种变体。第一种接收构造函数的简单 `Supplier` 和用于磁盘处理的常规 `Codec`。但是，如果你想存储当前的 `ServerLevel` 或世界种子，还有一个重载版本，为两者接收一个 `Function`，提供一个 `SavedData.Context`。

``` java
// 对于某个保存数据的实现
public class NoContextExampleSavedData extends SavedData {

    public static final SavedDataType<NoContextExampleSavedData> ID = new SavedDataType<>(
        // 保存数据的标识符
        // 用作维度 `data` 文件夹内的路径
        "example",
        // 初始构造函数
        NoContextExampleSavedData::new,
        // 用于序列化数据的编解码器
        RecordCodecBuilder.create(instance -> instance.group(
            Codec.INT.fieldOf("val1").forGetter(sd -> sd.val1),
            BuiltInRegistries.BLOCK.byNameCodec().fieldOf("val2").forGetter(sd -> sd.val2)
        ).apply(instance, NoContextExampleSavedData::new))
    );

    // 初始构造函数
    public NoContextExampleSavedData() {
        // ...
    }

    // 数据构造函数
    public NoContextExampleSavedData(int val1, Block val2) {
        // ...
    }

    public void foo() {
        // 更改保存数据中的数据
        // 如果数据更改，调用 setDirty
        this.setDirty();
    }
}

// 对于某个保存数据的实现
public class ContextExampleSavedData extends SavedData {

    public static final SavedDataType<ContextExampleSavedData> ID = new SavedDataType<>(
        // 保存数据的标识符
        // 用作维度 `data` 文件夹内的路径
        "example",
        // 初始构造函数
        ContextExampleSavedData::new,
        // 用于序列化数据的编解码器
        ctx -> RecordCodecBuilder.create(instance -> instance.group(
            RecordCodecBuilder.point(ctx),
            Codec.INT.fieldOf("val1").forGetter(sd -> sd.val1),
            BuiltInRegistries.BLOCK.byNameCodec().fieldOf("val2").forGetter(sd -> sd.val2)
        ).apply(instance, ContextExampleSavedData::new))
    );

    // 初始构造函数
    public ContextExampleSavedData(SavedData.Context ctx) {
        // ...
    }

    // 数据构造函数
    public ContextExampleSavedData(SavedData.Context ctx, int val1, Block val2) {
        // ...
    }

    public void foo() {
        // 更改保存数据中的数据
        // 如果数据更改，调用 setDirty
        this.setDirty();
    }
}
```

## 附加到维度

任何 `SavedData` 都是动态加载和/或附加到维度的。因此，如果一个 SD 从未在某个维度上创建，那么它将不存在。

`SavedData` 从 `DimensionDataStorage` 创建和加载，可以通过调用 `ServerChunkCache#getDataStorage` 或 `ServerLevel#getDataStorage` 来访问。然后，你可以通过调用 `DimensionDataStorage#computeIfAbsent` 并传入 `SavedDataType` 来获取或创建 SD 的实例。这将尝试获取 SD 的当前实例（如果存在），或者创建一个新实例并加载所有可用数据。

``` java
// 在某个可以访问 DimensionDataStorage 的方法中
netherDataStorage.computeIfAbsent(ContextExampleSavedData.ID);
```

如果一个 SD 不特定于某个维度，SD 应该附加到主世界(Overworld)，这可以从 `MinecraftServer#overworld` 获取。主世界是唯一永远不会完全卸载的维度，因此非常适合存储多维度数据。

[codec]: codecs.md