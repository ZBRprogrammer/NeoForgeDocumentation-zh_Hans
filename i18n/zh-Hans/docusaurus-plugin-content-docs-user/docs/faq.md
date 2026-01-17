---
sidebar_position: 4
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 故障排除与常见问题解答(Troubleshooting & FAQ)

此页面列出了一些安装或运行 NeoForge(NeoForge) 整合包时的常见问题。如果此处未涵盖您的问题，请参阅[获取支持][support]部分。

## 安装 Java 或 NeoForge

### 我下载了安装程序文件，但无法运行它！

请参阅[安装 Java][installjava]部分。是的，即使您已经安装了 Minecraft: Java Edition。

### 我安装了 Java，但版本仍然不对！

这可能意味着您的 PATH 变量未设置或设置不正确。PATH 变量是一个系统变量，它告诉您的计算机在哪里找到 Java。要修复此问题，请根据您的操作系统执行以下操作：

<Tabs defaultValue="windows">
  <TabItem value="windows" label="Windows">
下载并运行 [Jarfix 程序][jarfix]。
  </TabItem>
  <TabItem value="macos" label="MacOS">
打开 Finder。在 Finder 中，打开 Applications/Utilities 文件夹并双击 Terminal。

运行以下命令：

- `echo export "JAVA_HOME=\$(/usr/libexec/java_home)" >> ~/.zshenv`
- `echo export "PATH=$PATH:$JAVA_HOME/bin" >> ~/.zshenv`

然后，关闭 Terminal 并重试。
  </TabItem>
  <TabItem value="linux" label="Linux">
许多常见的 Linux 发行版都有专门管理 PATH 变量的程序。请参阅它们各自的 Java 安装说明：

<ul>
  <li>[在 Arch 上安装 Java][arch]</li>
  <li>[在 Debian 上安装 Java][debian]</li>
  <li>[在 Fedora 上安装 Java][fedora]</li>
  <li>[在 Ubuntu 上安装 Java][ubuntu]</li>
</ul>

如果您使用不同的发行版，您需要查找针对您特定发行版的方法，或者通过控制台手动设置 PATH 变量。

为此，请打开您的 Linux 发行版的终端。常见的名称是 `GNOME Terminal` 或 `Konsole`，但这可能因您的具体设置而异。

找到 Java 存储的位置。通常，这类似于 `/usr/lib/jvm/java-21-openjdk-amd64`。

运行以下命令：

- `echo export "JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64/" >> ~/.bashrc`（如果需要，将路径替换为实际路径）
- `echo export "PATH=$PATH:$JAVA_HOME/bin" >> ~/.bashrc`

然后，关闭终端并重试。
  </TabItem>
</Tabs>

### 我收到错误信息“Could not find or load main class @user_jvm_args.txt”，该怎么办？

这通常有两个原因：

- 您正在运行过时的 Java 版本。请参阅[安装 Java][installjava]部分。
- 您的计算机上安装了多个 Java 版本，可能您刚刚安装了其中一个，但您的计算机仍在使用旧版本。这意味着您的 PATH 变量设置不正确。请按照[此处][wrongjava]概述的步骤操作。

## 运行 NeoForge

<!--
本小节用作 links.neoforged.net/early-display-errors 的目标。避免更改标题，或在必要时同时更新短链接目标。
-->
### 早期显示错误(Early Display Errors)

如果您正在阅读本文，您很可能是被启动期间早期出现的错误消息重定向至此。原因是在尝试构建“早期加载进度”窗口时出现了问题，这通常意味着您系统的图形设置有问题。

- 参考 [Minecraft 自身的视频和图形问题常见问题解答][mcdriver] 并按照其中概述的步骤操作。
- 如果没有帮助，请尝试更新您的图形驱动程序。请参阅下面的[我遇到视觉问题！][visual]。
- 如果那也没有帮助，[请来我们的 Discord 服务器上与我们交流][support]，因为很明显出现了问题。但是，作为临时解决方案：
    - 打开您的实例文件夹。如果您使用[第三方启动器][launcher]，它们通常会提供一个标记为类似“打开实例文件夹”的按钮。如果您使用官方启动器，请使用您在[模组配置][clientinstall]中设置的文件夹。
    - 进入 `config` 文件夹。
    - 使用您最喜欢的文本编辑器（例如 Windows 记事本）打开 `fml.toml` 文件。
    - 将行 `earlyWindowControl=true` 更改为 `earlyWindowControl=false`。
    - 保存更改后的文件。
    - 尝试再次启动游戏。

### 我的游戏很卡！我该怎么办？

确定问题是出在服务器还是客户端（或两者都有）。

- 如果您的游戏以低 FPS（每秒帧数）运行，请阅读[我的客户端很卡！我该怎么办？][clientlag]
- 如果您的游戏以足够好的 FPS 运行，但您的世界很卡（机器运行太慢，生物在视觉上滞后等），请阅读[我的服务器很卡！我该怎么办？][serverlag]

### 我的服务器很卡！我该怎么办？

这可能有多重原因。常见的罪魁祸首是附近有太多的实体（= 生物或掉落的物品）或方块实体（= 箱子、机器等）。

如果您想要确切了解是什么导致了卡顿，请尝试使用 [Spark][spark] 模组。Spark 是一个分析器，可以告诉您哪些代码路径花费了多长时间。使用方法：

