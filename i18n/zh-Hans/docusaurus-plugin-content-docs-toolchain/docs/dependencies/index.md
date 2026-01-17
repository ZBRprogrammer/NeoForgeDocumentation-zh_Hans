# 依赖项(Dependencies)

依赖项不仅用于开发模组间的互操作性，或为游戏添加额外的库，它还决定了为哪个版本的 Minecraft 进行开发。本文将简要概述如何修改 `repositories` 和 `dependencies` 代码块，以便向您的开发环境添加依赖项。

:::note

本文不会深入解释 Gradle 概念。强烈建议在继续之前阅读 [Gradle 依赖管理指南][guide]。

:::

## 模组依赖项(Mod Dependencies)

所有模组依赖项的添加方式与其他构件相同。

```gradle
dependencies {
    // 假设我们有一个名为 'examplemod' 的构件，可以从指定仓库获取
    implementation 'com.example:examplemod:1.0'
}
```

### 本地模组依赖项(Local Mod Dependencies)

如果您尝试依赖的模组在 Maven 仓库（例如 [Maven Central][central]、[CurseMaven]、[Modrinth]）中不可用，您可以使用[平面目录][flat]来添加模组依赖项：

```gradle
repositories {
    // 将项目目录中的 'libs' 文件夹添加为平面目录
    flatDir {
        dir 'libs'
    }
}

dependencies {
    // ...

    // 给定 <group>:<name>:<version>:<classifier (默认无)>
    //   以及扩展名 <ext (默认 jar)>
    // 平面目录中的构件将按以下顺序解析：
    // - <name>-<version>.<ext>
    // - <name>-<version>-<classifier>.<ext>
    // - <name>.<ext>
    // - <name>-<classifier>.<ext>

    // 如果显式指定了分类器(classifier)
    //  则具有该分类器的构件将优先：
    // - examplemod-1.0-api.jar
    // - examplemod-api.jar
    // - examplemod-1.0.jar
    // - examplemod.jar
    implementation 'com.example:examplemod:1.0:api'
}
```

:::note
对于平面目录条目，组名可以是任意非空字符串，因为在解析构件文件时不会检查组名。
:::

[guide]: https://docs.gradle.org/8.14.3/userguide/dependency_management.html

[central]: https://central.sonatype.com/
[CurseMaven]: https://cursemaven.com/
[Modrinth]: https://docs.modrinth.com/docs/tutorials/maven/

[flat]: https://docs.gradle.org/8.14.3/userguide/supported_repository_types.html#sec:flat-dir-resolver