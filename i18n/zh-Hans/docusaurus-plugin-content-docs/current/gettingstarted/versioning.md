# 版本控制

本文将分解 Minecraft 和 NeoForge 中的版本控制如何工作，并提供一些模组版本控制的建议。

## Minecraft

Minecraft 使用[语义化版本控制][semver]。语义化版本控制，简称 "semver"，格式为 `major.minor.patch`。例如，Minecraft 1.20.2 的主版本为 1，次版本为 20，补丁版本为 2。

自 2011 年引入 Minecraft 1.0 以来，Minecraft 一直使用 `1` 作为主版本。在此之前，版本控制方案经常更改，并且有像 `a1.1` (Alpha 1.1)、`b1.7.3` (Beta 1.7.3) 甚至 `infdev` 版本这样的版本，这些版本根本没有遵循清晰的版本控制方案。由于 `1` 主版本已经持续了十多年，并且由于 Minecraft 2 的内部玩笑，人们普遍认为这不太可能改变。

### 快照

快照偏离了标准的 semver 方案。它们被标记为 `YYwWWa`，其中 `YY` 代表年份的最后两位数字（例如 `23`），`WW` 代表该年的第几周（例如 `01`）。例如，快照 `23w01a` 是 2023 年第一周发布的快照。

后缀 `a` 用于在同一周发布两个快照的情况（第二个快照将被命名为类似 `23w01b` 的名称）。Mojang 过去偶尔使用过这个。替代后缀也曾用于像 `20w14infinite` 这样的快照，这是[2020 年无限维度愚人节玩笑][infinite]。

### 预发布版和发布候选版

当快照周期即将完成时，Mojang 开始发布所谓的预发布版。预发布版被认为是一个版本的功能完整，并且仅专注于错误修复。它们使用其所针对版本的 semver 表示法，后缀为 `-preX`。例如，1.20.2 的第一个预发布版被命名为 `1.20.2-pre1`。可以有并且通常有多个预发布版，相应地后缀为 `-pre2`、`-pre3` 等。

类似地，当预发布周期完成时，Mojang 会发布发布候选版 1（版本后缀为 `-rc1`，例如 `1.20.2-rc1`）。Mojang 的目标是有一个发布候选版，如果没有进一步的错误发生，他们可以发布它。但是，如果出现意外的错误，那么也可能会有 `-rc2`、`-rc3` 等版本，类似于预发布版。

## NeoForge

NeoForge 使用了一个改编的 semver 系统：主版本是 Minecraft 的次版本，次版本是 Minecraft 的补丁版本，补丁版本是“实际的” NeoForge 版本。例如，NeoForge 20.2.59 是 Minecraft 1.20.2 的第 60 个版本（我们从 0 开始）。开头的 `1` 被省略了，因为它不太可能改变，原因见[上文][minecraft]。

NeoForge 中的一些地方也使用 [Maven 版本范围][mvr]，例如 [`neoforge.mods.toml`][neoforgemodstoml] 文件中的 Minecraft 和 NeoForge 版本范围。这些大多与 semver 兼容，但并非完全兼容（例如，它不考虑 `pre` 标签）。

## 模组

没有绝对最好的版本控制系统。不同的开发风格、项目范围等都影响选择哪种版本控制系统。有时，版本控制系统也可以组合使用。本节试图概述一些常用的版本控制系统，并提供真实示例。

通常，模组的文件名看起来像 `modid-<version>.jar`。所以如果我们的模组 ID 是 `examplemod`，我们的版本是 `1.2.3`，我们的模组文件将被命名为 `examplemod-1.2.3.jar`。

:::note
版本控制系统是建议，而不是严格强制执行的规则。这对于版本何时更改（“提升”）以及以何种方式更改尤其如此。如果你想使用不同的版本控制系统，没有人会阻止你。
:::

### 语义化版本控制

语义化版本控制（"semver"）由三部分组成：`major.minor.patch`。当对代码库进行重大更改时，主版本会提升，这通常与主要新功能和错误修复相关。次版本在引入次要功能时提升，补丁版本在更新仅包含错误修复时提升。

人们普遍认为，任何 `0.x.x` 版本都是开发版本，在第一次（完整）发布时，版本应提升到 `1.0.0`。

“次版本用于功能，补丁版本用于错误修复”的规则在实践中经常被忽视。一个流行的例子是 Minecraft 本身，它通过次版本号进行主要功能更新，通过补丁号进行次要功能更新，并通过快照进行错误修复（见上文）。

根据模组更新的频率，这些数字可以更小或更大。例如，[Supplementaries][supplementaries] 在撰写本文时版本为 `2.6.31`。三位数甚至四位数，尤其是在 `patch` 中，绝对是可能的。

