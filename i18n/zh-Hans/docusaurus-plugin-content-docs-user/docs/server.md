---
sidebar_position: 2
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 安装 NeoForge 服务器

_本文假定您[已安装正确版本的 Java][java]。_

## 安装

<Tabs defaultValue="linux">
  <TabItem value="linux" label="Linux/MacOS">
在 Linux 或 MacOS 上运行 NeoForge(原名称) 服务器需要您具备使用基本终端命令的能力。

- 导航到您想要安装服务器的文件夹。
- 使用 `wget` 从 Maven 仓库下载安装程序 `.jar` 文件（请根据需要替换版本号和 `-beta` 标签）：
```shell
wget https://maven.neoforged.net/releases/net/neoforged/neoforge/21.4.111-beta/neoforge-21.4.111-beta-installer.jar
```
- 使用命令 `java -jar /path/to/neoforge-installer.jar --installServer` 运行安装程序。
- （可选）在新创建的 `user_jvm_args.txt` 文件中修改分配给服务器的内存大小，以及其他可能的 JVM 参数。更多信息请参阅文件内的注释。
- 使用 `./run.sh` 启动服务器。它会在首次运行时关闭。
- 打开 `eula.txt` 文件，将 `eula=false` 改为 `eula=true`。这样做即表示您同意在运营服务器时遵守 [Minecraft EULA(最终用户许可协议)][eula]。
- 再次使用 `./run.sh` 启动服务器。

您的服务器现在应该可以运行了。您可能需要进行一些环境设置，例如在您的系统和/或网络防火墙中开放 Minecraft 端口（默认为 25565）。如果您计划让服务器 24/7 运行，还应安排一个 cronjob 或类似任务进行定期重启。
  </TabItem>
  <TabItem value="windows" label="Windows">
在 Windows 上运行 NeoForge(原名称) 服务器需要您具备使用基本终端命令的能力。

- 导航到您想要安装服务器的文件夹。
- 使用 `curl` 从 Maven 仓库下载安装程序 `.jar` 文件（请根据需要替换版本号和 `-beta` 标签）：
```shell
curl -O https://maven.neoforged.net/releases/net/neoforged/neoforge/21.4.111-beta/neoforge-21.4.111-beta-installer.jar
```
- 使用命令 `java -jar /path/to/neoforge-installer.jar --installServer` 运行安装程序。
- （可选）在新创建的 `user_jvm_args.txt` 文件中修改分配给服务器的内存大小，以及其他可能的 JVM 参数。更多信息请参阅文件内的注释。
- 使用 `.\run.bat` 启动服务器。它会在首次运行时关闭。
- 打开 `eula.txt` 文件，将 `eula=false` 改为 `eula=true`。这样做即表示您同意在运营服务器时遵守 [Minecraft EULA(最终用户许可协议)][eula]。
- 再次使用 `.\run.bat` 启动服务器。

您的服务器现在应该可以运行了。您可能需要进行一些环境设置，例如在您的系统和/或网络防火墙中开放 Minecraft 端口（默认为 25565）。如果您计划让服务器 24/7 运行，还应在 Windows 任务计划程序中安排定期重启任务。
  </TabItem>
</Tabs>

## 添加模组

首次启动游戏后，会出现一个 `mods` 文件夹。您应将模组文件放在此处。

模组文件只应从可信来源下载。我们通常建议您从 [CurseForge][curseforge] 或 [Modrinth][modrinth] 获取模组。

## 更新

要更新服务器的 NeoForge(原名称) 版本，只需按照与旧版本相同的方式下载并运行新版本的安装程序即可。安装程序将自动在需要的地方替换旧版本的使用。

:::danger
在更新 NeoForge(原名称) 或模组之前，请务必备份您的世界！
:::

## 安装整合包

在服务器上安装预制的整合包通常需要一些额外的设置。由于整合包是[第三方启动器][launchers]的功能，因此在服务器上安装整合包需要使用此类启动器。

- 按照上述说明安装一个服务器。
- 在您的启动器中，如果整合包提供“服务器包”，请安装一个单独的服务器包实例。否则，安装一个单独的普通整合包实例。
- 将新安装实例的所有内容移动到服务器的游戏文件夹中。
- （可选）从服务器中移除客户端模组。
  - 什么构成客户端模组并不总是清晰的，通常需要反复试验。客户端模组通常处理视觉方面的事情，例如启用着色器或额外的资源包功能。
  - 如果您安装了服务器包，这一步应该已经为您完成了。
- 使用 `./run.sh` (Linux) 或 `.\run.bat` (Windows) 启动服务器。

## 另请参阅

- [Minecraft Wiki][wiki] 上的[设置 Java 版服务器][wiki1]和[服务器维护][wiki2] - 请注意，这些文章讨论的是设置原版 Minecraft 服务器，而非模组服务器。

[curseforge]: https://www.curseforge.com/minecraft/search?class=mc-mods
[eula]: https://www.minecraft.net/en-us/eula
[java]: index.md#java
[launchers]: launchers.md
[modrinth]: https://modrinth.com/mods
[wiki]: https://minecraft.wiki/
[wiki1]: https://minecraft.wiki/w/Tutorial:Setting_up_a_Java_Edition_server
[wiki2]: https://minecraft.wiki/w/Tutorial:Server_maintenance