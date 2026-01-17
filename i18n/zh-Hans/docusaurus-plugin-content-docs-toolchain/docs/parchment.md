import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Parchment(Parchment)

[Parchment(Parchment)] 是一套由社区提供、用于补充 Mojang 发布的官方名称的**参数名**和**Java文档**映射集。通过使用 Parchment，您可以获得大多数 Minecraft 方法的参数名，而不是那些无描述性的 `p_` 名称。

## 配置 Parchment

最基本的配置是在 `gradle.properties` 文件中使用以下属性：

<Tabs defaultValue="mdg">
<TabItem value="mdg" label="ModDevGradle">

```properties
# Parchment 版本所对应的 Minecraft 版本
neoForge.parchment.minecraftVersion=1.20.2
# Parchment 映射的版本
neoForge.parchment.mappingsVersion=2023.12.10
```

</TabItem>
<TabItem value="ng" label="NeoGradle">

```properties
# Parchment 版本所对应的 Minecraft 版本
neogradle.subsystems.parchment.minecraftVersion=1.20.2
# Parchment 映射的版本
neogradle.subsystems.parchment.mappingsVersion=2023.12.10
```

</TabItem>
</Tabs>

该子系统也支持 Gradle DSL 并提供了更多参数，以下 Gradle 代码片段对此进行说明：

<Tabs defaultValue="mdg">
<TabItem value="mdg" label="ModDevGradle">

```gradle
neoForge {
    parchment {
        // 创建 Parchment 映射时所基于的 Minecraft 版本。
        // 这不一定需要与您的模组所针对的 Minecraft 版本相匹配
        // 默认为 Gradle 属性 neoForge.parchment.minecraftVersion 的值
        minecraftVersion = "1.20.2"
        
        // 要应用的 Parchment 映射版本。
        // 列表请参见 https://parchmentmc.org/docs/getting-started。
        // 默认为 Gradle 属性 neoForge.parchment.mappingsVersion 的值
        mappingsVersion = "2023.12.10"
        
        // 覆盖要使用的 Parchment 构件的完整 Maven 坐标
        // 默认情况下，这会根据 minecraftVersion 和 mappingsVersion 属性计算得出。
        // 如果您显式设置此属性，则 minecraftVersion 和 mappingsVersion 将被忽略。
        // parchmentArtifact = "org.parchmentmc.data:parchment-$minecraftVersion:$mappingsVersion:checked@zip"
        
        // 当参数与方法内部其他名称冲突时，为参数添加的前缀字符串。
        // 默认为 `p_`。您可以将此属性设置为空字符串以禁用冲突解决，
        // 例如，当您使用已包含前缀的 checked 版本的 parchment 时。
        // conflictResolutionPrefix = ''

        // 可用于显式禁用此子系统。默认情况下，一旦设置了 parchmentArtifact 或 minecraftVersion 和 mappingsVersion，它将自动启用。
        // enabled = true
    }
}
```

</TabItem>
<TabItem value="ng" label="NeoGradle">

```gradle
subsystems {
    parchment {
        // 创建 Parchment 映射时所基于的 Minecraft 版本。
        // 这不一定需要与您的模组所针对的 Minecraft 版本相匹配
        // 默认为 Gradle 属性 neogradle.subsystems.parchment.minecraftVersion 的值
        minecraftVersion = "1.20.2"
        
        // 要应用的 Parchment 映射版本。
        // 列表请参见 https://parchmentmc.org/docs/getting-started。
        // 默认为 Gradle 属性 neogradle.subsystems.parchment.mappingsVersion 的值
        mappingsVersion = "2023.12.10"
        
        // 覆盖要使用的 Parchment 构件的完整 Maven 坐标
        // 默认情况下，这会根据 minecraftVersion 和 mappingsVersion 属性计算得出。
        // 如果您显式设置此属性，则 minecraftVersion 和 mappingsVersion 将被忽略。
        // 内置默认值也可以通过 Gradle 属性 neogradle.subsystems.parchment.parchmentArtifact 覆盖
        // parchmentArtifact = "org.parchmentmc.data:parchment-$minecraftVersion:$mappingsVersion:checked@zip"
        
        // 覆盖用于应用 Parchment 映射的工具的完整 Maven 坐标
        // 参见 https://github.com/neoforged/JavaSourceTransformer
        // 内置默认值也可以通过 Gradle 属性 neogradle.subsystems.parchment.toolArtifact 覆盖
        // toolArtifact = "net.neoforged.jst:jst-cli-bundle:1.0.30"
        
        // 当参数与方法内部其他名称冲突时，为参数添加的前缀字符串。
        // 默认为 `p_`。您可以将此属性设置为空字符串以禁用冲突解决，
        // 例如，当您使用已包含前缀的 checked 版本的 parchment 时。
        // conflictPrefix = ''

        // 如果您不希望在启用应用 Parchment 映射时自动添加 https://maven.parchmentmc.org/ 仓库，请将此设置为 false
        // 内置默认值也可以通过 Gradle 属性 neogradle.subsystems.parchment.addRepository 覆盖
        // addRepository = true
        
        // 可用于显式禁用此子系统。默认情况下，一旦设置了 parchmentArtifact 或 minecraftVersion 和 mappingsVersion，它将自动启用。
        // 内置默认值也可以通过 Gradle 属性 neogradle.subsystems.parchment.enabled 覆盖
        // enabled = true
    }
}
```

</TabItem>
</Tabs>

:::tip
您可以在 Parchment 的[文档](https://parchmentmc.org/docs/getting-started)中找到最新的版本。
:::

[Parchment(Parchment)]: https://parchmentmc.org/