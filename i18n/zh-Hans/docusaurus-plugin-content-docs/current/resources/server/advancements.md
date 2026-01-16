# 进度(Advancements)

进度是玩家可以完成的类似任务的目标。进度基于进度标准(advancement criteria)授予，并且可以在完成时运行特定行为。

可以通过在您的命名空间(namespace)的`advancement`子文件夹中创建一个JSON文件来添加新的进度。例如，如果我们想为模组ID(mod id)为`examplemod`的模组添加一个名为`example_name`的进度，它将位于`data/examplemod/advancement/example_name.json`。进度的ID将相对于`advancement`目录，因此对于我们的例子，它将是`examplemod:example_name`。可以选择任何名称，游戏将自动拾取该进度。仅当您想添加新标准或从代码触发特定标准时才需要Java代码（见下文）。

## 规范(Specification)

一个进度JSON文件可能包含以下条目：

- `parent`：此进度的父进度ID。循环引用将被检测并导致加载失败。可选；如果不存在，此进度将被视为根进度(root advancement)。根进度是未设置父级的进度。它们将是其[进度树][tree]的根。
- `display`：包含用于在进度GUI中显示进度的多个属性的对象。可选；如果不存在，此进度将不可见，但仍可被触发。
    - `icon`：一个[物品堆叠的JSON表示][itemstackjson]。
    - `title`：用作进度标题的[文本组件][text]。
    - `description`：用作进度描述的[文本组件][text]。
    - `frame`：进度的边框类型。接受`challenge`、`goal`和`task`。可选，默认为`task`。
    - `background`：用于树背景的纹理。这是相对于`textures`目录的，即不应包含`textures/`文件夹前缀。可选，默认为缺失纹理(missing texture)。仅对根进度有效。
    - `show_toast`：完成时是否在右上角显示提示(toast)。可选，默认为true。
    - `announce_to_chat`：是否在聊天中宣布进度完成。可选，默认为true。
    - `hidden`：是否在完成之前从进度GUI中隐藏此进度及其所有子进度。对根进度本身无影响，但仍会隐藏其所有子进度。可选，默认为false。
- `criteria`：此进度应跟踪的标准(criteria)映射。每个标准由其映射键标识。Minecraft添加的标准触发器列表可在`CriteriaTriggers`类中找到，JSON规范可在[Minecraft Wiki][triggers]上找到。有关实现自定义标准或从代码触发标准，请参见下文。
- `requirements`：决定需要哪些标准的列表的列表。这是一个OR列表的列表，它们被AND在一起，换句话说，每个子列表必须至少有一个匹配的标准。可选，默认为所有标准都是必需的。
- `rewards`：表示完成此进度时授予的奖励的对象。可选，对象的所有值也是可选的。
    - `experience`：授予玩家的经验值数量。
    - `recipes`：要解锁的[配方(recipe)]ID列表。
    - `loot`：要掷骰并给予玩家的[战利品表(loot tables)][loottable]列表。
    - `function`：要运行的[函数(function)]。如果要运行多个函数，请创建一个运行所有其他函数的包装函数。
- `sends_telemetry_event`：决定完成此进度时是否应收集遥测数据。仅当在`minecraft`命名空间内时才实际生效。可选，默认为false。
- `neoforge:conditions`：NeoForge添加。必须通过才能使进度加载的[条件(conditions)]列表。可选。

### 进度树(Advancement Trees)

进度文件可以分组在目录中，这会告诉游戏创建多个进度选项卡。一个进度选项卡可能包含一个或多个进度树，具体取决于根进度的数量。空的进度选项卡将自动隐藏。

:::提示
Minecraft每个选项卡始终只有一个根进度，并且始终将根进度称为`root`。建议遵循此做法。
:::

## 标准触发器(Criteria Triggers)

要解锁进度，必须满足指定的标准。标准通过触发器(triggers)跟踪，当关联的操作发生时，触发器会从代码中执行（例如，当玩家杀死指定的[实体(entity)]时，`player_killed_entity`触发器执行）。每当进度加载到游戏中时，都会读取定义的标准并将其作为监听器添加到触发器。当触发器执行时，所有具有相应标准监听器的进度都会重新检查是否完成。如果进度完成，则移除监听器。

