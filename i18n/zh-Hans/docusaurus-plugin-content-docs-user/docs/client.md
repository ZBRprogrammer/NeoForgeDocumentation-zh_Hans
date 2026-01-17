---
sidebar_position: 1
---

# 安装 NeoForge 客户端(NeoForge Client)

_本文假设您[已安装正确版本的 Java][java]，并且您使用（官方）Minecraft 启动器。如果您使用[第三方启动器][launchers]，请查阅其文档。_

## 安装

要安装 NeoForge：

- 关闭您的 Minecraft 启动器。
- 从 [NeoForged 网站][neoforged] 下载安装程序 `.jar`。
- 确保选中 `安装客户端` 并点击 `继续`。
- 打开 Minecraft 启动器。此时应出现一个 NeoForged 运行选项。

现在，虽然您可以使用该运行选项，但建议在处理模组实例时改用自定义启动配置，以便将模组与原版游戏隔离。为此，Minecraft 启动器在“安装”选项卡中提供了创建自定义配置的功能：

- 转到“安装”选项卡。
- 点击“新建安装”。
- 为您的新安装命名。
- 选择所需的 NeoForge 版本。
- 选择游戏目录。这应该是标准 [`.minecraft`][dotminecraft] 目录之外的一个单独文件夹。
- 按“安装”并启动该配置。

## 添加模组

启动游戏一次后，指定的游戏目录中会出现一个 `mods` 文件夹。您应将模组文件放在此处。

模组文件应仅从可信来源下载。我们通常建议您在 [CurseForge][curseforge] 或 [Modrinth][modrinth] 上获取模组。

## 更新

要更新您的 NeoForge 版本，只需按照上述说明下载并运行新 NeoForge 版本的安装程序。然后，转到启动器中的“安装”选项卡，将您的 NeoForge 配置中的版本更改为新版本。

:::danger
更新 NeoForge 或模组前，请务必备份您的世界！

要备份您的世界，请打开游戏目录并选择 `saves` 子文件夹。然后，将包含您世界名称的文件夹（例如 `New World`）复制到安全位置。

如果在更新过程中出现任何问题，请删除更新后的世界文件夹，将 NeoForge 和/或模组降级到之前使用的版本，然后将您的世界备份复制回 `saves` 文件夹。
:::

## 安装整合包

Minecraft 启动器不提供自动安装模组集合及相关配置文件（即所谓的整合包）的功能。这通常属于[第三方启动器][launchers]的范畴。

:::tip
如果您正在自己构建整合包，我们建议您查看[整合包文档][modpack]以获取一些技巧和建议。
:::

[curseforge]: https://www.curseforge.com/minecraft/search?class=mc-mods
[dotminecraft]: https://minecraft.wiki/w/.minecraft
[java]: index.md#java
[launchers]: launchers.md
[modpack]: /modpack/docs
[modrinth]: https://modrinth.com/mods
[neoforged]: https://neoforged.net