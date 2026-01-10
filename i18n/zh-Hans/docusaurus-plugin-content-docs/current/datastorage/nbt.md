---
sidebar_position: 1
---
# 命名二进制标签 (Named Binary Tag, NBT)

NBT 是 Minecraft 早期引入的一种格式，由 Notch 本人编写。它在整个 Minecraft 代码库中广泛用于数据存储。

## 规范 (Specification)

NBT 规范类似于 JSON 规范，但有一些区别：
- 存在字节(byte)、短整型(short)、长整型(long)和浮点数(float)的独特类型，分别后缀为 `b`、`s`、`l` 和 `f`，类似于它们在 Java 代码中的表示方式。
    - 双精度浮点数(double)也可以后缀 `d`，但这不是必需的，类似于 Java 代码。Java 中可选的整数后缀 `i` 是不允许的。
    - 后缀不区分大小写。例如，`64b` 与 `64B` 相同，`0.5F` 与 `0.5f` 相同。
- 布尔值(boolean)不存在，它们由字节表示。`true` 变成 `1b`，`false` 变成 `0b`。
    - 当前实现将所有非零值视为 `true`，所以 `2b` 也会被视为 `true`。
- NBT 中没有 `null` 的等价物。
- 键周围的引号是可选的。所以 JSON 属性 `"duration": 20` 在 NBT 中可以变成 `duration: 20` 和 `"duration": 20`。
- 在 JSON 中称为子对象(sub-object)的东西，在 NBT 中称为**复合标签(compound tag)**（或简称复合物(compound)）。
- NBT 列表不能混合匹配类型，这与 JSON 不同。列表类型由第一个元素决定，或在代码中定义。
    - 但是，列表的列表可以混合匹配不同的列表类型。所以一个包含两个列表的列表是允许的，其中第一个是字符串列表，第二个是字节列表。
- 有特殊的**数组(array)**类型，它们与列表不同，但遵循用方括号包含元素的方案。有三种数组类型：
    - 字节数组(Byte arrays)，在数组开头用 `B;` 表示。例如：`[B;0b,30b]`
    - 整数数组(Integer arrays)，在数组开头用 `I;` 表示。例如：`[I;0,-300]`
    - 长整型数组(Long arrays)，在数组开头用 `L;` 表示。例如：`[L;0l,240l]`
- 列表、数组和复合标签中的尾随逗号是允许的。

## NBT 文件

Minecraft 广泛使用 `.nbt` 文件，例如用于[数据包(datapack)][datapack]中的结构文件。包含区域（即一组区块）内容的区域文件（`.mca`），以及游戏在不同地方使用的各种 `.dat` 文件，也都是 NBT 文件。

NBT 文件通常使用 GZip 压缩。因此，它们是二进制文件，不能直接编辑。

## 代码中的 NBT

与 JSON 类似，所有 NBT 对象都是某个封闭对象的子对象。所以让我们创建一个：

``` java
CompoundTag tag = new CompoundTag();
```

我们现在可以将数据放入该标签中：

``` java
tag.putInt("Color", 0xffffff);
tag.putString("Level", "minecraft:overworld");
tag.putDouble("IAmRunningOutOfIdeasForNamesHere", 1d);
```

这里存在几个辅助方法，例如，`putIntArray` 除了接受 `int[]` 的标准变体外，还有一个方便的方法接受 `List<Integer>`。

当然，我们也可以从该标签中获取值：

``` java
Optional<Integer> color = tag.getInt("Color");
Optional<String> level = tag.getString("Level");
Optional<Double> d = tag.getDouble("IAmRunningOutOfIdeasForNamesHere");
```

由于未知标签是否存在，返回的值是用 `Optional` 包装的。可以使用其中一个 `*Or*` 方法为基本类型指定默认值。`ListTag` 可以通过 `getListOrEmpty` 默认，`CompoundTag` 可以通过 `getCompoundOrEmpty` 默认。基本数组类型没有 `*Or*` 等价物。

``` java
int color = tag.getIntOr("Color", 0xffffff);
String level = tag.getStringOr("Level", "minecraft:overworld");
double d = tag.getDoubleOr("IAmRunningOutOfIdeasForNamesHere", 1d);
```

所有标签类型都实现 `Tag` 接口。除了 `CompoundTag` 之外，大多数标签类型（例如 `ByteTag` 或 `StringTag`）大多是内部使用的，尽管如果你偶然碰到一些，直接的 `CompoundTag#get` 和 `#put` 方法可以与它们一起工作。

不过，有一个明显的例外：`ListTag`。使用这些很特殊，因为它们与某些在内部计算的标签类型相关联：

``` java
ListTag newList = new ListTag();
// 将标签添加到列表中
newList.add(StringTag.valueOf("Value1"));
newList.add(StringTag.valueOf("Value2"));

// 获取标签
ListTag getList = tag.getListOrEmpty("SomeListHere");
```

最后，在其他 `CompoundTag` 内部使用 `CompoundTag` 直接利用 `CompoundTag#get` 和 `#put`：

``` java
tag.put("Tag", new CompoundTag());

// 如果你想要处理 null 情况，也可以使用常规的 `get`
tag.getCompoundOrEmpty("Tag");
```

## NBT 的用途

NBT 在 Minecraft 的许多地方被使用。[`BlockEntity`s][blockentity] 和 [`Entity`s][entity] 将 NBT 用法抽象为[值访问(value accesses)][valueio]。`ItemStack` 将用法抽象为[数据组件(data components)][datacomponents]。

## 另请参阅

- [Minecraft Wiki 上的 NBT 格式][nbtwiki]

[blockentity]: ../blockentities/index.md
[datapack]: ../resources/index.md#data
[datacomponents]: ../items/datacomponents.md
[entity]: ../entities/index.md
[nbtwiki]: https://minecraft.wiki/w/NBT_format
[valueio]: valueio.md