---
sidebar_position: 2
---

# 数据组件(Data Components)

数据组件是 `ItemStack` 中用于存储数据的键值对映射。每一条数据，例如烟花爆炸或工具，都作为实际对象存储在堆叠上，使得这些值可见且可操作，而无需动态转换通用编码实例（例如 `CompoundTag`、`JsonElement`）。

## `DataComponentType`

每个数据组件都有一个关联的 `DataComponentType<T>`，其中 `T` 是组件值类型。`DataComponentType` 表示一个引用存储组件值的键，以及一些编解码器，用于在需要时处理读写磁盘和网络。

现有组件的列表可以在 `DataComponents` 中找到。

### 创建自定义数据组件

与 `DataComponentType` 关联的组件值必须实现 `hashCode` 和 `equals`，并且在存储时应被视为**不可变**的。

:::note
组件值可以很容易地使用记录(record)实现。记录字段是不可变的，并实现 `hashCode` 和 `equals`。
:::

``` java
// 一个记录示例
public record ExampleRecord(int value1, boolean value2) {}

// 一个类示例
public class ExampleClass {

    private final int value1;
    // 可以是可变的，但在使用时需要注意
    private boolean value2;

    public ExampleClass(int value1, boolean value2) {
        this.value1 = value1;
        this.value2 = value2;
    }

    @Override
    public int hashCode() {
        return Objects.hash(this.value1, this.value2);
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == this) {
            return true;
        } else {
            return obj instanceof ExampleClass ex
                && this.value1 == ex.value1
                && this.value2 == ex.value2;
        }
    }
}
```

可以通过 `DataComponentType#builder` 创建标准的 `DataComponentType`，并使用 `DataComponentType.Builder#build` 构建。构建器包含三个设置：`persistent`、`networkSynchronized`、`cacheEncoding`。

`persistent` 指定用于从磁盘读取和写入组件值的 [`Codec`][codec]。`networkSynchronized` 指定用于跨网络读取和写入组件的 `StreamCodec`。如果未指定 `networkSynchronized`，则 `persistent` 中提供的 `Codec` 将被包装并用作 [`StreamCodec`][streamcodec]。

:::warning
构建器中必须提供 `persistent` 或 `networkSynchronized` 中的一个；否则，将抛出 `NullPointerException`。如果没有数据需要通过网络发送，则将 `networkSynchronized` 设置为 `StreamCodec#unit`，提供默认组件值。
:::

`cacheEncoding` 缓存 `Codec` 的编码结果，使得如果组件值未更改，任何后续编码都使用缓存值。这应仅在组件值预期很少或从不更改时使用。

`DataComponentType` 是注册对象，必须[注册(registered)][registered]。

``` java
// 使用 ExampleRecord(int, boolean)
// 下面只应使用一个 Codec 和/或 StreamCodec
// 提供了多个作为示例

// 基本 codec
public static final Codec<ExampleRecord> BASIC_CODEC = RecordCodecBuilder.create(instance ->
    instance.group(
        Codec.INT.fieldOf("value1").forGetter(ExampleRecord::value1),
        Codec.BOOL.fieldOf("value2").forGetter(ExampleRecord::value2)
    ).apply(instance, ExampleRecord::new)
);
public static final StreamCodec<ByteBuf, ExampleRecord> BASIC_STREAM_CODEC = StreamCodec.composite(
    ByteBufCodecs.INT, ExampleRecord::value1,
    ByteBufCodecs.BOOL, ExampleRecord::value2,
    ExampleRecord::new
);

// 如果不通过网络发送任何内容，使用单位流编解码器
public static final StreamCodec<ByteBuf, ExampleRecord> UNIT_STREAM_CODEC = StreamCodec.unit(new ExampleRecord(0, false));


// 在另一个类中
// 专门的 DeferredRegister.DataComponents 简化了数据组件注册，并避免了 `Supplier` 中 `DataComponentType.Builder` 的一些泛型推断问题
public static final DeferredRegister.DataComponents REGISTRAR = DeferredRegister.createDataComponents(Registries.DATA_COMPONENT_TYPE, "examplemod");

public static final Supplier<DataComponentType<ExampleRecord>> BASIC_EXAMPLE = REGISTRAR.registerComponentType(
    "basic",
    builder -> builder
        // 用于从磁盘读/写数据的编解码器
        .persistent(BASIC_CODEC)
        // 用于跨网络读/写数据的编解码器
        .networkSynchronized(BASIC_STREAM_CODEC)
);

/// 组件不会保存到磁盘
public static final Supplier<DataComponentType<ExampleRecord>> TRANSIENT_EXAMPLE = REGISTRAR.registerComponentType(
    "transient",
    builder -> builder.networkSynchronized(BASIC_STREAM_CODEC)
);

// 没有数据会通过网络同步
public static final Supplier<DataComponentType<ExampleRecord>> NO_NETWORK_EXAMPLE = REGISTRAR.registerComponentType(
   "no_network",
   builder -> builder
        .persistent(BASIC_CODEC)
        // 注意这里我们使用单位流编解码器
        .networkSynchronized(UNIT_STREAM_CODEC)
);
```

