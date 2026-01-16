# 自定义战利品对象

由于战利品表系统的复杂性，有几个[注册表][registries]在起作用，模组开发者都可以使用它们来添加更多行为。

所有与战利品表相关的注册表都遵循类似的模式。要添加新的注册表条目，通常需要扩展某个类或实现某个接口来承载你的功能。然后，为序列化定义一个[编解码器][codec]，并使用 `DeferredRegister` 像往常一样将该编解码器注册到相应的注册表。这与大多数注册表（例如方块/方块状态和物品/物品堆）使用的“一个基础对象，多个实例”方法一致。

## 自定义战利品条目类型

要创建自定义战利品条目类型，请扩展 `LootPoolEntryContainer` 或其两个直接子类之一：`LootPoolSingletonContainer` 或 `CompositeEntryBase`。为了举例说明，我们想要创建一个返回[实体][entity]掉落物的战利品条目类型——这纯粹是为了示例目的，实际上直接引用另一个战利品表会更理想。让我们从创建战利品条目类型类开始：

```java
// 我们扩展 LootPoolSingletonContainer，因为我们有一组“有限”的掉落物。
// 部分代码改编自 NestedLootTable。
public class EntityLootEntry extends LootPoolSingletonContainer {
    // 一个 Holder，用于我们希望为其掷出其他表的实体类型。
    private final Holder<EntityType<?>> entity;

    // 常见的做法是使用私有构造函数和静态工厂方法。
    // 这是因为 weight、quality、conditions 和 functions 由下面的 lambda 表达式提供。
    private EntityLootEntry(Holder<EntityType<?>> entity, int weight, int quality, List<LootItemCondition> conditions, List<LootItemFunction> functions) {
        // 将 lambda 提供的参数传递给父类。
        super(weight, quality, conditions, functions);
        // 设置我们的值。
        this.entity = entity;
    }

    // 静态构建器方法，接受我们的自定义参数，并与提供所有条目类型通用值的 lambda 结合。
    public static LootPoolSingletonContainer.Builder<?> entityLoot(Holder<EntityType<?>> entity) {
        // 使用 LootPoolSingletonContainer 中定义的静态 simpleBuilder() 方法。
        return simpleBuilder((weight, quality, conditions, functions) -> new EntityLootEntry(entity, weight, quality, conditions, functions));
    }

    // 这就是魔法发生的地方。要添加物品堆，我们通常调用 consumer 的 #accept 方法。
    // 但是，在这种情况下，我们让 #getRandomItems 为我们完成。
    @Override
    public void createItemStack(Consumer<ItemStack> consumer, LootContext context) {
        // 获取实体的战利品表。如果不存在，将返回空的战利品表，因此不需要进行 null 检查。
        LootTable table = context.getLevel().reloadableRegistries().getLootTable(entity.value().getDefaultLootTable());
        // 这里使用原始版本，因为原版也这样做。:P
        // #getRandomItemsRaw 为我们调用 consumer#accept 来处理掷骰结果。
        table.getRandomItemsRaw(context, consumer);
    }
}
```

接下来，我们为战利品条目创建一个 `MapCodec`：

```java
// 这作为常量放在 EntityLootEntry 中。
public static final MapCodec<EntityLootEntry> CODEC = RecordCodecBuilder.mapCodec(inst ->
        // 添加我们自己的字段。
        inst.group(
                        // 一个引用实体类型 id 的值。
                        BuiltInRegistries.ENTITY_TYPE.holderByNameCodec().fieldOf("entity").forGetter(e -> e.entity)
                )
                // 添加公共字段：weight、display、conditions 和 functions。
                .and(singletonFields(inst))
                .apply(inst, EntityLootEntry::new)
);
```

然后我们在[注册][registries]中使用这个编解码器：

```java
public static final DeferredRegister<LootPoolEntryType> LOOT_POOL_ENTRY_TYPES =
        DeferredRegister.create(Registries.LOOT_POOL_ENTRY_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootPoolEntryType> ENTITY_LOOT =
        LOOT_POOL_ENTRY_TYPES.register("entity_loot", () -> new LootPoolEntryType(EntityLootEntry.CODEC));
```

最后，在我们的战利品条目类中，必须重写 `getType()`：

```java
public class EntityLootEntry extends LootPoolSingletonContainer {
    // 其他内容在此

    @Override
    public LootPoolEntryType getType() {
        return ENTITY_LOOT.get();
    }
}
```

## 自定义数字提供器

要创建自定义数字提供器，请实现 `NumberProvider` 接口。为了举例说明，假设我们想创建一个改变所提供数字符号的数字提供器：

