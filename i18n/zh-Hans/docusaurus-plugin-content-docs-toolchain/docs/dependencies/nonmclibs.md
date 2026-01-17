---
sidebar_position: 1
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# 非 Minecraft 依赖项(Non-Minecraft Dependencies)

非 Minecraft 依赖项是指既不是模组，也不是 Minecraft 或 NeoForge 本身所依赖的依赖项的构件。默认情况下，NeoForge 在加载模组时不会加载非 Minecraft 依赖项。对于开发环境，必须将它们作为运行时依赖项添加，而生产环境则应使用 [jar-in-jar 系统][jij]。

## 1.21.9 及以上版本

在 1.21.9 及以上版本上运行 NeoForge 时，将加载开发环境中类路径上可用的任何内容，包括非 Minecraft 依赖项。这意味着添加库就像添加任何其他 Gradle 依赖项一样简单：

```gradle
// 这会在编译时和运行时添加该库
// 在实践中，这应该用 'jarJar' 包装，以便将库包含在您的 jar 中
implementation 'com.example:example:1.0'
```

## 1.21.8 及以下版本

在 1.21.8 及以下版本上运行 NeoForge 时，还需要将库添加到运行时类路径：

<Tabs defaultValue="mdg">
<TabItem value="mdg" label="ModDevGradle">

```gradle
dependencies {
    // 这是为了在编译时添加该库
    implementation 'com.example:example:1.0'
    // 这会将库添加到所有运行配置中
    additionalRuntimeClasspath 'com.example:example:1.0'
}
```

</TabItem>
<TabItem value="ng" label="NeoGradle">

```gradle
dependencies {
    implementation 'com.example:example:1.0'
}

runs {
    configureEach {
        dependencies {
            // highlight-next-line
            runtime 'com.example:example:1.0'
        }
    }
}
```

或者，您可以使用一个配置(configuration)：

```gradle
configurations {
    libraries
    // 这将确保您添加到 libraries 配置的所有依赖项也会添加到 implementation 配置
    // 这样，您只需一次依赖声明即可同时用于运行时和编译时依赖项
    implementation.extendsFrom libraries
}

dependencies {
    libraries 'com.example:example:1.0'
}

runs {
    configureEach {
        dependencies {
            // highlight-next-line
            runtime project.configurations.libraries
        }
    }
}
```

</TabItem>
</Tabs>

[jij]: jarinjar.md