自定义标准触发器由两部分组成：触发器（通过在代码中调用`#trigger`来激活）和定义触发器应在何种条件下授予标准的实例(instance)。触发器扩展`SimpleCriterionTrigger<T>`，而实例实现`SimpleCriterionTrigger.SimpleInstance`。泛型值`T`代表触发器实例类型。

### `SimpleCriterionTrigger.SimpleInstance`

`SimpleCriterionTrigger.SimpleInstance`表示`criteria`对象中定义的单个标准。触发器实例负责保存定义的条件，并返回输入是否匹配条件。

条件通常通过构造函数传递。`SimpleCriterionTrigger.SimpleInstance`接口只需要一个函数，称为`#player`，它返回玩家必须满足的条件作为`Optional<ContextAwarePredicate>`。如果子类是具有此类型`player`参数的记录(record)（如下所示），则自动生成的`#player`方法就足够了。

```java
public record ExampleTriggerInstance(Optional<ContextAwarePredicate> player/*, 其他参数在此*/)
        implements SimpleCriterionTrigger.SimpleInstance {}
```

通常，触发器实例具有静态辅助方法，这些方法从实例的参数构造完整的`Criterion<T>`对象。这允许在数据生成期间轻松创建这些实例，但它们是可选的。

```java
// 在这个例子中，EXAMPLE_TRIGGER 是 DeferredHolder<CriterionTrigger<?>, ExampleTrigger>。
// 有关如何注册触发器，请参见下文。
public static Criterion<ExampleTriggerInstance> instance(ContextAwarePredicate player, ItemPredicate item) {
    return EXAMPLE_TRIGGER.get().createCriterion(new ExampleTriggerInstance(Optional.of(player), item));
}
```

最后，应添加一个方法，该方法接受当前数据状态并返回用户是否满足必要条件。玩家的条件已经通过`SimpleCriterionTrigger#trigger(ServerPlayer, Predicate)`检查。大多数触发器实例将此方法称为`#matches`。

```java
// 假设我们有一个额外的 ItemPredicate 参数。这可以是您需要的任何内容。
// 例如，这也可能是 Predicate<LivingEntity>。
public record ExampleTriggerInstance(Optional<ContextAwarePredicate> player, ItemPredicate predicate)
        implements SimpleCriterionTrigger.SimpleInstance {
    // 此方法对每个实例是唯一的，因此不被重写。
    // 参数可以是您需要正确匹配的任何内容，例如，这也可能是 LivingEntity。
    // 如果您只需要玩家以外的上下文，此方法也可以完全不接受参数。
    public boolean matches(ItemStack stack) {
        // 由于 ItemPredicate 匹配堆叠，我们在这里使用堆叠作为输入。
        return this.predicate.test(stack);
    }
}
```

### `SimpleCriterionTrigger`

`SimpleCriterionTrigger<T>`实现有两个目的：提供检查触发器实例并在成功时运行附加监听器的方法，以及指定用于序列化触发器实例(`T`)的[编解码器(codec)]。

首先，我们想添加一个方法，该方法接受我们需要的输入并调用`SimpleCriterionTrigger#trigger`来正确处理检查所有监听器。大多数触发器实例也将此方法命名为`#trigger`。重用我们上面的示例触发器实例，我们的触发器将如下所示：

```java
public class ExampleCriterionTrigger extends SimpleCriterionTrigger<ExampleTriggerInstance> {
    // 此方法对每个触发器是唯一的，因此不是要重写的方法
    public void trigger(ServerPlayer player, ItemStack stack) {
        this.trigger(player,
                // SimpleCriterionTrigger.SimpleInstance 子类内部的条件检查器方法
                triggerInstance -> triggerInstance.matches(stack)
        );
    }
}
```

触发器必须注册到`Registries.TRIGGER_TYPE` [注册表(registry)][registration]：

