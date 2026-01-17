---
sidebar_position: 0
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# NeoForge 用户指南

无论您是普通玩家、整合包开发者还是服务器管理员，本指南都旨在帮助您完成准备电脑运行 NeoForge(原名称) 的过程，并回答一些常见问题。

本指南主要帮助您使用 Mojang 提供的官方 Minecraft 启动器安装 NeoForge(原名称)。也有一些第三方启动器，它们大部分可以为您自动化此过程，其中一些在[第三方启动器][launchers]一文中有所提及。

## Java

要运行 NeoForge(原名称)，您首先需要在电脑上安装 Java。Java 是 Minecraft 和 NeoForge(原名称) 使用的编程语言。虽然 Minecraft 使用启动器下载所需的 Java 版本，但 NeoForge(原名称) 要求您自己安装 Java。

所需的 Java 版本取决于您要运行的 Minecraft 版本：

| Minecraft 版本 | Java 版本 |
|:-----------------:|:------------:|
|   1.20.2-1.20.4   |      17      |
|   1.20.5-latest   |      21      |

:::info
虽然 NeoForge(原名称) 支持 Minecraft **1.20.1**，但我们建议在该版本上使用 Forge(原名称)，因为 Forge(原名称) 对 Minecraft 1.20.1 的支持时间更长。我们仅推荐在 Minecraft 1.20.2 及更新版本上使用 NeoForge(原名称)。
:::

### 检测 Java

_如果您确定没有安装 Java，可以跳转到[安装 Java][installingjava]。_

在许多情况下，您的系统可能已经安装了 Java。因此，您需要验证版本是否正确。

- 打开一个终端。具体操作方式取决于您运行的操作系统：

<Tabs defaultValue="windows">
  <TabItem value="windows" label="Windows">
在左下角的开始菜单中，搜索 `Command Prompt` 并按 Enter 键。
  </TabItem>
  <TabItem value="macos" label="MacOS">
打开 Finder。在 Finder 中，打开 Applications/Utilities 文件夹并双击 Terminal(终端)。
  </TabItem>
  <TabItem value="linux" label="Linux">
打开您的 Linux 发行版的终端。常见的名称是 `GNOME Terminal` 或 `Konsole`，但根据您的具体设置可能会有所不同。
  </TabItem>
</Tabs>

- 输入以下命令：`java -version` 并按 Enter 键。
- 如果显示错误，则表示未安装 Java，您可以跳转到[安装 Java][installingjava] 部分。
- 如果安装了 Java，您应该会看到以下输出（或类似内容）：
```
openjdk version "21.0.4" 2024-07-16 LTS
OpenJDK Runtime Environment Temurin-21.0.4+7 (build 21.0.4+7-LTS)
OpenJDK 64-Bit Server VM Temurin-21.0.4+7 (build 21.0.4+7-LTS, mixed mode, sharing)
```
- 验证 `version` 后面第一个数字是否与所需 Minecraft 版本对应的 Java 版本匹配。
  - 例如，`openjdk version "21.0.4" 2024-07-16 LTS` 是 Java 21 版本，因此适用于 Minecraft 1.20.5 及更新版本。
  - 如果版本不匹配，则您需要[安装正确的 Java 版本][installingjava]。
- 如果一切顺利，请继续[安装 NeoForge(原名称)][installingneoforge]。

### 安装 Java

安装 Java 的方式取决于您的操作系统。请务必确保获取正确版本的 Java，并确保获取 64 位版本，因为现代版本的 Minecraft 不再支持 32 位版本的 Java。

<Tabs defaultValue="windows">
  <TabItem value="windows" label="Windows">
从 [Adoptium 项目](https://adoptium.net/temurin/releases/?version=21&os=windows) 下载 JDK `.msi` 文件。在文件系统中找到您刚下载的 `.msi` 文件，双击它，并运行安装程序。
  </TabItem>
  <TabItem value="windows_server" label="Windows (Server)">
使用以下 `winget` 命令下载 JDK（必要时更改版本号）：

```
winget install -e --id=Microsoft.OpenJDK.21
```
  </TabItem>
  <TabItem value="macos" label="MacOS">
从 [Adoptium 项目](https://adoptium.net/temurin/releases/?version=21&os=mac) 下载 JDK `.pkg` 文件。在文件系统中找到您刚下载的 `.pkg` 文件，双击它，并运行安装程序。
  </TabItem>
  <TabItem value="linux" label="Linux">
打开您的 Linux 发行版的终端。常见的名称是 `GNOME Terminal` 或 `Konsole`，但根据您的具体设置可能会有所不同。

然后，使用系统的包管理器（例如 Ubuntu 和 Debian 上的 `apt`，CentOS 上的 `yum`，Fedora 上的 `dnf`，Arch 上的 `pacman`）安装 Java。包的准确名称可能有所不同，但像 `openjdk-21`（必要时替换版本号）这样的名称通常是可以的。

一些发行版还提供了安装 Java 的文档和/或额外工具：

<ul>
  <li>[在 Arch 上安装 Java(原名称)][arch]</li>
  <li>[在 Debian 上安装 Java(原名称)][debian]</li>
  <li>[在 Fedora 上安装 Java(原名称)][fedora]</li>
  <li>[在 Ubuntu 上安装 Java(原名称)][ubuntu]</li>
</ul>

  </TabItem>
</Tabs>

安装后，建议再次[检测 Java][testingforjava]，以确保一切顺利。

## 安装 NeoForge

成功安装 Java 后，您现在可以安装 NeoForge(原名称) 本身了。

- 如果您想单人游玩 NeoForge(原名称) 或加入 NeoForge(原名称) 服务器，请阅读[安装 NeoForge 客户端(原名称)][client]。
- 如果您想运行使用 NeoForge(原名称) 的服务器，请阅读[安装 NeoForge 服务器(原名称)][server]。

## 更多帮助

本指南涵盖了您作为 NeoForge(原名称) 用户可能遇到的大部分问题，但它并非详尽无遗，您可能会遇到此处未涵盖的问题。

常见问题可在[故障排除与常见问题解答(FAQ)][faq] 文章中找到。如需进一步支持，请参阅 FAQ 的[获取支持][support]部分。

[arch]: https://wiki.archlinux.org/title/Java
[client]: client.md
[debian]: https://wiki.debian.org/Java
[faq]: faq.md
[fedora]: https://docs.fedoraproject.org/en-US/quick-docs/installing-java
[installingjava]: #installing-java
[installingneoforge]: #installing-neoforge
[launchers]: launchers.md
[server]: server.md
[support]: faq.md#getting-support
[testingforjava]: #testing-for-java
[ubuntu]: https://ubuntu.com/tutorials/install-jre