# NeoForge 更新检查器(NeoForge Update Checker)

NeoForge 提供了一个非常轻量级、可选择加入的更新检查框架。如果任何模组有可用更新，它将在主菜单和模组列表的“模组”(Mods)按钮上显示闪烁图标以及相应的更新日志。它*不会*自动下载更新。

## 开始使用

您要做的第一件事是在 `mods.toml` 文件中指定 `updateJSONURL` 参数。此参数的值应是指向更新 JSON 文件的有效 URL。此文件可以托管在您自己的网络服务器、GitHub 或任何您希望的地方，只要您模组的所有用户都能可靠地访问它。

## 更新 JSON 格式

JSON 本身具有相对简单的格式，如下所示：

```json5
{
    "homepage": "<您的模组的主页/下载页面>",
    "<mcversion>": {
        "<modversion>": "<此版本的更新日志>",
        // 列出给定《我的世界》(Minecraft)版本下您的模组的所有版本及其更新日志
        // ...
    },
    "promos": {
        "<mcversion>-latest": "<modversion>",
        // 声明给定《我的世界》(Minecraft)版本下您的模组的最新“前沿”版本
        "<mcversion>-recommended": "<modversion>",
        // 声明给定《我的世界》(Minecraft)版本下您的模组的最新“稳定”版本
        // ...
    }
}
```

这相当不言自明，但有一些注意事项：

- `homepage` 下的链接是模组过时将向用户显示的链接。
- NeoForge 使用内部算法来确定您模组的一个版本字符串是否比另一个“更新”。大多数版本控制方案应该兼容，但如果您担心您的方案是否受支持，请参阅 `ComparableVersion` 类。强烈建议遵循 [Maven 版本控制(Maven versioning)][mvnver]。
- 更新日志字符串可以使用 `\n` 分隔成行。有些人喜欢包含简短的更新日志，然后链接到提供完整更改列表的外部站点。
- 手动输入数据可能很繁琐。您可以配置您的 `build.gradle` 在构建版本时自动更新此文件，因为 Groovy 具有本机 JSON 解析支持。如何做到这一点留给读者作为练习。
- 可以在 [nocubes]、[Corail Tombstone][corail] 和 [Chisels & Bits 2][chisel] 找到一些示例。

## 检索更新检查结果(Retrieving Update Check Results)

您可以使用 `VersionChecker.getResult(IModInfo)` 检索 NeoForge 更新检查器的结果。您可以通过 `ModContainer.getModInfo` 获取您的 `IModInfo`，其中 `ModContainer` 可以作为参数添加到您的模组构造函数中。您可以使用 `ModList.get().getModContainerById(<modId>)` 获取任何其他模组的 `ModContainer`。返回的对象有一个 `status` 方法，指示版本检查的状态。

| 状态(Status)       | 描述                                       |
| ------------------ | :----------------------------------------- |
| `FAILED`           | 版本检查器无法连接到提供的 URL。           |
| `UP_TO_DATE`       | 当前版本等于推荐版本。                     |
| `AHEAD`            | 如果没有最新版本，当前版本比推荐版本新。   |
| `OUTDATED`         | 有新的推荐或最新版本。                     |
| `BETA_OUTDATED`    | 有新的最新版本。                           |
| `BETA`             | 当前版本等于或新于最新版本。               |
| `PENDING`          | 请求的结果尚未完成，因此您应该稍后再试。   |

返回的对象还将包含目标版本和 `update.json` 中指定的任何更新日志行。

[mvnver]: ../gettingstarted/versioning.md
[nocubes]: https://cadiboo.github.io/projects/nocubes/update.json
[corail]: https://github.com/Corail31/tombstone_lite/blob/master/update.json
[chisel]: https://curseupdate.com/231095/chiselsandbits?ml=neoforge