## 组件映射(Component Map)

所有数据组件都存储在 `DataComponentMap` 中，使用 `DataComponentType` 作为键，对象作为值。`DataComponentMap` 的功能类似于只读的 `Map`。因此，有一些方法可以根据其 `DataComponentType` `#get` 条目，或者如果不存在则提供默认值（通过 `#getOrDefault`）。

``` java
// 对于某个 DataComponentMap map

// 如果组件存在，将获取染料颜色
// 否则为 null
@Nullable
DyeColor color = map.get(DataComponents.BASE_COLOR);
```

### `PatchedDataComponentMap`

由于默认的 `DataComponentMap` 只提供基于读操作的方法，写操作通过子类 `PatchedDataComponentMap` 支持。这包括 `#set` 组件的值或 `#remove` 完全移除它。

`PatchedDataComponentMap` 使用原型和补丁映射存储更改。原型是一个 `DataComponentMap`，包含此映射应具有的默认组件及其值。补丁映射是从 `DataComponentType` 到包含对默认组件所做更改的 `Optional` 值的映射。

``` java
// 对于某个 PatchedDataComponentMap map

// 将基础颜色设置为白色
map.set(DataComponents.BASE_COLOR, DyeColor.WHITE);

// 通过以下方式移除基础颜色
// - 如果没有默认值，则移除补丁
// - 如果有默认值，则设置一个空的 Optional
map.remove(DataComponents.BASE_COLOR);
```

:::danger
原型和补丁映射都是 `PatchedDataComponentMap` 哈希码的一部分。因此，映射中的任何组件值都应被视为**不可变**的。在修改数据组件的值后，始终调用 `#set` 或下面讨论的其引用方法之一。
:::

## 组件持有者(Component Holder)

所有可以持有数据组件的实例都实现 `DataComponentHolder`。`DataComponentHolder` 实际上是对 `DataComponentMap` 中只读方法的委托。

``` java
// 对于某个 ItemStack stack

// 委托给 'DataComponentMap#get'
@Nullable
DyeColor color = stack.get(DataComponents.BASE_COLOR);
```

### `MutableDataComponentHolder`

`MutableDataComponentHolder` 是 NeoForge 提供的一个接口，用于支持组件映射的基于写操作的方法。Vanilla 和 NeoForge 中的所有实现都使用 `PatchedDataComponentMap` 存储数据组件，因此 `#set` 和 `#remove` 方法也有同名的委托。

此外，`MutableDataComponentHolder` 还提供了一个 `#update` 方法，该方法处理获取组件值或提供的默认值（如果未设置），对值进行操作，然后将其设置回映射。操作符要么是一个 `UnaryOperator`，它接受组件值并返回组件值，要么是一个 `BiFunction`，它接受组件值和另一个对象并返回组件值。

``` java
// 对于某个 ItemStack stack

FireworkExplosion explosion = stack.get(DataComponents.FIREWORK_EXPLOSION);

// 修改组件值
explosion = explosion.withFadeColors(new IntArrayList(new int[] {1, 2, 3}));

// 由于我们修改了组件值，之后应调用 'set'
stack.set(DataComponents.FIREWORK_EXPLOSION, explosion);

// 更新组件值（内部调用 'set'）
stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    // 如果没有组件值存在，则使用默认值
    FireworkExplosion.DEFAULT,
    // 返回一个新的 FireworkExplosion 来设置
    explosion -> explosion.withFadeColors(new IntArrayList(new int[] {4, 5, 6}))
);

stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    // 如果没有组件值存在，则使用默认值
    FireworkExplosion.DEFAULT,
    // 提供给函数的对象
    new IntArrayList(new int[] {7, 8, 9}),
    // 返回一个新的 FireworkExplosion 来设置
    FireworkExplosion::withFadeColors
);
```

## 向物品添加默认数据组件

虽然数据组件存储在 `ItemStack` 上，但可以在 `Item` 上设置默认组件的映射，以便在构造时作为原型传递给 `ItemStack`。可以通过 `Item.Properties#component` 向 `Item` 添加组件。