```java
public static final DeferredRegister<CriterionTrigger<?>> TRIGGER_TYPES =
        DeferredRegister.create(Registries.TRIGGER_TYPE, ExampleMod.MOD_ID);

public static final Supplier<ExampleCriterionTrigger> EXAMPLE_TRIGGER =
        TRIGGER_TYPES.register("example", ExampleCriterionTrigger::new);
```

然后，触发器必须通过重写`#codec`来定义用于序列化和反序列化触发器实例的[编解码器(codec)]。此编解码器通常在实例实现中创建为常量。

```java
public record ExampleTriggerInstance(Optional<ContextAwarePredicate> player/*, 其他参数在此*/)
        implements SimpleCriterionTrigger.SimpleInstance {
    public static final Codec<ExampleTriggerInstance> CODEC = ...;

    // ...
}

public class ExampleTrigger extends SimpleCriterionTrigger<ExampleTriggerInstance> {
    @Override
    public Codec<ExampleTriggerInstance> codec() {
        return ExampleTriggerInstance.CODEC;
    }

    // ...
}
```

对于之前带有`ContextAwarePredicate`和`ItemPredicate`的记录示例，编解码器可以是：

```java
public static final Codec<ExampleTriggerInstace> CODEC = RecordCodecBuilder.create(instance -> instance.group(
        EntityPredicate.ADVANCEMENT_CODEC.optionalFieldOf("player").forGetter(ExampleTriggerInstance::player),
        ItemPredicate.CODEC.fieldOf("item").forGetter(ExampleTriggerInstance::item)
).apply(instance, ExampleTriggerInstance::new));
```

### 调用标准触发器(Calling Criterion Triggers)

每当执行被检查的操作时，应该调用由我们的`SimpleCriterionTrigger`子类定义的`#trigger`方法。当然，您也可以调用在`CriteriaTriggers`中找到的原版触发器。

```java
// 在执行操作的某段代码中
// 再次说明，EXAMPLE_TRIGGER 是已注册的自定义标准触发器实例的提供者(Supplier)
public void performExampleAction(ServerPlayer player, additionalContextParametersHere) {
    // 在此处运行执行操作的代码
    EXAMPLE_TRIGGER.get().trigger(player, additionalContextParametersHere);
}
```

## 数据生成(Data Generation)

进度可以使用`AdvancementProvider`进行[数据生成(datagenned)][datagen]。`AdvancementProvider`接受`AdvancementSubProviders`s列表，这些子提供者实际上使用`Advancement.Builder`生成进度。

首先，在其中一个`GatherDataEvent`s中创建`AdvancementProvider`的实例：

```java
@SubscribeEvent // 在模组事件总线(mod event bus)上
public static void gatherData(GatherDataEvent.Client event) {
    // 如果添加数据包对象，首先调用 event.createDatapackRegistryObjects(...)

    event.createProvider((output, lookupProvider) -> new AdvancementProvider(
        output, lookupProvider,
        // 在此处添加生成器
        List.of(...)
    ));

     // 其他提供者
}
```

现在，下一步是用我们的生成器填充列表。为此，我们可以将生成器添加为类或lambda，然后将每个生成器的实例添加到构造函数参数中当前为空的列表中。

```java
// 类示例
public class MyAdvancementGenerator implements AdvancementSubProvider {

    @Override
    public void generate(HolderLookup.Provider registries, Consumer<AdvancementHolder> saver) {
        // 在此处生成您的进度。
    }
}

// 方法示例
public class ExampleClass {

    // 匹配 AdvancementSubProvider#generate 提供的参数
    public static void generateExampleAdvancements(HolderLookup.Provider registries, Consumer<AdvancementHolder> saver) {
        // 在此处生成您的进度。
    }
}

// 在其中一个 `GatherDataEvent`s 中
event.createProvider((output, lookupProvider) -> new AdvancementProvider(
    output, lookupProvider,
    // 在此处添加生成器
    List.of(
        // 将我们的生成器的实例添加到列表参数中。可以根据需要执行多次。
        // 拥有多个生成器纯粹是为了组织，所有功能都可以通过单个生成器实现。
        new MyAdvancementGenerator(),
        ExampleClass::generateExampleAdvancements
    )
));
```

要生成进度，您需要使用`Advancement.Builder`：

