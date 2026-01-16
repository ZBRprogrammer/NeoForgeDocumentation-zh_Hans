# 构建你的模组

结构化的模组有利于维护、贡献，并提供对底层代码库更清晰的理解。下面列出了来自 Java、Minecraft 和 NeoForge 的一些建议。

:::note
你不必遵循下面的建议；你可以按照你认为合适的方式构建你的模组。但是，仍然强烈建议这样做。
:::

## 包结构

在构建你的模组时，选择一个独特的顶级包结构。许多程序员会对不同的类、接口等使用相同的名称。Java 允许类具有相同的名称，只要它们在不同的包中即可。因此，如果两个类具有相同名称的相同包，则只会加载一个，很可能会导致游戏崩溃。

```
a.jar
    - com.example.ExampleClass
b.jar
    - com.example.ExampleClass // 这个类通常不会被加载
```

在加载模块时，这一点甚至更为重要。如果在不同模块的两个包中有同名类文件，这将导致模组加载器在启动时崩溃，因为模组模块被导出到游戏和其他模组。

```
module A
    - package X
        - class I
        - class J
module B
    - package X // 这个包会导致模组加载器崩溃，因为已经有一个导出包 X 的模块
        - class R
        - class S
        - class T
```

因此，你的顶级包应该是你拥有的东西：域名、电子邮件地址、网站（的子域名）等。它甚至可以是你的名字或用户名，只要你能保证它在预期目标中是唯一可识别的。此外，顶级包也应与你的[组 ID][group] 匹配。

| 类型         | 值                   | 顶级包            |
|:------------:|:---------------------:|:------------------|
| 域名         | example.com           | `com.example`     |
| 子域名       | example.github.io     | `io.github.example` |
| 电子邮件     | example@gmail.com     | `com.gmail.example` |

下一个级别的包应该是你的模组 ID（例如 `com.example.examplemod`，其中 `examplemod` 是模组 ID）。这将保证，除非你有两个相同 ID 的模组（这绝不应该发生），否则你的包应该没有任何加载问题。

你可以在 [Oracle 的教程页面][naming] 上找到一些额外的命名约定。

### 子包组织

除了顶级包之外，强烈建议将模组的类分解到子包中。有两种主要的方法可以做到这一点：

- **按功能分组**：为具有共同目的的类创建子包。例如，方块可以在 `block` 下，物品在 `item` 下，实体在 `entity` 下，等等。Minecraft 本身使用类似的结构（有一些例外）。
- **按逻辑分组**：为具有共同逻辑的类创建子包。例如，如果你要创建一种新的工作台，你会将其方块、菜单、物品等放在 `feature.crafting_table` 下。

#### 客户端、服务器和数据包

通常，仅用于给定端或运行时的代码应与其他类隔离，放在单独的子包中。例如，与[数据生成][datagen] 相关的代码应放在 `data` 包中，仅在专用服务器上的代码应放在 `server` 包中。

强烈建议将[仅客户端代码][sides] 隔离在 `client` 子包中。这是因为专用服务器无法访问 Minecraft 中的任何仅客户端包，如果你的模组尝试访问它们，将会崩溃。因此，拥有一个专用的包提供了一个合理的检查，以验证你没有在模组内部跨端访问。

## 类命名方案

通用的类命名方案可以更容易地解读类的用途或轻松定位特定类。

类通常以其类型为后缀，例如：

- 名为 `PowerRing` 的 `Item` -> `PowerRingItem`。
- 名为 `NotDirt` 的 `Block` -> `NotDirtBlock`。
- `Oven` 的菜单 -> `OvenMenu`。

:::tip
Mojang 通常对所有类遵循类似的结构，除了实体。实体仅由其名称表示（例如 `Pig`、`Zombie` 等）。
:::

## 从多种方法中选择一种

执行特定任务的方法有很多：注册对象、监听事件等。通常建议通过使用单一方法来完成给定任务来保持一致。这提高了可读性，并避免了可能发生的奇怪交互或冗余（例如，你的事件监听器运行两次）。

[group]: index.md#the-group-id
[naming]: https://docs.oracle.com/javase/tutorial/java/package/namingpkgs.html
[datagen]: ../resources/index.md#data-generation
[sides]: ../concepts/sides.md