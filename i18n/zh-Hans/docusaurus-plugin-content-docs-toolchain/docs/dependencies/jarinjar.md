# Jar-in-Jar(Jar-in-Jar)

Jar-in-Jar 是一种从模组 jar 文件内部加载依赖项的方式。为此，Jar-in-Jar 在构建时会在 `META-INF/jarjar/metadata.json` 中生成一个元数据 json 文件，其中包含要从 jar 内部加载的构件。

Jar-in-Jar 是一个完全可选的系统。这会将 `jarJar` 配置中的所有依赖项包含到 `jarJar` 任务中。

## 添加依赖项(Adding Dependencies)

您可以使用 `jarJar` 配置添加要包含在您的 jar 文件内部的依赖项。Jar-in-Jar 是一个协商系统，默认情况下，将根据 `prefer` 版本创建并包含最高版本。

```gradle
// 在 build.gradle 中
dependencies {
    // 编译并包含从 2.0（含）开始支持的 examplelib 版本
    jarJar(implementation(group: 'com.example', name: 'examplelib')) {
        version {
            // 在您的工作区中使用并捆绑到模组 jar 中的版本
            prefer '2.0'
        }
    }
}
```

如果您的库只能在确切的版本范围内工作，而不是从首选版本到最高版本，您可以配置 `strictly` 以指定您的模组兼容的范围：

```gradle
// 在 build.gradle 中
dependencies {
    // 编译并包含 examplelib 在 2.0（含）到 3.0（不含）之间的最高支持版本
    jarJar(implementation(group: 'com.example', name: 'examplelib') {
        version {
            // 设置 examplelib 的支持版本范围为 2.0（含）到 3.0（不含）
            strictly '[2.0,3.0)'
            // 在您的工作区中使用并捆绑到模组 jar 中的版本
            prefer '2.8.0'
        }
    }
}
```