### “简化”和“扩展” Semver

有时，semver 可能只看到两个数字。这是一种“简化”的 semver，或“两部分” semver。它们的版本号只有 `major.minor` 方案。这通常用于只添加几个简单对象的小型模组，因此很少需要更新（除了 Minecraft 版本更新），通常永远保持在版本 `1.0`。

“扩展” semver 或“四部分” semver 有四个数字（所以类似于 `1.0.0.0`）。根据模组的不同，格式可以是 `major.api.minor.patch`，或 `major.minor.patch.hotfix`，或者其他完全不同的格式——没有标准的方法。

对于 `major.api.minor.patch`，`major` 版本与 `api` 版本解耦。这意味着 `major`（功能）位和 `api` 位可以独立提升。这通常用于向其他模组开发者公开 API 的模组。例如，[Mekanism][mekanism] 在撰写本文时版本为 10.4.5.19。

对于 `major.minor.patch.hotfix`，补丁级别被分成两部分。这是 [Create][create] 模组使用的方法，在撰写本文时版本为 0.5.1f。请注意，Create 将热修复表示为字母而不是第四个数字，以保持与常规 semver 的兼容性。

:::info
简化 semver、扩展 semver、两部分 semver 和四部分 semver 不是官方术语或标准化格式。
:::

### Alpha、Beta、Release

像 Minecraft 本身一样，模组开发通常在软件工程中经典的 `alpha`/`beta`/`release` 阶段进行，其中 `alpha` 表示不稳定/实验性版本（有时也称为 `experimental` 或 `snapshot`），`beta` 表示半稳定版本，`release` 表示稳定版本（有时称为 `stable` 而不是 `release`）。

一些模组使用它们的主版本来表示 Minecraft 版本提升。一个例子是 [JEI][jei]，它对 Minecraft 1.19.2 使用 `13.x.x.x`，对 1.19.4 使用 `14.x.x.x`，对 1.20.1 使用 `15.x.x.x`（没有 1.19.3 和 1.20.0 的版本）。其他模组将标签附加到模组名称，例如 [Minecolonies][minecolonies] 模组，在撰写本文时版本为 `1.1.328-BETA`。

### 包含 Minecraft 版本

在文件名中包含模组所针对的 Minecraft 版本是很常见的。这使得最终用户可以更容易地找出模组适用于哪个 Minecraft 版本。一个常见的位置是在模组版本之前或之后，前者比后者更广泛。例如，适用于 1.20.2 的 JEI 版本 `16.0.0.28`（撰写本文时的最新版本）将变为 `jei-1.20.2-16.0.0.28` 或 `jei-16.0.0.28-1.20.2`。

### 包含模组加载器

你可能知道，NeoForge 并不是唯一的模组加载器，许多模组开发者在多个平台上进行开发。因此，需要一种方法来区分同一模组同一版本但适用于不同模组加载器的两个文件。

通常，这是通过在名称中包含模组加载器来完成的。`jei-neoforge-1.20.2-16.0.0.28`、`jei-1.20.2-neoforge-16.0.0.28` 或 `jei-1.20.2-16.0.0.28-neoforge` 都是有效的方法。对于其他模组加载器，`neoforge` 部分将被替换为 `forge`、`fabric`、`quilt` 或你与 NeoForge 一起开发的其他不同模组加载器。

### 关于 Maven 的说明

Maven 是用于依赖项托管的系统，它使用的版本控制系统在细节上与 semver 有所不同（尽管一般的 `major.minor.patch` 模式保持不变）。相关的 [Maven 版本范围 (MVR)][mvr] 系统在 NeoForge 的一些地方使用（见[上文][neoforge]）。选择版本控制方案时，应确保它与 MVR 兼容，否则模组将无法依赖你模组的特定版本！

[create]: https://www.curseforge.com/minecraft/mc-mods/create
[infinite]: https://minecraft.wiki/w/Java_Edition_20w14∞
[jei]: https://www.curseforge.com/minecraft/mc-mods/jei
[mekanism]: https://www.curseforge.com/minecraft/mc-mods/mekanism
[minecolonies]: https://www.curseforge.com/minecraft/mc-mods/minecolonies
[minecraft]: #minecraft
[neoforgemodstoml]: modfiles.md#neoforgemodstoml
[mvr]: https://maven.apache.org/enforcer/enforcer-rules/versionRanges.html
[mvr]: https://maven.apache.org/ref/3.9.9/maven-artifact/apidocs/org/apache/maven/artifact/versioning/ComparableVersion.html
[neoforge]: #neoforge
[pre]: #pre-releases
[rc]: #release-candidates
[semver]: https://semver.org/
[supplementaries]: https://www.curseforge.com/minecraft/mc-mods/supplementaries