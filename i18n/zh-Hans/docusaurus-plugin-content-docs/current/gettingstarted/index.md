---
sidebar_position: 1
---
# NeoForge 入门指南

本节包含关于如何设置 NeoForge 工作空间，以及如何运行和测试你的模组的信息。

## 先决条件

- 熟悉 Java 编程语言，特别是其面向对象、多态、泛型和函数式特性。
- 安装 Java 21 开发工具包(JDK) 和 64 位 Java 虚拟机(JVM)。NeoForge 推荐并官方支持 [Microsoft builds of OpenJDK][jdk]，但任何其他 JDK 也应该可以工作。

:::caution
确保你使用的是 64 位 JVM。一种检查方法是在终端中运行 `java -version`。Minecraft 不支持 32 位 JVM。
:::

- 熟悉你选择的集成开发环境(IDE)。
      - NeoForge 官方支持 [IntelliJ IDEA][intellij] 和 [Eclipse][eclipse]，两者都有集成的 Gradle 支持。然而，任何 IDE 都可以使用，从 Netbeans 或 Visual Studio Code 到 Vim 或 Emacs。
- 熟悉 [Git][git] 和 [GitHub][github]。这在技术上不是必需的，但这会让你的生活更轻松。

## 设置工作空间

- 访问 [Mod Generator](https://neoforged.net/mod-generator/) 网页，输入模组名称（可选模组ID）、包名、Minecraft 版本和 Gradle 插件（[ModDevGradle][mdg] 或 [NeoGradle][ng]），点击“下载模组项目”，然后解压下载的 ZIP 文件。
- 打开你的 IDE 并导入 Gradle 项目。Eclipse 和 IntelliJ IDEA 会自动为你完成。如果你的 IDE 不自动完成，你也可以通过 `gradlew` 终端命令来完成。
      - 第一次执行此操作时，Gradle 将下载 NeoForge 的所有依赖项，包括 Minecraft 本身，并对它们进行反编译。这可能需要相当长的时间（最多一小时，取决于你的硬件和网络强度）。
      - 每当你对 Gradle 文件进行更改时，都需要重新加载 Gradle 更改，可以通过 IDE 中的“重新加载 Gradle”按钮，或者再次通过 `gradlew` 终端命令。

## 自定义你的模组信息

你的模组的许多基本属性都可以在 `gradle.properties` 文件中更改。这包括基本内容，如模组名称或模组版本。更多信息，请参阅 `gradle.properties` 文件中的注释，或参见 [`gradle.properties` 文件的文档][properties]。

如果你想在此基础上修改构建过程，可以编辑 `build.gradle` 和 `settings.gradle` 文件。NeoForge 提供的 Gradle 插件，无论是 [ModDevGradle][mdg] 还是 [NeoGradle][ng]，都提供了几个配置选项，其中一些在构建脚本中以注释形式解释。

:::caution
只有在知道自己在做什么的情况下才编辑 `build.gradle` 和 `settings.gradle` 文件。所有基本属性都可以通过 `gradle.properties` 设置。
:::

## 构建和测试你的模组

要构建你的模组，请运行 `gradlew build`。这将在 `build/libs` 中输出一个名为 `<archivesBaseName>-<version>.jar` 的文件。`<archivesBaseName>` 和 `<version>` 是由 `build.gradle` 设置的属性，默认分别对应于 `gradle.properties` 文件中的 `mod_id` 和 `mod_version` 值；如果需要，可以在 `build.gradle` 中更改。生成的 JAR 文件可以放置在启用了 NeoForge 的 Minecraft 设置的 `mods` 文件夹中，或上传到模组分发平台。

要在测试环境中运行你的模组，你可以使用生成的运行配置或使用相关的任务（例如 `gradlew runClient`）。这将从相应的运行目录（例如 `runs/client` 或 `runs/server`）启动 Minecraft，以及指定的任何源集。默认的 MDK 包含 `main` 源集，因此 `src/main/java` 中编写的任何代码都将被应用。

### 服务器测试

如果你正在运行专用服务器，无论是通过运行配置还是 `gradlew runServer`，服务器都会立即关闭。你需要通过编辑运行目录中的 `eula.txt` 文件来接受 Minecraft 最终用户许可协议(EULA)。

接受后，服务器将加载并在 `localhost`（或默认的 `127.0.0.1`）下可用。但是，你仍然无法加入，因为服务器默认会进入在线模式，这需要身份验证（而开发玩家没有）。要解决此问题，请再次停止服务器，并将 `server.properties` 文件中的 `online-mode` 属性设置为 `false`。现在，启动你的服务器，你应该能够连接了。

:::tip
你应该始终在专用服务器环境中测试你的模组。这包括[仅客户端模组][client]，因为这些模组在服务器上加载时不应做任何事情。
:::

[client]: ../concepts/sides.md
[eclipse]: https://www.eclipse.org/downloads/
[git]: https://www.git-scm.com/
[github]: https://github.com/
[intellij]: https://www.jetbrains.com/idea/
[jdk]: https://learn.microsoft.com/en-us/java/openjdk/download#openjdk-21
[mdg]: https://github.com/neoforged/ModDevGradle
[modgen]: https://neoforged.net/mod-generator/
[ng]: https://github.com/neoforged/NeoGradle
[properties]: modfiles.md#gradleproperties