# NeoGradle(NG)

[![发布](https://github.com/neoforged/NeoGradle/actions/workflows/release.yml/badge.svg?branch=NG_7.0)](https://github.com/neoforged/NeoGradle/actions/workflows/release.yml)

* * *

Minecraft 模组开发框架，被 NeoForge(NeoForge) 和 FML(FML) 用于 Gradle 构建系统。

快速入门，请参阅 [NeoForge 模组开发工具包](https://github.com/neoforged/MDK) 如何使用 NeoGradle。

要查看 NeoGradle 的最新可用版本，请访问 [NeoForged 项目页面](https://projects.neoforged.net/neoforged/neogradle)。

## 插件[​](#plugins "Direct link to Plugins")

NeoGradle 分为几个不同的插件，这些插件可以彼此独立应用。

### 用户开发插件(Userdev Plugin)[​](#userdev-plugin "Direct link to Userdev Plugin")

此插件用于构建 NeoForge 模组。作为模组开发者，在许多情况下，这将是您使用的唯一插件。

```
plugins {
  id 'net.neoforged.gradle.userdev' version '<neogradle_version>'
}

dependencies {
  implementation 'net.neoforged:neoforge:<neoforge_version>'
}
```

#### 访问转换器(Access Transformers)[​](#-access-transformers "Direct link to -access-transformers")

用户开发插件提供了一种为您的模组配置访问转换器的方式。您需要在资源目录中创建一个访问转换器配置文件，然后配置用户开发插件以使用它。

```
userdev {
    accessTransformer {
        file 'src/main/resources/META-INF/accesstransformer.cfg'
    }
}
```

这里的路径由您决定，并且不需要包含在最终的 jar 中。

#### 接口注入(Interface Injections)[​](#-interface-injections "Direct link to -interface-injections")

用户开发插件提供了一种为您的模组配置接口注入的方式。这允许您拥有一个反编译的 Minecraft 构件，其中已静态应用了您想要通过 Mixins 注入的接口。这种方法的优点是您可以在代码中使用接口，而 Mixins 将应用于接口，而不是实现它们的类。

```
userdev {
    interfaceInjection {
        file 'src/main/resources/META-INF/interfaceinjection.json'
    }
}
```

有关文件格式的更多信息，请参见[此处](https://github.com/neoforged/JavaSourceTransformer?tab=readme-ov-file#interface-injection)。

#### 用户开发插件的依赖管理[​](#dependency-management-by-the-userdev-plugin "Direct link to Dependency management by the userdev plugin")

当此插件检测到对 NeoForge 的依赖时，它将开始工作并创建必要的 NeoForm(NeoForm) 运行时任务，以构建包含所请求 NeoForge 版本的可用的 Minecraft JAR 文件。此外（如果通过约定配置这样做，这是默认设置），它将为您的项目创建运行，并将必要的依赖项添加到运行的类路径中。

##### 版本目录[​](#version-catalogues "Direct link to Version Catalogues")

使用 Gradle 的现代版本目录功能意味着依赖项在最后一刻添加，无论是在脚本评估期间还是之后。这意味着，如果您使用此功能，有时可能无法完全按照您希望的方式配置默认运行。在这种情况下，禁用运行的约定并手动配置它们可能是有益的。

参见以下示例：

_lib.versions.toml:_

```
[versions]
# Neoforge 设置
neoforge = "+"

[libraries]
neoforge = { group = "net.neoforged", name = "neoforge", version.ref = "neoforge" }
```

_build.gradle:_

```
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}
            
dependencies {
    implementation(libs.neoforge)
}

runs {
    configureEach { run ->
        run.modSource sourceSets.main
    }
    
    client { }
    server { }
    datagen { }
    gameTestServer { }
    junit { }
}
```

您不需要创建所有五种不同的运行，只需要您需要的那些。这是因为在 Gradle 实际添加依赖项时，我们无法再创建任何运行，但我们仍然可以在添加依赖项时为您配置它们。

### 通用插件(Common Plugin)[​](#common-plugin "Direct link to Common Plugin")

#### 可用的依赖管理[​](#-available-dependency-management "Direct link to -available-dependency-management")

##### 额外 Jar[​](#-extra-jar "Direct link to -extra-jar")

通用插件为 Minecraft 的 extra-jar 提供依赖管理。这是一个特殊的 jar，仅包含 Minecraft 的资源和资产，不包含代码。这对于依赖于 Minecraft 资源但不依赖于其代码的模组很有用。

```
plugins {
  id 'net.neoforged.gradle.common' version '<neogradle_version>'
}

dependencies {
  implementation "net.minecraft:client:<minecraft_version>:client-extra"
}
```

#### Jar-in-Jar 支持[​](#jar-in-jar-support "Direct link to Jar-in-Jar support")

通用插件为 jar-in-jar 依赖项提供支持，包括发布和使用时。

##### 使用 Jar-in-Jar 依赖项[​](#consuming-jar-in-jar-dependencies "Direct link to Consuming Jar-in-Jar dependencies")

如果您想依赖于使用 jar-in-jar 依赖项的项目，可以在 build.gradle 中添加以下内容：

```
dependencies {
    implementation "project.with:jar-in-jar:1.2.3"
}
```

显然，您需要将 `project.with:jar-in-jar:1.2.3` 替换为您想依赖的项目的实际坐标，并且可以使用任何配置来确定依赖项的范围。

##### 发布[​](#-publishing "Direct link to -publishing")

如果您想发布一个 jar-in-jar 依赖项，可以在 build.gradle 中添加以下内容：

```
dependencies {
    jarJar("project.you.want.to:include-as-jar-in-jar:[my.lowe.supported.version,potentially.my.upper.supported.version)") {
        version {
            prefer "the.version.you.want.to.use"
        }
    }
}
```

这里需要注意的是，需要指定版本范围。Jar-in-Jar 依赖项不支持直接指定单个版本。如果需要指定单个版本，可以通过指定相同的版本作为版本范围的下限和上限来实现：`[the.version.you.want.to.use]`。

###### Jar-in-Jar 依赖项移动的处理[​](#-handling-of-moved-jar-in-jar-dependencies "Direct link to -handling-of-moved jar-in-jar dependencies")

当依赖项从一个 GAV 移动到另一个时，通常会通过 maven-metadata.xml、单独的 pom 文件或通过 Gradle 的 available-at 元数据发布一个转移坐标。这可能导致依赖项的版本与您指定的版本不同。最好将您的依赖项更新到新的 GAV，以防止将来出现问题和混淆。

#### 管理运行[​](#managing-runs "Direct link to Managing runs")

通用插件提供了一种管理项目中运行的方法。其主要目的是确保无论您使用原版、neoform、平台还是用户开发模块，您始终可以以相同的方式管理运行。

```
plugins {
  id 'net.neoforged.gradle.common' version '<neogradle_version>'
}

runs {
    //...运行配置
}
```

##### 配置运行[​](#-configuring-runs "Direct link to -configuring-runs")

当您在项目中创建一个运行时，它最初是空的。如果您不使用运行类型，或从另一个运行克隆配置（如下所述），您将需要自己配置运行。

```
runs {
    someRun {
        isIsSingleInstance true //这将使运行成为单实例运行，意味着一次只能运行一个该运行的实例
        mainClass 'com.example.Main' //这将设置运行的主类
        arguments 'arg1', 'arg2' //这将设置运行的参数
        jvmArguments '-Xmx4G' //这将设置运行的 JVM 参数
        isClient true //这将设置运行为客户端运行
        isServer true //这将设置运行为服务端运行
        isDataGenerator true //这将设置运行为数据生成运行
        isGameTest true //这将设置运行为游戏测试运行
        isJUnit true //这将设置运行为 junit 运行，表示应使用单元测试环境而不是普通运行
        environmentVariables 'key1': 'value1', 'key2': 'value2' //这将设置运行的环境变量
        systemProperties 'key1': 'value1', 'key2': 'value2' //这将设置运行的系统属性
        classPath.from project.configurations.runtimeClasspath //这将向此运行的类路径添加一个元素
        
        shouldBuildAllProjects true //这将设置运行在运行前构建所有项目
        workingDirectory file('some/path') //这将设置运行的工作目录
    }
}
```

除了这些基本配置外，您始终可以查看 [Runs DSL 对象](https://github.com/neoforged/NeoGradle/blob/fec82f73860e8a3fdd21ac0701827ed890ff576e/dsl/common/src/main/groovy/net/neoforged/gradle/dsl/common/runs/run/Run.groovy) 以查看其他可用选项。

##### 配置运行类型[​](#-configuring-run-types "Direct link to -configuring-run-types")

如果您使用的运行类型不是来自 SDK（例如那些来自用户开发插件的），您可以按如下方式配置它们：

```
runs {
    someRun {
        isIsSingleInstance true //这将使此类型的运行成为单实例运行，意味着一次只能运行一个此类型运行的实例
        mainClass 'com.example.Main' //这将设置此类型运行的主类
        arguments 'arg1', 'arg2' //这将设置此类型运行的参数
        jvmArguments '-Xmx4G' //这将设置此类型运行的 JVM 参数
        isClient true //这将设置运行为客户端此类型运行
        isServer true //这将设置运行为服务端此类型运行
        isDataGenerator true //这将设置运行为数据生成此类型运行
        isGameTest true //这将设置运行为游戏测试此类型运行
        isJUnit true //这将设置运行为 junit 此类型运行，表示应使用单元测试环境而不是普通运行
        environmentVariables 'key1': 'value1', 'key2': 'value2' //这将设置此类型运行的环境变量
        systemProperties 'key1': 'value1', 'key2': 'value2' //这将设置此类型运行的系统属性
        classPath.from project.configurations.runtimeClasspath //这将向此类型运行的类路径添加一个元素
    }
}
```

##### 运行的类型[​](#-types-of-runs "Direct link to -types-of-runs")

通用插件管理运行的存在，但它本身不创建或配置运行。它只是确保在您创建运行任务和 IDE 运行时，根据运行本身的配置为您创建它们。

为了基于给定模板配置运行，存在运行类型。例如，如果您使用用户开发插件，那么它将从您依赖的 SDK 加载运行类型，并允许您基于这些类型配置运行。

###### 按名称配置运行[​](#-configuring-runs-by-name "Direct link to -configuring-runs-by-name")

首先，默认情况下，通用插件会将任何名称与运行类型相同的运行配置为该运行类型。您可以通过为运行设置以下内容来禁用此功能：

```
runs {
    someRun {
        configureFromTypeWithName false
    }
}
```

###### 使用类型配置运行[​](#-configuring-run-using-types "Direct link to -configuring-run-using-types")

如果您想基于运行类型配置运行，可以通过设置以下内容来实现：

```
runs {
    someRun {
        runType 'client' //如果您想根据运行类型的名称配置运行类型
        configure runTypes.client //如果您想基于运行类型对象配置运行类型，这可能并不总是可行，因为 Gradle 的工作方式
        
        //以下方法已弃用，将在未来版本中移除
        configure 'client'
    }
}
```

如果在将运行实现为任务或 IDE 运行时找不到运行类型，这两者都会抛出异常。

##### 运行的配置[​](#-configuration-of-runs "Direct link to -configuration-of-runs")

添加运行时，通用插件还会检查它，并确定是否有任何 SDK 或 MDK 添加到其任何源集中，如果是，则给该 SDK/MDK 一个机会将其自己的设置添加到运行中。这允许像 NeoForge 这样的提供者为您预配置运行。

但是，如果您按如下方式设置运行：

```
runs {
    someRun {
        modSource sourceSets.some_version_a_sourceset
        modSource sourceSets.some_version_b_sourceset
    }
}
```

其中相关源集在其编译类路径中对给定的 SDK 版本有依赖，并且这些依赖不同，则会引发错误。这是因为通用插件无法确定运行应使用哪个版本的 SDK。要解决此问题，请选择其中一个版本并将其添加到运行中：

```
runs {
    someRun {
        modSource sourceSets.some_version_a_sourceset
    }
}
```

或

```
runs {
    someRun {
        modSource sourceSets.some_version_b_sourceset
    }
}
```

###### 使用另一个运行配置运行[​](#-configuring-run-using-another-run "Direct link to -configuring-run-using-another-run")

此外，您还可以将一个运行克隆到另一个运行中：

```
runs {
    someRun {
        run 'client' //如果您想将客户端运行克隆到 someRun 中
        configure runTypes.client //如果您想将客户端运行类型克隆到 someRun 中
    }
}
```

##### 在运行中使用 DevLogin[​](#using-devlogin-in-runs "Direct link to Using DevLogin in runs")

DevLogin 工具是一种允许您在开发期间无需使用 Minecraft 启动器即可登录 Minecraft 账户的工具。首次启动时，它将显示一个链接以登录您的 Minecraft 账户，然后您可以使用该工具登录您的账户。凭据缓存在您的本地机器上，然后用于未来的登录，因此仅当令牌过期时才需要重新登录。运行子系统使用此工具在所有客户端运行中启用已登录的玩法。该工具可以使用以下属性配置：

```
neogradle.subsystems.devLogin.enabled=<true/false>
```

默认情况下，子系统是启用的，它将为您准备好登录 Minecraft 账户的一切，但您仍然需要在您想要的运行上启用它。如果您想禁用它，可以将属性设置为 false，然后您将不会被要求登录，而是使用一个随机的未登录开发账户。

###### 每个运行的配置[​](#-per-run-configuration "Direct link to -per-run configuration")

如果您想为每个运行配置 dev login 工具，可以在运行配置中设置以下属性：

```
runs {
    someRun {
        devLogin {
            enabled true
        }
    }
}
```

这将为此运行启用 dev login 工具。默认情况下，dev login 工具是禁用的，并且只能为客户端运行启用。

警告

如果您为非客户端运行启用 dev login 工具，您将收到错误消息。

此外，通过设置运行配置中的以下属性，可以为 dev login 工具使用不同的用户配置文件：

```
runs {
    someRun {
        devLogin {
            profile '<profile>'
        }
    }
}
```

如果未设置，则将使用默认配置文件。有关配置文件的更多信息，请参阅 DevLogin 文档：[Covers1624 的 DevLogin](https://github.com/covers1624/DevLogin/blob/main/README.md#multiple-accounts)

###### 配置[​](#configurations "Direct link to Configurations")

为了将 dev login 工具添加到您的运行中，我们创建一个自定义配置，并向其添加 dev login 工具。此配置是为您注册为模组源集的第一个源集创建的。配置的后缀可以根据您的个人偏好进行配置，通过在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.devLogin.configurationSuffix=<suffix>
```

默认情况下，后缀设置为 "DevLoginLocalOnly"。我们采用这种方法创建一个仅运行时的配置，不会泄漏到项目的其他使用者。

##### 在运行中使用 RenderDoc[​](#using-renderdoc-in-runs "Direct link to Using RenderDoc in runs")

RenderDoc 工具是一种允许您从游戏中捕获帧并在其帧调试器中检查它们的工具。我们的连接器实现可以选择性地注入以启动带有您游戏的 RenderDoc。

###### 每个运行的配置[​](#-per-run-configuration-1 "Direct link to -per-run-configuration-1")

如果您想为每个运行配置 render doc 工具，可以在运行配置中设置以下属性：

```
runs {
    someRun {
        renderDoc {
            enabled true
        }
    }
}
```

这将为此运行启用 render doc 工具。默认情况下，render doc 工具是禁用的，并且只能为客户端运行启用。

警告

如果您为非客户端运行启用 render doc 工具，您将收到错误消息。

##### 资源处理[​](#resource-processing "Direct link to Resource processing")

Gradle 开箱即用地支持资源处理，使用 `processResources` 任务（或非主源集的等效任务）。然而，没有 IDE 开箱即用地支持此功能。为了为您提供最佳体验，NeoGradle 将在您运行游戏之前运行 `processResources` 任务。

如果您使用的是 IDEA 并启用了 "Build and Run using IntelliJ IDEA" 选项，那么 NeoGradle 还将修改其运行，以确保在游戏启动之前运行 `processResources` 任务，并将其输出重定向到构建目录中的单独位置，并且在您的运行中使用这些处理后的资源。

警告

这意味着 IDEA 编译过程创建的资源文件不会被使用，您不应依赖它们是最新的。

##### IDEA 兼容性[​](#idea-compatibility "Direct link to IDEA Compatibility")

由于 IDEA 从边栏启动单元测试的方式，如果使用 IDEA 测试引擎运行，则无法重新配置此类测试。为了支持此场景，默认情况下 NeoGradle 将重新配置 IDEA 的测试默认值，以支持在单元测试环境中运行这些测试。

然而，由于许多构造，无法为每个人正确配置默认值，如果您有一个非标准的测试设置，可以按如下方式配置默认值：

```
idea {
    unitTests {
        //普通的运行属性，以及像配置运行一样的源集配置：
        modSource sourceSets.anotherTestSourceSet
    }
}
```

如果您想禁用此功能，可以禁用相关约定，请参阅[禁用约定](#disabling-conventions)。

#### 源集管理[​](#-sourceset-management "Direct link to -sourceset-management")

通用插件提供了一种更轻松地管理项目中源集的方法。特别是，它允许您轻松依赖于不同的源集，或继承其依赖项。

警告

但是，重要的是要知道，您只能对同一项目中的源集执行此操作。如果您尝试依赖于另一个项目的源集，NeoGradle 将抛出异常。

##### 继承依赖项[​](#-inheriting-dependencies "Direct link to -inheriting-dependencies")

如果您想继承另一个源集的依赖项，可以在 build.gradle 中添加以下内容：

```
sourceSets {
    someSourceSet {
        inherit.from sourceSets.someOtherSourceSet
    }
}
```

##### 依赖于另一个源集[​](#-depending-on-another-sourceset "Direct link to -depending-on-another-sourceset")

如果您想依赖于另一个源集，可以在 build.gradle 中添加以下内容：

```
sourceSets {
    someSourceSet {
        depends.on sourceSets.someOtherSourceSet
    }
}
```

### NeoForm 运行时插件(NeoForm Runtime Plugin)[​](#neoform-runtime-plugin "Direct link to NeoForm Runtime Plugin")

此插件启用 NeoForm(NeoForm) 运行时的使用，并允许项目直接依赖于已反混淆但其他方面未修改的 Minecraft 构件。

此插件被其他插件内部使用，通常仅用于高级用例。

```
plugins {
  id 'net.neoforged.gradle.neoform' version '<neogradle_version>'
}

dependencies {
  // 依赖于包含客户端和服务端类的 Minecraft JAR 文件
  implementation "net.minecraft:neoform_joined:<neoform-version>"
  
  // 依赖于 Minecraft 客户端 JAR 文件
  implementation "net.minecraft:neoform_client:<neoform-version>"
  
  // 依赖于 Minecraft 专用服务器 JAR 文件
  implementation "net.minecraft:neoform_server:<neoform-version>"
}
```

## 应用 Parchment 映射[​](#apply-parchment-mappings "Direct link to Apply Parchment Mappings")

为了在反编译的 Minecraft 源代码中获得人类可读的参数名以及 Javadocs，可以在重新编译之前将来自 [Parchment 项目(Parchment project)](https://parchmentmc.org) 的众包数据应用到 Minecraft 源代码中。

目前仅支持在应用 NeoGradle 用户开发插件时使用。

最基本的配置是在 gradle.properties 中使用以下属性：

```
neogradle.subsystems.parchment.minecraftVersion=1.20.2
neogradle.subsystems.parchment.mappingsVersion=2023.12.10
```

该子系统还有一个 Gradle DSL 并支持更多参数，以下 Gradle 代码片段对此进行解释：

```
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
    
    // 如果您不希望在启用应用 Parchment 映射时自动添加 https://maven.parchmentmc.org/ 仓库，请将此设置为 false
    // 内置默认值也可以通过 Gradle 属性 neogradle.subsystems.parchment.addRepository 覆盖
    // addRepository = true
    
    // 可用于显式禁用此子系统。默认情况下，一旦设置了 parchmentArtifact 或 minecraftVersion 和 mappingsVersion，它将自动启用。
    // 内置默认值也可以通过 Gradle 属性 neogradle.subsystems.parchment.enabled 覆盖
    // enabled = true
  }
}
```

## 高级设置[​](#advanced-settings "Direct link to Advanced Settings")

### 覆盖反编译器设置[​](#-override-decompiler-settings "Direct link to -override-decompiler settings")

准备 Minecraft 依赖项时反编译器使用的设置可以使用 [Gradle 属性](https://docs.gradle.org/current/userguide/project_properties.html)覆盖。这对于在低端机器上运行 NeoGradle 很有用，但代价是构建时间更长。

| 属性 | 描述 |
| --- | --- |
| `neogradle.subsystems.decompiler.maxMemory` | 分配给反编译器的堆内存量。可以以千兆字节（`4g`）或兆字节（`4096m`）指定。 |
| `neogradle.subsystems.decompiler.maxThreads` | 默认情况下，反编译器使用所有可用的 CPU 核心。此设置可用于将其限制为给定数量的线程，线程数越少，消耗的内存越少。如果设置为 0，则禁用限制。 |
| `neogradle.subsystems.decompiler.logLevel` | 可用于覆盖[反编译器日志级别](https://vineflower.org/usage/#cmdoption-log)。 |

### 覆盖重新编译器设置[​](#override-recompiler-settings "Direct link to Override Recompiler Settings")

NeoGradle 用于重新编译反编译的 Minecraft 源代码的设置可以使用 [Gradle 属性](https://docs.gradle.org/current/userguide/project_properties.html)进行自定义。

| 属性 | 描述 |
| --- | --- |
| `neogradle.subsystems.recompiler.maxMemory` | 分配给反编译器的堆内存量。可以以千兆字节（`4g`）或兆字节（`4096m`）指定。默认为 `1g`。 |
| `neogradle.subsystems.recompiler.jvmArgs` | 将任意 JVM 参数传递给运行编译器的分叉 Gradle 进程。例如 `-XX:+HeapDumpOnOutOfMemoryError` |
| `neogradle.subsystems.recompiler.args` | 将额外的命令行参数传递给 Java 编译器。 |
| `neogradle.subsystems.recompiler.shouldFork` | 指示重新编译器是否应使用进程分叉。（默认为 true）。 |

## 运行特定的依赖管理[​](#-run-specific-dependency-management "Direct link to -run-specific-dependency-management")

警告

Minecraft 21.9 及更高版本不再需要运行特定的依赖管理。请移除您的 Gradle 代码。NeoGradle 不再使用它来配置 NeoForge 或 FML

这为运行的类路径实现了运行特定的依赖管理。过去，这必须通过手动修改 "minecraft_classpath" 令牌来实现，但是令牌不再作为可以在运行上配置的组件存在。因此，不可能将非 FML 感知的库添加到您的运行类路径中。

### 用法：[​](#usage "Direct link to Usage:")

#### 直接[​](#direct "Direct link to Direct")

```
dependencies {
    implementation 'some:library:1.2.3'
}

runs {
   testRun {
      dependencies {
         runtime 'some:library:1.2.3'
      }
   }
}
```

#### 配置[​](#configuration "Direct link to Configuration")

```
configurations {
   libraries {}
   implementation.extendsFrom libraries
}

dependencies {
    libraries 'some:library:1.2.3'
}

runs {
   testRun {
      dependencies {
         runtime project.configurations.libraries
      }
   }
}
```

#### 运行依赖处理器[​](#run-dependency-handler "Direct link to Run Dependency Handler")

运行上的依赖处理器与项目自己的依赖处理器非常相似，但它只有一个可用的“配置”来添加依赖项：“runtime”。此外，它提供了一个方法，用于您希望将整个配置转换为运行时依赖项时使用。

## 处理非 NeoGradle 兄弟项目[​](#handling-of-none-neogradle-sibling-projects "Direct link to Handling of None-NeoGradle sibling projects")

一般来说，我们建议，不强烈鼓励，**不要**为此解决方案使用胖 jar。创建包含所有兄弟项目代码的胖 jar 的过程很难以一种既正确又对开发项目高效的方式进行建模，特别是当兄弟项目不使用 NeoGradle 时。

### 兄弟项目使用 NeoGradle 模块[​](#sibling-project-uses-a-neogradle-module "Direct link to Sibling project uses a NeoGradle module")

如果可以使用 NeoGradle 模块（例如原版模块，而不是 VanillaGradle），那么您可以使用源集的模组标识符：

```
sourceSets {
    main {
        run {
            modIdentifier '<您的胖 jar 中所有项目共有的某个字符串>'
        }
    }
}
```

模组标识符的值在这里无关紧要，所有具有相同源集模组标识符的项目在运行您的运行时将被包含在同一个假的胖 jar 中。

### 兄弟项目不使用 NeoGradle[​](#sibling-project-does-not-use-neogradle "Direct link to Sibling project does not use NeoGradle")

如果兄弟项目不使用 NeoGradle，那么您必须确保其清单配置正确：

```
FMLModType: GAMELIBRARY #或任何其他不是模组的模组类型，如 LIBRARY
Automatic-Module-Name: '<此项目独有的某个字符串>'
```

警告

如果这样做，那么您的兄弟项目不允许包含同一个包中的类！这是因为不允许两个模块包含同一个包。如果您有两个兄弟项目，它们在同一个包中都有一个类，那么您需要移动其中一个！

### 在您的运行中包含兄弟项目[​](#including-the-sibling-project-in-your-run "Direct link to Including the sibling project in your run")

要将兄弟项目包含在您的运行中，您需要将其作为 modSource 添加到您的运行中：

```
runs {
    someRun {
        modSources {
            add project.sourceSets.main // 将拥有项目的主源集添加到基于该源集模组标识符的组中（这里可以是任何内容，取决于源集的扩展值或项目名称）
            add project(':api').sourceSets.main // 假设 API 项目不使用 NeoGradle，这将使用 `api` 键将 api 项目添加到一个组中，因为非 NeoGradle 项目的默认模组标识符是项目名称，这里是 api
            local project(':api').sourceSets.main // 假设 API 项目不使用 NeoGradle，这将使用拥有项目的名称将 api 项目添加到一个组中，而不是作为回退使用 api 项目的名称（这里可以是任何内容，取决于源集的扩展值或项目名称）
            add('something', project(':api').sourceSets.main) // 这将组标识符硬编码为 'something'，不对源集执行模组标识符查找，也不使用拥有项目或源集的项目。
        }
    }
}
```

不需要其他操作。

## 使用约定[​](#using-conventions "Direct link to Using conventions")

### 禁用约定[​](#disabling-conventions "Direct link to Disabling conventions")

默认情况下，约定是启用的。如果您想禁用约定，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.enabled=false
```

我们将认为约定是启用的，因此如果您想禁用它们，您必须显式地这样做。

### 配置[​](#configurations-1 "Direct link to Configurations")

NeoGradle 将向您的项目添加多个 `配置`。此约定可以通过在 gradle.properties 中设置以下属性来禁用：

```
neogradle.subsystems.conventions.configurations.enabled=false
```

每个源集添加以下配置，其中 XXX 是源集名称：

* XXXLocalRuntime
* XXXLocalRunRuntime

注意

为此，您的源集需要在依赖块之前定义。

每个运行添加以下配置：

* XXXRun

注意

为此，您的运行需要在依赖块之前定义。

全局添加以下配置：

* runs

#### LocalRuntime（每个源集）[​](#localruntime-per-sourceset "Direct link to LocalRuntime (Per SourceSet)")

此配置用于将依赖项仅添加到您的本地项目运行时，而不将它们暴露给其他项目的运行时。需要启用源集约定

#### LocalRunRuntime（每个源集）[​](#localrunruntime-per-sourceset "Direct link to LocalRunRuntime (Per SourceSet)")

此配置用于将依赖项添加到您添加源集的运行的本地运行时，而不将它们暴露给其他运行的运行时。需要启用源集约定

#### Run（每个运行）[​](#run-per-run "Direct link to Run (Per Run)")

此配置用于将依赖项仅添加到特定运行的运行时，而不将它们暴露给其他运行的运行时。

#### run（全局）[​](#run-global "Direct link to run (Global)")

此配置用于将依赖项添加到所有运行的运行时。

### 源集管理[​](#sourceset-management "Direct link to Sourceset Management")

要禁用源集管理，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.sourcesets.enabled=false
```

#### 当前项目自动包含在其运行中[​](#automatic-inclusion-of-the-current-project-in-its-runs "Direct link to Automatic inclusion of the current project in its runs")

默认情况下，当前项目的主源集自动包含在其运行中。如果您想禁用此功能，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.sourcesets.automatic-inclusion=false
```

如果您想禁用此功能，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.sourcesets.automatic-inclusion=false
```

这相当于在 build.gradle 中设置以下内容：

```
runs {
    configureEach { run ->
        run.modSource sourceSets.main
    }
}
```

##### 在运行配置中自动包含源集的本地运行运行时配置[​](#automatic-inclusion-of-a-sourcesets-local-run-runtime-configuration-in-a-runs-configuration "Direct link to Automatic inclusion of a sourcesets local run runtime configuration in a runs configuration")

默认情况下，源集的本地运行运行时配置会自动包含在运行的运行配置中。如果您想禁用此功能，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.sourcesets.automatic-inclusion-local-run-runtime=false
```

这相当于在 build.gradle 中设置以下内容：

```
runs {
    configureEach { run ->
        run.dependencies {
            runtime sourceSets.main.configurations.localRunRuntime
        }
    }
}
```

如果禁用此功能，则不会创建相关的本地运行运行时配置。

### IDE 集成[​](#ide-integrations "Direct link to IDE Integrations")

要禁用 IDE 集成，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.ide.enabled=false
```

#### IDEA[​](#idea "Direct link to IDEA")

要禁用 IDEA 集成，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.ide.idea.enabled=false
```

##### 使用 IDEA 运行[​](#run-with-idea "Direct link to Run with IDEA")

如果您已配置您的 IDEA IDE 使用其自己的编译器运行，可以通过在 gradle.properties 中设置以下属性来禁用 IDEA 编译器的自动检测：

```
neogradle.subsystems.conventions.ide.idea.compiler-detection=false
```

此自动检测将设置 DSL 属性，在运行和编译时禁用此属性意味着您需要自己设置此属性。

```
idea {
    project {
        runs {
            runWithIdea = true / false
        }
    }
}
```

注意

此 API 仅适用于根项目，因为那是 IDEA 项目模型所在的位置。

如果由于某种原因 IDEA 没有在根项目中正确注册其模型，您也可以使用：

```
minecraft {
    idea {
        runWithIdea = true / false
    }
}
```

警告

此模型是项目特定的，与上面显示的仅适用于根项目的 API 不同。因此，应同时在所有项目上设置。我们建议使用约定插件或类似的配置缓存兼容机制。

##### 使用参数文件[​](#using-arg-files "Direct link to Using Arg Files")

有时，当使用参数文件缩短其命令行参数并启用使用 Gradle 运行时（这是默认设置），IDEA 将生成一个配置缓存不兼容的初始化脚本。为了防止 Gradle 对 IDEA 的不兼容脚本感到不满，您可以让 NG 生成一个不缩短命令行的启动配置。

注意

这可能不适用于所有操作系统。某些路径配置可能会阻止您启动游戏，因为启动命令变得太长。

您可以使用以下约定：

```
neogradle.subsystems.conventions.ide.idea.use-args-file=false
```

或以下模型 API：

```
idea {
    project {
        runs {
            useArgsFile = true / false
        }
    }
}
```

将配置系统使用您首选的约定。

注意

此 API 仅适用于根项目，因为那是 IDEA 项目模型所在的位置。

如果由于某种原因 IDEA 没有在根项目中正确注册其模型，您也可以使用：

```
minecraft {
    idea {
        useArgsFile = true / false
    }
}
```

警告

此模型是项目特定的，与上面显示的仅适用于根项目的 API 不同。因此，应同时在所有项目上设置。我们建议使用约定插件或类似的配置缓存兼容机制。

##### 同步后任务使用[​](#post-sync-task-usage "Direct link to Post Sync Task Usage")

默认情况下，IDEA 中的导入在同步任务期间运行。如果您想禁用此功能，并使用同步后任务，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.ide.idea.use-post-sync-task=true
```

##### 重新配置 IDEA 单元测试模板[​](#reconfiguration-of-idea-unit-test-templates "Direct link to Reconfiguration of IDEA Unit Test Templates")

默认情况下，IDEA 单元测试模板不会重新配置以支持从边栏运行单元测试。您可以通过在 gradle.properties 中设置以下属性来启用此行为：

```
neogradle.subsystems.conventions.ide.idea.reconfigure-unit-test-templates=true
```

### 运行[​](#runs "Direct link to Runs")

要禁用运行约定，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.runs.enabled=false
```

#### 每个类型自动创建默认运行[​](#automatic-default-run-per-type "Direct link to Automatic default run per type")

默认情况下，为每种运行类型创建一个运行。如果您想禁用此功能，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.runs.create-default-run-per-type=false
```

#### DevLogin 约定[​](#devlogin-conventions "Direct link to DevLogin Conventions")

如果您想为所有客户端运行启用 dev login 工具，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.runs.devlogin.conventionForRun=true
```

这将为所有客户端运行启用 dev login 工具，除非显式禁用。

#### RenderDoc 约定[​](#renderdoc-conventions "Direct link to RenderDoc Conventions")

如果您想为所有客户端运行启用 render doc 工具，可以在 gradle.properties 中设置以下属性：

```
neogradle.subsystems.conventions.runs.renderdoc.conventionForRun=true
```

这将为所有客户端运行启用 render doc 工具，除非显式禁用。

## 工具覆盖[​](#tool-overrides "Direct link to Tool overrides")

要配置 NG 不同子系统使用的工具，可以使用子系统 dsl 和属性来配置以下工具：

### JST[​](#jst "Direct link to JST")

此工具被 parchment 子系统用于应用其名称和 javadoc，也被源访问转换器系统用于应用其转换。以下属性可用于配置 JST 工具：

```
neogradle.subsystems.tools.jst=<jst cli 工具的构件坐标>
```

### DevLogin[​](#devlogin "Direct link to DevLogin")

此工具被运行中的 dev login 子系统用于在客户端运行中启用 Minecraft 身份验证。以下属性可用于配置 DevLogin 工具：

```
neogradle.subsystems.tools.devLogin=<devlogin cli 工具的构件坐标>
```

有关相关工具、其发布版本和文档的更多信息，请参见此处：[Covers1624 的 DevLogin](https://github.com/covers1624/DevLogin)

### RenderDoc[​](#renderdoc "Direct link to RenderDoc")

此工具被运行中的 RenderDoc 子系统用于允许在客户端运行中捕获游戏帧。以下属性可用于配置 RenderDoc 工具：

```
neogradle.subsystems.tools.renderDoc.path=<RenderDoc 下载和安装目录的路径>
neogradle.subsystems.tools.renderDoc.version=<要使用的 RenderDoc 版本>
neogradle.subsystems.tools.renderDoc.renderNurse=<rendernurse 代理工具的构件坐标>
```

有关相关工具、其发布版本和文档的更多信息，请参见此处：[RenderDoc](https://renderdoc.org/) 和 [RenderNurse](https://github.com/neoforged/RenderNurse)

## 集中式缓存[​](#centralized-cache "Direct link to Centralized Cache")

NeoGradle 有一个集中式缓存，可用于存储反编译的 Minecraft 源代码、重新编译的 Minecraft 源代码以及其他复杂任务的任务输出。缓存默认启用，可以通过在 gradle.properties 中设置以下属性来禁用：

```
net.neoforged.gradle.caching.enabled=false
```

您可以通过运行以下命令清理存储在缓存中的构件：

```
./gradlew cleanCache
```

此命令在运行 clean 任务时也会自动运行。该命令将检查存储的构件数量是否高于配置的阈值，如果是，则删除最旧的构件，直到数量低于阈值。数量由 gradle.properties 中的以下属性配置：

```
net.neoforged.gradle.caching.maxCacheSize=<数字>
```

### 调试[​](#debugging "Direct link to Debugging")

有两个属性可以调整以获取有关缓存的更多信息：

```
net.neoforged.gradle.caching.logCacheHits=<true/false>
```

和

```
net.neoforged.gradle.caching.debug=<true/false>
```

第一个属性将在缓存命中时记录，第二个属性将记录有关缓存的一般信息，包括哈希的计算方式。如果您遇到缓存问题，可以启用这些属性以获取有关正在发生的情况的更多信息。