```java
// 所有方法都遵循构建器模式，意味着可以且鼓励链式调用。
// 为了解释的可读性，这里不会进行链式调用。

// 使用静态方法 #advancement() 创建进度构建器。
// 使用 #advancement() 会自动启用遥测事件。如果不需要此功能，
// 可以使用 #recipeAdvancement()，没有其他功能差异。
Advancement.Builder builder = Advancement.Builder.advancement();

// 设置进度的父级。您可以使用已经生成的另一个进度，
// 或使用静态方法 AdvancementSubProvider#createPlaceholder 创建占位符进度。
builder.parent(AdvancementSubProvider.createPlaceholder("minecraft:story/root"));

// 设置进度的显示属性。这可以是 DisplayInfo 对象，
// 或者直接传入值。如果直接传入值，将为您创建 DisplayInfo 对象。
builder.display(
        // 进度图标。可以是 ItemStack 或 ItemLike。
        new ItemStack(Items.GRASS_BLOCK),
        // 进度标题和描述。别忘了为这些添加翻译！
        Component.translatable("advancements.examplemod.example_advancement.title"),
        Component.translatable("advancements.examplemod.example_advancement.description"),
        // 背景纹理。如果不需要背景纹理（对于非根进度），请使用 null。
        null,
        // 边框类型。有效值为 AdvancementType.TASK、CHALLENGE 或 GOAL。
        AdvancementType.GOAL,
        // 是否显示进度提示(toast)。
        true,
        // 是否在聊天中宣布进度。
        true,
        // 进度是否应隐藏。
        false
);

// 进度奖励构建器。可以用四种奖励类型中的任何一种创建，并且可以使用以 add 为前缀的方法添加更多奖励。
// 这也可以预先构建，然后得到的 AdvancementRewards 可以在多个进度构建器中重用。
builder.rewards(
    // 或者，使用 addExperience() 添加到现有的构建器。
    AdvancementRewards.Builder.experience(100)
    // 或者，使用 loot() 创建新的构建器。
    .addLootTable(ResourceKey.create(Registries.LOOT_TABLE, ResourceLocation.fromNamespaceAndPath("minecraft", "chests/igloo")))
    // 或者，使用 recipe() 创建新的构建器。
    .addRecipe(ResourceKey.create(Registries.RECIPE, ResourceLocation.fromNamespaceAndPath("minecraft", "iron_ingot")))
    // 或者，使用 function() 创建新的构建器。
    .runs(ResourceLocation.fromNamespaceAndPath("examplemod", "example_function"))
);

// 将具有给定名称的标准添加到进度中。使用相应触发器实例的静态方法。
builder.addCriterion("pickup_dirt", InventoryChangeTrigger.TriggerInstance.hasItems(Items.DIRT));

// 添加需求(requirements)处理器。Minecraft 原生提供 allOf() 和 anyOf()，更复杂的需求
// 必须手动实现。仅在有两个或更多标准时才有效果。
builder.requirements(AdvancementRequirements.allOf(List.of("pickup_dirt")));

// 使用给定的资源位置(ResourceLocation)将进度保存到磁盘。这将返回一个 AdvancementHolder，
// 可以存储在变量中并被其他进度构建器用作父级。
builder.save(saver, ResourceLocation.fromNamespaceAndPath("examplemod", "example_advancement"));
```

[编解码器(codec)]: ../../datastorage/codecs.md
[条件(conditions)]: conditions.md
[数据生成(datagen)]: ../index.md#data-generation
[实体(entity)]: ../../entities/index.md
[函数(function)]: https://minecraft.wiki/w/Function_(Java_Edition)
[物品堆叠JSON表示(itemstackjson)]: ../../items/index.md#json-representation
[战利品表(loottable)]: loottables/index.md
[配方(recipe)]: recipes/index.md
[注册(registration)]: ../../concepts/registries.md#methods-for-registering
[根进度(root)]: #root-advancements
[文本(text)]: ../client/i18n.md#components
[进度树(tree)]: #advancement-trees
[触发器(triggers)]: https://minecraft.wiki/w/Advancement/JSON_format#List_of_triggers