```java
// 我们接受另一个数字提供器作为基础。
public record InvertedSignProvider(NumberProvider base) implements NumberProvider {
    public static final MapCodec<InvertedSignProvider> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            NumberProviders.CODEC.fieldOf("base").forGetter(InvertedSignProvider::base)
    ).apply(inst, InvertedSignProvider::new));

    // 返回一个浮点值。根据需要上下文和记录参数。
    @Override
    public float getFloat(LootContext context) {
        return -this.base.getFloat(context);
    }

    // 返回一个整数值。根据需要上下文和记录参数。
    // 重写此方法是可选的，默认实现会舍入 #getFloat 的结果。
    @Override
    public int getInt(LootContext context) {
        return -this.base.getInt(context);
    }

    // 返回此提供器使用的战利品上下文参数集合。更多信息见下文。
    // 因为我们有一个基础值，所以我们直接委托给基础值。
    @Override
    public Set<ContextKey<?>> getReferencedContextParams() {
        return this.base.getReferencedContextParams();
    }
}
```

与自定义战利品条目类型类似，我们在[注册][registries]中使用这个编解码器：

```java
public static final DeferredRegister<LootNumberProviderType> LOOT_NUMBER_PROVIDER_TYPES =
        DeferredRegister.create(Registries.LOOT_NUMBER_PROVIDER_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootNumberProviderType> INVERTED_SIGN =
        LOOT_NUMBER_PROVIDER_TYPES.register("inverted_sign", () -> new LootNumberProviderType(InvertedSignProvider.CODEC));
```

类似地，在我们的数字提供器类中，必须重写 `getType()`：

```java
public record InvertedSignProvider(NumberProvider base) implements NumberProvider {
    // 其他内容在此

    @Override
    public LootNumberProviderType getType() {
        return INVERTED_SIGN.get();
    }
}
```

## 自定义基于等级的值

可以通过在记录中实现 `LevelBasedValue` 接口来创建自定义 `LevelBasedValue`。再次为了示例，假设我们想要反转另一个 `LevelBasedValue` 的输出：

```java
public record InvertedSignLevelBasedValue(LevelBasedValue base) implements LevelBaseValue {
    public static final MapCodec<InvertedLevelBasedValue> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            LevelBasedValue.CODEC.fieldOf("base").forGetter(InvertedLevelBasedValue::base)
    ).apply(inst, InvertedLevelBasedValue::new));

    // 执行我们的操作。
    @Override
    public float calculate(int level) {
        return -this.base.calculate(level);
    }

    // 与 NumberProviders 不同，我们不返回注册的类型，而是直接返回编解码器。
    @Override
    public MapCodec<InvertedLevelBasedValue> codec() {
        return CODEC;
    }
}
```

再次，我们在[注册][registries]中使用编解码器，不过这次是直接使用：

```java
public static final DeferredRegister<MapCodec<? extends LevelBasedValue>> LEVEL_BASED_VALUES =
        DeferredRegister.create(Registries.ENCHANTMENT_LEVEL_BASED_VALUE_TYPE, ExampleMod.MOD_ID);

public static final Supplier<MapCodec<? extends LevelBasedValue>> INVERTED_SIGN =
        LEVEL_BASED_VALUES.register("inverted_sign", () -> InvertedSignLevelBasedValue.CODEC);
```

## 自定义战利品条件

首先，我们创建实现 `LootItemCondition` 的战利品物品条件类。为了举例，假设我们只希望条件在杀死生物的玩家拥有特定经验等级时通过：

```java
public record HasXpLevelCondition(int level) implements LootItemCondition {
    // 添加此条件需要的上下文。在我们的案例中，这将是玩家必须拥有的经验等级。
    public static final MapCodec<HasXpLevelCondition> CODEC = RecordCodecBuilder.mapCodec(inst -> inst.group(
            Codec.INT.fieldOf("level").forGetter(HasXpLevelCondition::level)
    ).apply(inst, HasXpLevelCondition::new));
    
    // 在此评估条件。从提供的 LootContext 中获取所需的战利品上下文参数。
    // 在我们的案例中，我们希望 KILLER_ENTITY 至少拥有我们要求的等级。
    @Override
    public boolean test(LootContext context) {
        @Nullable
        Entity entity = context.getOptionalParameter(LootContextParams.KILLER_ENTITY);
        return entity instanceof Player player && player.experienceLevel >= level; 
    }
    
    // 告诉游戏我们期望从战利品上下文中获得哪些参数。用于验证。
    @Override
    public Set<ContextKey<?>> getReferencedContextParams() {
        return ImmutableSet.of(LootContextParams.KILLER_ENTITY);
    }
}
```

我们可以使用条件的编解码器将条件类型[注册][registries]到注册表：