``` java
// 对于某个 DeferredRegister.Items REGISTRAR
public static final Item COMPONENT_EXAMPLE = REGISTRAR.register("component",
    // 使用 register 而不是其他重载，因为 DataComponentType 尚未注册
    registryName -> new Item(
        new Item.Properties()
        .setId(ResourceKey.create(Registries.ITEM, registryName))
        .component(BASIC_EXAMPLE.get(), new ExampleRecord(24, true))
    )
);
```

如果数据组件应添加到属于 Vanilla 或其他模组的现有物品上，则应在 [**模组事件总线**][modbus] 上侦听 `ModifyDefaultComponentEvent`。该事件提供了 `modify` 和 `modifyMatching` 方法，允许修改相关物品的 `DataComponentPatch.Builder`。构建器可以 `#set` 组件或 `#remove` 现有组件。

``` java
@SubscribeEvent // 在模组事件总线上
public static void modifyComponents(ModifyDefaultComponentsEvent event) {
    // 在西瓜种子上设置组件
    event.modify(Items.MELON_SEEDS, builder ->
        builder.set(BASIC_EXAMPLE.get(), new ExampleRecord(10, false))
    );

    // 为任何有合成剩余物的物品移除组件
    event.modifyMatching(
        item -> !item.getCraftingRemainder().isEmpty(),
        builder -> builder.remove(DataComponents.BUCKET_ENTITY_DATA)
    );
}
```

## 使用自定义组件持有者

要创建自定义数据组件持有者，持有者对象只需实现 `MutableDataComponentHolder` 并实现缺失的方法。持有者对象必须包含一个表示 `PatchedDataComponentMap` 的字段来实现相关方法。

``` java
public class ExampleHolder implements MutableDataComponentHolder {

    private int data;
    private final PatchedDataComponentMap components;

    // 可以提供重载来提供映射本身
    public ExampleHolder() {
        this.data = 0;
        this.components = new PatchedDataComponentMap(DataComponentMap.EMPTY);
    }

    @Override
    public DataComponentMap getComponents() {
        return this.components;
    }

    @Nullable
    @Override
    public <T> T set(DataComponentType<? super T> componentType, @Nullable T value) {
        return this.components.set(componentType, value);
    }

    @Nullable
    @Override
    public <T> T remove(DataComponentType<? extends T> componentType) {
        return this.components.remove(componentType);
    }

    @Override
    public void applyComponents(DataComponentPatch patch) {
        this.components.applyPatch(patch);
    }

    @Override
    public void applyComponents(DataComponentMap components) {
        this.components.setAll(components);
    }

    // 其他方法
}
```

### `DataComponentPatch` 和编解码器

为了将组件持久化到磁盘或通过网络发送信息，持有者可以发送整个 `DataComponentMap`。然而，这通常是一种信息浪费，因为任何默认值都已经存在于数据发送到的位置。因此，我们使用 `DataComponentPatch` 来发送相关数据。`DataComponentPatch` 只包含组件映射的补丁信息，没有任何默认值。然后补丁被应用于接收方位置的原型。

可以通过 `#patch` 从 `PatchedDataComponentMap` 创建 `DataComponentPatch`。同样，`PatchedDataComponentMap#fromPatch` 可以在给定原型 `DataComponentMap` 和 `DataComponentPatch` 的情况下构造 `PatchedDataComponentMap`。

``` java
public class ExampleHolder implements MutableDataComponentHolder {

    public static final Codec<ExampleHolder> CODEC = RecordCodecBuilder.create(instance ->
        instance.group(
            Codec.INT.fieldOf("data").forGetter(ExampleHolder::getData),
            DataCopmonentPatch.CODEC.optionalFieldOf("components", DataComponentPatch.EMPTY).forGetter(holder -> holder.components.asPatch())
        ).apply(instance, ExampleHolder::new)
    );

    public static final StreamCodec<RegistryFriendlyByteBuf, ExampleHolder> STREAM_CODEC = StreamCodec.composite(
        ByteBufCodecs.INT, ExampleHolder::getData,
        DataComponentPatch.STREAM_CODEC, holder -> holder.components.asPatch(),
        ExampleHolder::new
    );

    // ...

    public ExampleHolder(int data, DataComponentPatch patch) {
        this.data = data;
        this.components = PatchedDataComponentMap.fromPatch(
            // 要应用的原型映射
            DataComponentMap.EMPTY,
            // 相关的补丁
            patch
        );
    }

    // ...
}
```

[通过网络同步持有者数据][network]和从磁盘读/写数据必须手动完成。

[registered]: ../concepts/registries.md
[codec]: ../datastorage/codecs.md
[modbus]: ../concepts/events.md#event-buses
[network]: ../networking/payload.md
[streamcodec]: ../networking/streamcodecs.md