- 加入一个世界。
- 运行 `/spark profiler` 命令。
- 等待几分钟。
- 运行 `/spark profiler --stop` 命令。
- 打开链接并查看，或者如果您无法理解，请将其发布到 [Discord 服务器][support] 的 `#user-support` 频道，我们将尽力帮助您。

### 我的客户端很卡！我该怎么办？

这通常是因为 Minecraft 本身并不是一个优化得很好的游戏，而模组可能会使情况更糟。

如果您正在运行着色器，很可能您的显卡不够强大，无法支持它们。尝试禁用着色器，看看是否解决了问题。

如果您想要确切了解是什么导致了卡顿，请尝试使用 [Spark][spark] 模组。Spark 是一个分析器，可以告诉您哪些代码路径花费了多长时间。使用方法：

- 加入一个世界。
- 运行 `/sparkc profiler` 命令。（注意使用 `/sparkc`，与 `/spark` 不同，用于分析客户端。）
- 等待几分钟。
- 运行 `/sparkc profiler --stop` 命令。
- 打开链接并查看，或者如果您无法理解，请将其发布到 [Discord 服务器][support] 的 `#user-support` 频道，我们将尽力帮助您。

### 如何找到有问题的模组？

如果您需要在没有任何线索的情况下查找有问题的模组，最好的方法可能是使用二分查找。二分查找是一种常见的方法，用于在许多事物中查找有问题的项目，而无需逐个排查。针对模组，步骤如下：

1. 移除一半现有模组，并将它们放入另一个文件夹。
    - 确保依赖项（需要其他模组的模组）保持完整。
2. 运行游戏。
3. 查明问题是否仍然存在。
    - 如果是：用当前模组从步骤 1 重复。
    - 如果否：将最近添加的模组与最近搁置的模组交换，然后从步骤 1 重复。
4. 重复此过程，直到找到有问题的模组。

### 我遇到视觉问题！

视觉问题通常由过时的图形驱动程序引起。请根据您的显卡品牌更新图形驱动程序：[AMD][amd] | [Intel][intel] | [NVIDIA][nvidia]

如果更新图形驱动程序后您的问题仍然存在，请参阅[获取支持][support]。

## 获取支持(Getting Support)

如果此处未涵盖您的问题，您可以随时加入 [Discord 服务器][discord] 并在 `#user-support` 频道中寻求帮助。这样做时，请尽可能提供以下信息：

- 日志（一种特殊的文本文件）。
    - 如果您在安装期间遇到问题，日志位于安装程序本身相同的位置，其名称后附加了 `.jar`（或者如果您启用了文件扩展名，则为 `.log`）。
    - 如果您在游戏过程中遇到问题，日志位于您的实例文件夹中。进入 `logs` 文件夹并使用名为 `debug` 的文件，或者如果您启用了文件扩展名，则为 `debug.log`。
- 如果您制作了 [Spark 报告][sparkreport]，请提供它。

### 没有 `debug.log`！

如果您使用 [CurseForge 应用程序][curseforge] 玩游戏，该应用程序可能禁用了 `debug.log` 文件的创建。如果是这种情况，您需要重新启用它。为此：

- 打开设置（左下角的齿轮图标）。
- 在“游戏特定设置”下，导航到 Minecraft。
- 在“高级”下，打开“启用 Forge debug.log”选项。

然后，重新启动游戏以提供一个新的 `debug.log`。

如果您不使用 CurseForge，或者您使用 CurseForge 并已启用日志记录选项但仍然没有 `debug.log`，则可能是游戏在 `debug.log` 创建之前就崩溃了。在这种情况下，启动器日志可以帮助我们。

要获取启动器日志，请打开您的 [`.minecraft`][dotminecraft] 文件夹并找到 `launcher_log` 文件（如果您启用了文件扩展名，则为 `launcher_log.txt`）。

### 我的 .minecraft 文件夹在哪里？

根据您的操作系统，`.minecraft` 文件夹可以在以下位置找到：

<Tabs defaultValue="windows">
  <TabItem value="windows" label="Windows">
`%APPDATA%\.minecraft`
  </TabItem>
  <TabItem value="macos" label="MacOS">
`~/Library/Application Support/minecraft`
  </TabItem>
  <TabItem value="linux" label="Linux">
`~/.minecraft`
  </TabItem>
</Tabs>

[amd]: https://www.amd.com/en/support
[arch]: https://wiki.archlinux.org/title/Java
[clientinstall]: client.md#installing
[clientlag]: #my-client-is-lagging-what-can-i-do
[curseforge]: launchers.md#curseforge-app
[debian]: https://wiki.debian.org/Java
[discord]: https://discord.neoforged.net/
[dotminecraft]: #where-is-my-minecraft-folder
[fedora]: https://docs.fedoraproject.org/en-US/quick-docs/installing-java
[intel]: https://www.intel.com/content/www/us/en/support/detect.html
[installjava]: index.md#installing-java
[jarfix]: https://johann.loefflmann.net/en/software/jarfix/index.html
[launcher]: launchers.md
[mcdriver]: https://aka.ms/mcdriver
[nvidia]: https://www.nvidia.com/download/index.aspx
[serverlag]: #my-server-is-lagging-what-can-i-do
[spark]: https://www.curseforge.com/minecraft/mc-mods/spark
[sparkreport]: #my-game-is-lagging-what-can-i-do
[support]: #getting-support
[ubuntu]: https://ubuntu.com/tutorials/install-jre
[visual]: #i-am-having-visual-issues
[wrongjava]: #i-installed-java-but-im-still-getting-the-wrong-version