```java
public static final DeferredRegister<LootItemConditionType> LOOT_CONDITION_TYPES =
        DeferredRegister.create(Registries.LOOT_CONDITION_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootItemConditionType> MIN_XP_LEVEL =
        LOOT_CONDITION_TYPES.register("min_xp_level", () -> new LootItemConditionType(HasXpLevelCondition.CODEC));
```

完成之后，我们需要在条件中重写 `#getType` 并返回注册的类型：

```java
public record HasXpLevelCondition(int level) implements LootItemCondition {
    // 其他内容在此

    @Override
    public LootItemConditionType getType() {
        return MIN_XP_LEVEL.get();
    }
}
```

## 自定义战利品函数

首先，我们创建扩展 `LootItemFunction` 的类。`LootItemFunction` 扩展了 `BiFunction<ItemStack, LootContext, ItemStack>`，因此我们想要做的是使用现有的物品堆和战利品上下文来返回一个新的、修改过的物品堆。然而，几乎所有的战利品函数并不直接扩展 `LootItemFunction`，而是扩展 `LootItemConditionalFunction`。这个类具有将战利品条件应用于函数的内置功能——只有在战利品条件满足时才应用函数。为了举例，让我们将一个带有指定等级的随机附魔应用到物品上：

```java
// 代码改编自原版的 EnchantRandomlyFunction 类。
// LootItemConditionalFunction 是一个抽象类，不是接口，因此我们不能在此使用记录。
public class RandomEnchantmentWithLevelFunction extends LootItemConditionalFunction {
    // 我们的上下文：一个可选的附魔列表，以及一个等级。
    private final Optional<HolderSet<Enchantment>> enchantments;
    private final int level;
    // 我们的编解码器。
    public static final MapCodec<RandomEnchantmentWithLevelFunction> CODEC =
            // #commonFields 添加 conditions 字段。
            RecordCodecBuilder.mapCodec(inst -> commonFields(inst).and(inst.group(
                    RegistryCodecs.homogeneousList(Registries.ENCHANTMENT).optionalFieldOf("enchantments").forGetter(e -> e.enchantments),
                    Codec.INT.fieldOf("level").forGetter(e -> e.level)
            ).apply(inst, RandomEnchantmentWithLevelFunction::new));
    
    public RandomEnchantmentWithLevelFunction(List<LootItemCondition> conditions, Optional<HolderSet<Enchantment>> enchantments, int level) {
        super(conditions);
        this.enchantments = enchantments;
        this.level = level;
    }
    
    // 运行我们的附魔应用逻辑。大部分内容复制自 EnchantRandomlyFunction#run。
    @Override
    public ItemStack run(ItemStack stack, LootContext context) {
        RandomSource random = context.getRandom();
        List<Holder<Enchantment>> stream = this.enchantments
                .map(HolderSet::stream)
                .orElseGet(() -> context.getLevel().registryAccess().lookupOrThrow(Registries.ENCHANTMENT).listElements().map(Function.identity()))
                .filter(e -> e.value().canEnchant(stack))
                .toList();
        Optional<Holder<Enchantment>> optional = Util.getRandomSafe(list, random);
        if (optional.isEmpty()) {
            LOGGER.warn("Couldn't find a compatible enchantment for {}", stack);
        } else {
            Holder<Enchantment> enchantment = optional.get();
            if (stack.is(Items.BOOK)) {
                stack = new ItemStack(Items.ENCHANTED_BOOK);
            }
            stack.enchant(enchantment, Mth.nextInt(random, enchantment.value().getMinLevel(), enchantment.value().getMaxLevel()));
        }
        return stack;
    }
}
```

然后我们可以使用函数的编解码器将函数类型[注册][registries]到注册表：

```java
public static final DeferredRegister<LootItemFunctionType<?>> LOOT_FUNCTION_TYPES =
        DeferredRegister.create(Registries.LOOT_FUNCTION_TYPE, ExampleMod.MOD_ID);

public static final Supplier<LootItemFunctionType<RandomEnchantmentWithLevelFunction>> RANDOM_ENCHANTMENT_WITH_LEVEL =
        LOOT_FUNCTION_TYPES.register("random_enchantment_with_level", () -> new LootItemFunctionType(RandomEnchantmentWithLevelFunction.CODEC));
```

完成之后，我们需要在条件中重写 `#getType` 并返回注册的类型：

```java
public class RandomEnchantmentWithLevelFunction extends LootItemConditionalFunction {
    // 其他内容在此

    @Override
    public LootItemFunctionType<?> getType() {
        return RANDOM_ENCHANTMENT_WITH_LEVEL.get();
    }
}
```

[codec]: ../../../datastorage/codecs.md
[entity]: ../../../entities/index.md
[registries]: ../../../concepts/registries.md#methods-for-registering