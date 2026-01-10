# NeoGradle

[![Release](https://github.com/neoforged/NeoGradle/actions/workflows/release.yml/badge.svg?branch=NG_7.0)](https://github.com/neoforged/NeoGradle/actions/workflows/release.yml)

* * *

Minecraft mod development framework, used by NeoForge and FML for the Gradle build system.

For a quick start, see how the [NeoForge Mod Development Kit](https://github.com/neoforged/MDK) uses NeoGradle.

To see the latest available version of NeoGradle, visit the [NeoForged project page](https://projects.neoforged.net/neoforged/neogradle).

## Plugins[​](#plugins "Direct link to Plugins")

NeoGradle is separated into several different plugins that can be applied independently of each other.

### Userdev Plugin[​](#userdev-plugin "Direct link to Userdev Plugin")

This plugin is used for building mods with NeoForge. As a modder, this will in many cases be the only plugin you use.

```
plugins {
  id 'net.neoforged.gradle.userdev' version '<neogradle_version>'
}

dependencies {
  implementation 'net.neoforged:neoforge:<neoforge_version>'
}
```

#### Access Transformers[​](#-access-transformers "Direct link to -access-transformers")

The userdev plugin provides a way to configure access transformers for your mod. You need to create an access transformer configuration file in your resources directory, and then configure the userdev plugin to use it.

```
userdev {
    accessTransformer {
        file 'src/main/resources/META-INF/accesstransformer.cfg'
    }
}
```

The path here is up to you, and does not need to be included in your final jar.

#### Interface Injections[​](#-interface-injections "Direct link to -interface-injections")

The userdev plugin provides a way to configure interface injections for your mod. This allows you to have a decompiled minecraft artifact that contains the interfaces you want to inject via mixins already statically applied. The advantage of this approach is that you can use the interfaces in your code, and the mixins will be applied to the interfaces, and not the classes that implement them.

```
userdev {
    interfaceInjection {
        file 'src/main/resources/META-INF/interfaceinjection.json'
    }
}
```

You can find more information on the format of the file [here](https://github.com/neoforged/JavaSourceTransformer?tab=readme-ov-file#interface-injection).

#### Dependency management by the userdev plugin[​](#dependency-management-by-the-userdev-plugin "Direct link to Dependency management by the userdev plugin")

When this plugin detects a dependency on NeoForge, it will spring into action and create the necessary NeoForm runtime tasks to build a usable Minecraft JAR-file that contains the requested NeoForge version. It additionally (if configured to do so via conventions, which is the default) will create runs for your project, and add the necessary dependencies to the classpath of the run.

##### Version Catalogues[​](#version-catalogues "Direct link to Version Catalogues")

Using gradles modern version catalog feature means that dependencies are added at the very last moment possible, regardless of that is during script evaluation or not. This means that if you use this feature it might sometimes not be possible to configure the default runs exactly as you wished. In that case it might be beneficial to disable the conventions for runs, and configure them manually.

See the following example:

_lib.versions.toml:_

```
[versions]
# Neoforge Settings
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

You do not need to create all five of the different runs, only the ones you need. This is because at the point where gradle actually adds the dependency, we can not create any further runs, yet we can still configure them for you when the dependency is added.

### Common Plugin[​](#common-plugin "Direct link to Common Plugin")

#### Available Dependency Management[​](#-available-dependency-management "Direct link to -available-dependency-management")

##### Extra Jar[​](#-extra-jar "Direct link to -extra-jar")

The common plugin provides dependency management for extra-jar of minecraft. This is a special jar containing just the resources and assets of minecraft, no code. This is useful for mods that want to depend on the resources of minecraft, but not the code.

```
plugins {
  id 'net.neoforged.gradle.common' version '<neogradle_version>'
}

dependencies {
  implementation "net.minecraft:client:<minecraft_version>:client-extra"
}
```

#### Jar-In-Jar support[​](#jar-in-jar-support "Direct link to Jar-In-Jar support")

The common plugin provides support for jar-in-jar dependencies, both for publishing and when you consume them.

##### Consuming Jar-In-Jar dependencies[​](#consuming-jar-in-jar-dependencies "Direct link to Consuming Jar-In-Jar dependencies")

If you want to depend on a project that uses jar-in-jar dependencies, you can do so by adding the following to your build.gradle:

```
dependencies {
    implementation "project.with:jar-in-jar:1.2.3"
}
```

Obviously you need to replace `project.with:jar-in-jar:1.2.3` with the actual coordinates of the project you want to depend on, and any configuration can be used to determine the scope of your dependency.

##### Publishing[​](#-publishing "Direct link to -publishing")

If you want to publish a jar-in-jar dependency, you can do so by adding the following to your build.gradle:

```
dependencies {
    jarJar("project.you.want.to:include-as-jar-in-jar:[my.lowe.supported.version,potentially.my.upper.supported.version)") {
        version {
            prefer "the.version.you.want.to.use"
        }
    }
}
```

Important here to note is that specifying a version range is needed. Jar-in-jar dependencies are not supported with a single version, directly. If you need to specify a single version, you can do so by specifying the same version for both the lower and upper bounds of the version range: `[the.version.you.want.to.use]`.

###### Handling of moved Jar-In-Jar dependencies[​](#-handling-of-moved-jar-in-jar-dependencies "Direct link to -handling-of-moved-jar-in-jar-dependencies")

When dependency gets moved from one GAV to another, generally a transfer coordinate gets published, either via maven-metadata.xml, a seperate pom file, or via gradles available-at metadata. This can cause the version of the dependency to be different from the version you specified. It is best that you update your dependency to the new GAV to prevent problems and confusion in the future.

#### Managing runs[​](#managing-runs "Direct link to Managing runs")

The common plugin provides a way to manage runs in your project. Its main purpose is to ensure that whether you use the vanilla, neoform, platform or userdev modules, you can always manage your runs in the same way.

```
plugins {
  id 'net.neoforged.gradle.common' version '<neogradle_version>'
}

runs {
    //...Run configuration
}
```

##### Configuring runs[​](#-configuring-runs "Direct link to -configuring-runs")

When you create a run in your project, it will initially be empty. If you do not use a run type, or clone the configuration from another run, as described below, you will have to configure the run yourself.

```
runs {
    someRun {
        isIsSingleInstance true //This will make the run a single instance run, meaning that only one instance of the run can be run at a time
        mainClass 'com.example.Main' //This will set the main class of the run
        arguments 'arg1', 'arg2' //This will set the arguments of the run
        jvmArguments '-Xmx4G' //This will set the jvm arguments of the run
        isClient true //This will set the run to be a client run
        isServer true //This will set the run to be a server run
        isDataGenerator true //This will set the run to be a data gen run
        isGameTest true //This will set the run to be a game test run
        isJUnit true //This will set the run to be a junit run, indicating that a Unit Test environment should be used and not a normal run
        environmentVariables 'key1': 'value1', 'key2': 'value2' //This will set the environment variables of the run
        systemProperties 'key1': 'value1', 'key2': 'value2' //This will set the system properties of the run
        classPath.from project.configurations.runtimeClasspath //This will add an element to just the classpath of this run
        
        shouldBuildAllProjects true //This will set the run to build all projects before running
        workingDirectory file('some/path') //This will set the working directory of the run
    }
}
```

Beyond these basic configurations you can always look at the [Runs DSL object](https://github.com/neoforged/NeoGradle/blob/fec82f73860e8a3fdd21ac0701827ed890ff576e/dsl/common/src/main/groovy/net/neoforged/gradle/dsl/common/runs/run/Run.groovy) to see what other options are available.

##### Configuring run types[​](#-configuring-run-types "Direct link to -configuring-run-types")

If you are using run types that do not come from an SDK (like those that the userdev plugin), you can configure them as follows:

```
runs {
    someRun {
        isIsSingleInstance true //This will make the runs of this type a single instance run, meaning that only one instance of the run can be run at a time
        mainClass 'com.example.Main' //This will set the main class of the runs of this type
        arguments 'arg1', 'arg2' //This will set the arguments of the runs of this type
        jvmArguments '-Xmx4G' //This will set the jvm arguments of the runs of this type
        isClient true //This will set the run to be a client runs of this type
        isServer true //This will set the run to be a server runs of this type
        isDataGenerator true //This will set the run to be a data gen runs of this type
        isGameTest true //This will set the run to be a game test runs of this type
        isJUnit true //This will set the run to be a junit runs of this type, indicating that a Unit Test environment should be used and not a normal run
        environmentVariables 'key1': 'value1', 'key2': 'value2' //This will set the environment variables of the runs of this type
        systemProperties 'key1': 'value1', 'key2': 'value2' //This will set the system properties of the runs of this type
        classPath.from project.configurations.runtimeClasspath //This will add an element to just the classpath of this runs of this type
    }
}
```

##### Types of runs[​](#-types-of-runs "Direct link to -types-of-runs")

The common plugin manages the existence of runs, but it on its own does not create or configure a run. It just ensures that when you create a run tasks, and IDE runs, are created for it, based on the configuration of the run itself.

To configure a run based on a given template, run types exist. And if you use the userdev plugin for example then it will load the run types from the SDKs you depend on and allow you to configure runs based on those types.

###### Configuring runs by name[​](#-configuring-runs-by-name "Direct link to -configuring-runs-by-name")

First and foremost by default the common plugin will configure any run that is named identical to a run type with that run type. You can disable that for a run by setting:

```
runs {
    someRun {
        configureFromTypeWithName false
    }
}
```

###### Configuring run using types[​](#-configuring-run-using-types "Direct link to -configuring-run-using-types")

If you want to configure a run based on a run type, you can do so by setting:

```
runs {
    someRun {
        runType 'client' //In case you want to configure the run type based on its name
        configure runTypes.client //In case you want to configure the run type based on the run type object, this might not always be possible, due to the way gradle works
        
        //The following method is deprecated and will be removed in a future release
        configure 'client'
    }
}
```

Both of these will throw an exception if the run type could not be found during realization of the run to a task or ide run.

##### Configuration of runs[​](#-configuration-of-runs "Direct link to -configuration-of-runs")

When a run is added the common plugin will also inspect it and figure out if any SDK or MDK was added to any of its sourcesets, if so, it gives that SDK/MDK a chance to add its own settings to the run. This allows provides like NeoForge to preconfigure runs for you.

However, if you set up the run as follows:

```
runs {
    someRun {
        modSource sourceSets.some_version_a_sourceset
        modSource sourceSets.some_version_b_sourceset
    }
}
```

where the relevant sourcesets have a dependency in their compile classpath on the given sdk version, and those differ, and error will be raised. This is because the common plugin can not determine which version of the sdk to use for the run. To fix this, choose one of the versions and add it to the run:

```
runs {
    someRun {
        modSource sourceSets.some_version_a_sourceset
    }
}
```

or

```
runs {
    someRun {
        modSource sourceSets.some_version_b_sourceset
    }
}
```

###### Configuring run using another run[​](#-configuring-run-using-another-run "Direct link to -configuring-run-using-another-run")

Additionally, you can also clone a run into another run:

```
runs {
    someRun {
        run 'client' //In case you want to clone the client run into the someRun
        configure runTypes.client //In case you want to clone the client run type into the someRun
    }
}
```

##### Using DevLogin in runs[​](#using-devlogin-in-runs "Direct link to Using DevLogin in runs")

The DevLogin tool is a tool that allows you to log in to a Minecraft account without having to use the Minecraft launcher, during development. During first start it will show you a link to log in to your Minecraft account, and then you can use the tool to log in to your account. The credentials are cached on your local machine, and then reused for future logins, so that re-logging is only needed when the tokens expire. This tool is used by the runs subsystem to enable logged in plays on all client runs. The tool can be configured using the following properties:

```
neogradle.subsystems.devLogin.enabled=<true/false>
```

By default, the subsystem is enabled, and it will prepare everything for you to log in to a Minecraft account, however you will still need to enable it on the runs you want. If you want to disable this you can set the property to false, and then you will not be asked to log in, but use a random non-signed in dev account.

###### Per run configuration[​](#-per-run-configuration "Direct link to -per-run-configuration")

If you want to configure the dev login tool per run, you can do so by setting the following properties in your run configuration:

```
runs {
    someRun {
        devLogin {
            enabled true
        }
    }
}
```

This will enable the dev login tool for this run. By default, the dev login tool is disabled, and can only be enabled for client runs.

warning

If you enable the dev login tool for a non-client run, you will get an error message.

Additionally, it is possible to use a different user profile for the dev login tool, by setting the following property in your run configuration:

```
runs {
    someRun {
        devLogin {
            profile '<profile>'
        }
    }
}
```

If it is not set then the default profile will be used. See the DevLogin documentation for more information on profiles: [DevLogin by Covers1624](https://github.com/covers1624/DevLogin/blob/main/README.md#multiple-accounts)

###### Configurations[​](#configurations "Direct link to Configurations")

To add the dev login tool to your run we create a custom configuration to which we add the dev login tool. This configuration is created for the first source-set you register with the run as a mod source set. The suffix for the configuration can be configured to your personal preference, by setting the following property in your gradle.properties:

```
neogradle.subsystems.devLogin.configurationSuffix=<suffix>
```

By default, the suffix is set to "DevLoginLocalOnly". We use this approach to create a runtime only configuration, that does not leak to other consumers of your project.

##### Using RenderDoc in runs[​](#using-renderdoc-in-runs "Direct link to Using RenderDoc in runs")

The RenderDoc tool is a tool that allows you to capture frames from your game, and inspect them in its frame debugger. Our connector implementation can be optionally injected to start RenderDoc with your game.

###### Per run configuration[​](#-per-run-configuration-1 "Direct link to -per-run-configuration-1")

If you want to configure the render doc tool per run, you can do so by setting the following properties in your run configuration:

```
runs {
    someRun {
        renderDoc {
            enabled true
        }
    }
}
```

This will enable the render doc tool for this run. By default, the render doc tool is disabled, and can only be enabled for client runs.

warning

If you enable the render doc tool for a non-client run, you will get an error message.

##### Resource processing[​](#resource-processing "Direct link to Resource processing")

Gradle supports resource processing out of the box, using the `processResources` task (or equivalent for none main sourcesets). However, no IDE supports this out of the box. To provide you with the best experience NeoGradle, will run the `processResources` task for you, before you run the game.

If you are using IDEA and have enabled the "Build and Run using IntelliJ IDEA" option, then NeoGradle will additionally modify its runs, to ensure that the `processResources` task is run before the game is started, and redirects its output to a separate location in your build directory, and these processed resources are used during your run.

warning

This means that the resource files created by ideas compile process are not used, and you should not rely on them being up-to-date.

##### IDEA Compatibility[​](#idea-compatibility "Direct link to IDEA Compatibility")

Due to the way IDEA starts Unit Tests from the gutter, it is not possible to reconfigure these kinds of tests, if you are running with the IDEA testing engine. To support this scenario, by default NeoGradle will reconfigure IDEAs testing defaults to support running these tests within a unit test environment.

However, due to the many constructs it is not possible to configure the defaults correctly for everybody, if you have a none standard testing setup, you can configure the defaults like so:

```
idea {
    unitTests {
        //Normal run properties, and sourceset configuration as if you are configuring a run:
        modSource sourceSets.anotherTestSourceSet
    }
}
```

If you want to disable this feature, you can disable the relevant conventions, see [Disabling conventions](#disabling-conventions).

#### SourceSet Management[​](#-sourceset-management "Direct link to -sourceset-management")

The common plugin provides a way to manage sourcesets in your project a bit easier. In particular, it allows you to easily depend on a different sourceset, or inherit its dependencies.

warning

However, it is important to know that you can only do this for sourcesets from the same project. NeoGradle will throw an exception if you try to depend on a sourceset from another project.

##### Inheriting dependencies[​](#-inheriting-dependencies "Direct link to -inheriting-dependencies")

If you want to inherit the dependencies of another sourceset, you can do so by adding the following to your build.gradle:

```
sourceSets {
    someSourceSet {
        inherit.from sourceSets.someOtherSourceSet
    }
}
```

##### Depending on another sourceset[​](#-depending-on-another-sourceset "Direct link to -depending-on-another-sourceset")

If you want to depend on another sourceset, you can do so by adding the following to your build.gradle:

```
sourceSets {
    someSourceSet {
        depends.on sourceSets.someOtherSourceSet
    }
}
```

### NeoForm Runtime Plugin[​](#neoform-runtime-plugin "Direct link to NeoForm Runtime Plugin")

This plugin enables use of the NeoForm runtime and allows projects to depend directly on deobfuscated but otherwise unmodified Minecraft artifacts.

This plugin is used internally by other plugins and is usually only needed for advanced use cases.

```
plugins {
  id 'net.neoforged.gradle.neoform' version '<neogradle_version>'
}

dependencies {
  // For depending on a Minecraft JAR-file with both client- and server-classes
  implementation "net.minecraft:neoform_joined:<neoform-version>"
  
  // For depending on the Minecraft client JAR-file
  implementation "net.minecraft:neoform_client:<neoform-version>"
  
  // For depending on the Minecraft dedicated server JAR-file
  implementation "net.minecraft:neoform_server:<neoform-version>"
}
```

## Apply Parchment Mappings[​](#apply-parchment-mappings "Direct link to Apply Parchment Mappings")

To get human-readable parameter names in decompiled Minecraft source-code, as well as Javadocs, crowdsourced data from the [Parchment project](https://parchmentmc.org) can be applied to the Minecraft source-code before it is recompiled.

This is currently only supported when applying the NeoGradle userdev Plugin.

The most basic configuration is using the following properties in gradle.properties:

```
neogradle.subsystems.parchment.minecraftVersion=1.20.2
neogradle.subsystems.parchment.mappingsVersion=2023.12.10
```

The subsystem also has a Gradle DSL and supports more parameters, explained in the following Gradle snippet:

```
subsystems {
  parchment {
    // The Minecraft version for which the Parchment mappings were created.
    // This does not necessarily need to match the Minecraft version your mod targets
    // Defaults to the value of Gradle property neogradle.subsystems.parchment.minecraftVersion
    minecraftVersion = "1.20.2"
    
    // The version of Parchment mappings to apply.
    // See https://parchmentmc.org/docs/getting-started for a list.
    // Defaults to the value of Gradle property neogradle.subsystems.parchment.mappingsVersion
    mappingsVersion = "2023.12.10"
    
    // Overrides the full Maven coordinate of the Parchment artifact to use
    // This is computed from the minecraftVersion and mappingsVersion properties by default.
    // If you set this property explicitly, minecraftVersion and mappingsVersion will be ignored.
    // The built-in default value can also be overriden using the Gradle property neogradle.subsystems.parchment.parchmentArtifact
    // parchmentArtifact = "org.parchmentmc.data:parchment-$minecraftVersion:$mappingsVersion:checked@zip"
    
    // Set this to false if you don't want the https://maven.parchmentmc.org/ repository to be added automatically when
    // applying Parchment mappings is enabled
    // The built-in default value can also be overriden using the Gradle property neogradle.subsystems.parchment.addRepository
    // addRepository = true
    
    // Can be used to explicitly disable this subsystem. By default, it will be enabled automatically as soon
    // as parchmentArtifact or minecraftVersion and mappingsVersion are set.
    // The built-in default value can also be overriden using the Gradle property neogradle.subsystems.parchment.enabled
    // enabled = true
  }
}
```

## Advanced Settings[​](#advanced-settings "Direct link to Advanced Settings")

### Override Decompiler Settings[​](#-override-decompiler-settings "Direct link to -override-decompiler-settings")

The settings used by the decompiler when preparing Minecraft dependencies can be overridden using [Gradle properties](https://docs.gradle.org/current/userguide/project_properties.html). This can be useful to run NeoGradle on lower-end machines, at the cost of slower build times.

| Property | Description |
| --- | --- |
| `neogradle.subsystems.decompiler.maxMemory` | How much heap memory is given to the decompiler. Can be specified either in gigabyte (`4g`) or megabyte (`4096m`). |
| `neogradle.subsystems.decompiler.maxThreads` | By default the decompiler uses all available CPU cores. This setting can be used to limit it to a given number of threads, the lower the amount of threads the less memory is consumed. If set to 0 then the limit is disabled. |
| `neogradle.subsystems.decompiler.logLevel` | Can be used to override the [decompiler loglevel](https://vineflower.org/usage/#cmdoption-log). |

### Override Recompiler Settings[​](#override-recompiler-settings "Direct link to Override Recompiler Settings")

The settings used by Neogradle for recompiling the decompiled Minecraft source code can be customized using [Gradle properties](https://docs.gradle.org/current/userguide/project_properties.html).

| Property | Description |
| --- | --- |
| `neogradle.subsystems.recompiler.maxMemory` | How much heap memory is given to the decompiler. Can be specified either in gigabyte (`4g`) or megabyte (`4096m`). Defaults to `1g`. |
| `neogradle.subsystems.recompiler.jvmArgs` | Pass arbitrary JVM arguments to the forked Gradle process that runs the compiler. I.e. `-XX:+HeapDumpOnOutOfMemoryError` |
| `neogradle.subsystems.recompiler.args` | Pass additional command line arguments to the Java compiler. |
| `neogradle.subsystems.recompiler.shouldFork` | Indicates whether or not a process fork should be used for the recompiler. (Default is true). |

## Run specific dependency management[​](#-run-specific-dependency-management "Direct link to -run-specific-dependency-management")

warning

Minecraft Version 21.9 and later do not need Run Specific dependency management anymore. Please remove your gradle code. It is not used by NeoGradle anymore to configure NeoForge or FML

This implements run specific dependency management for the classpath of a run. In the past this had to happen via a manual modification of the "minecraft_classpath" token, however tokens don't exist anymore as a component that can be configured on a run. It was as such not possible to add none FML aware libraries to your classpath of a run.

### Usage:[​](#usage "Direct link to Usage:")

#### Direct[​](#direct "Direct link to Direct")

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

#### Configuration[​](#configuration "Direct link to Configuration")

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

#### Run Dependency Handler[​](#run-dependency-handler "Direct link to Run Dependency Handler")

The dependency handler on a run works very similar to a projects own dependency handler, however it has only one "configuration" available to add dependencies to: "runtime". Additionally, it provides a method to use when you want to turn an entire configuration into a runtime dependency.

## Handling of None-NeoGradle sibling projects[​](#handling-of-none-neogradle-sibling-projects "Direct link to Handling of None-NeoGradle sibling projects")

In general, we suggest, no strongly encourage, to **not** use fat jars for this solution. The process of creating a fat jar with all the code from your sibling projects is difficult to model in a way that is both correct and efficient for a dev project, especially if the sibling project does not use NeoGradle.

### Sibling project uses a NeoGradle module[​](#sibling-project-uses-a-neogradle-module "Direct link to Sibling project uses a NeoGradle module")

If it is possible to use a NeoGradle module (for example the Vanilla module, instead of VanillaGradle) then you can use the source-set's mod identifier:

```
sourceSets {
    main {
        run {
            modIdentifier '<some string that all projects in your fat jar have in common>'
        }
    }
}
```

The value of the modIdentifier does not matter here, all projects with the same source-set mod identifier will be included in the same fake fat jar when running your run.

### Sibling project does not use NeoGradle[​](#sibling-project-does-not-use-neogradle "Direct link to Sibling project does not use NeoGradle")

If the sibling project does not use NeoGradle, then you have to make sure that its Manifest is configured properly:

```
FMLModType: GAMELIBRARY #Or any other mod type that is not a mod, like LIBRARY
Automatic-Module-Name: '<some string that is unique to this project>'
```

warning

If you do this, then your sibling projects are not allowed to contain a class in the same package! This is because no two modules are allowed to contain the same package. If you have two sibling projects with a class in the same package, then you will need to move one of them!

### Including the sibling project in your run[​](#including-the-sibling-project-in-your-run "Direct link to Including the sibling project in your run")

To include the sibling project in your run, you need to add it as a modSource to your run:

```
runs {
    someRun {
        modSources {
            add project.sourceSets.main // Adds the owning projects main sourceset to a group based on that sourcesets mod identifier (could be anything here, depending on the sourcesets extension values, or the project name)
            add project(':api').sourceSets.main // Assuming the API project is not using NeoGradle, this would add the api project to a group using the `api` key, because the default mod identifier for non-neogradle projects is the projects name, here api
            local project(':api').sourceSets.main // Assuming the API project is not using NeoGradle, this would add the api project to a group using the owning projects name, instead of the api projects name as a fallback (could be anything here, depending on the sourcesets extension values, or the project name)
            add('something', project(':api').sourceSets.main) // This hardcodes the group identifier to 'something', performing no lookup of the mod identifier on the sourceset, or using the owning project, or the sourcesets project.
        }
    }
}
```

No other action is needed.

## Using conventions[​](#using-conventions "Direct link to Using conventions")

### Disabling conventions[​](#disabling-conventions "Direct link to Disabling conventions")

By default, conventions are enabled. If you want to disable conventions, you can do so by setting the following property in your gradle.properties:

```
neogradle.subsystems.conventions.enabled=false
```

We will consider the conventions to be enabled going forward, so if you want to disable them, you will have to do so explicitly.

### Configurations[​](#configurations-1 "Direct link to Configurations")

NeoGradle will add several `Configurations` to your project. This convention can be disabled by setting the following property in your gradle.properties:

```
neogradle.subsystems.conventions.configurations.enabled=false
```

Per SourceSet the following configurations are added, where XXX is the SourceSet name:

* XXXLocalRuntime
* XXXLocalRunRuntime

note

For this to work, your SourceSets need to be defined before your dependency block.

Per Run the following configurations are added:

* XXXRun

note

For this to work, your Runs need to be defined before your dependency block.

Globally the following configurations are added:

* runs

#### LocalRuntime (Per SourceSet)[​](#localruntime-per-sourceset "Direct link to LocalRuntime (Per SourceSet)")

This configuration is used to add dependencies to your local projects runtime only, without exposing them to the runtime of other projects. Requires source set conventions to be enabled

#### LocalRunRuntime (Per SourceSet)[​](#localrunruntime-per-sourceset "Direct link to LocalRunRuntime (Per SourceSet)")

This configuration is used to add dependencies to the local runtime of the runs you add the SourceSets too, without exposing them to the runtime of other runs. Requires source set conventions to be enabled

#### Run (Per Run)[​](#run-per-run "Direct link to Run (Per Run)")

This configuration is used to add dependencies to the runtime of a specific run only, without exposing them to the runtime of other runs.

#### run (Global)[​](#run-global "Direct link to run (Global)")

This configuration is used to add dependencies to the runtime of all runs.

### Sourceset Management[​](#sourceset-management "Direct link to Sourceset Management")

To disable the sourceset management, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.sourcesets.enabled=false
```

#### Automatic inclusion of the current project in its runs[​](#automatic-inclusion-of-the-current-project-in-its-runs "Direct link to Automatic inclusion of the current project in its runs")

By default, the current projects main sourceset is automatically included in its runs. If you want to disable this, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.sourcesets.automatic-inclusion=false
```

If you want to disable this, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.sourcesets.automatic-inclusion=false
```

This is equivalent to setting the following in your build.gradle:

```
runs {
    configureEach { run ->
        run.modSource sourceSets.main
    }
}
```

##### Automatic inclusion of a sourcesets local run runtime configuration in a runs configuration[​](#automatic-inclusion-of-a-sourcesets-local-run-runtime-configuration-in-a-runs-configuration "Direct link to Automatic inclusion of a sourcesets local run runtime configuration in a runs configuration")

By default, the local run runtime configuration of a sourceset is automatically included in the runs configuration of the run. If you want to disable this, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.sourcesets.automatic-inclusion-local-run-runtime=false
```

This is equivalent to setting the following in your build.gradle:

```
runs {
    configureEach { run ->
        run.dependencies {
            runtime sourceSets.main.configurations.localRunRuntime
        }
    }
}
```

If this functionality is disabled then the relevant configurations local run runtime configurations will not be created.

### IDE Integrations[​](#ide-integrations "Direct link to IDE Integrations")

To disable the IDE integrations, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.ide.enabled=false
```

#### IDEA[​](#idea "Direct link to IDEA")

To disable the IDEA integration, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.ide.idea.enabled=false
```

##### Run with IDEA[​](#run-with-idea "Direct link to Run with IDEA")

If you have configured your IDEA IDE to run with its own compiler, you can disable the autodetection of the IDEA compiler by setting the following property in your gradle.properties:

```
neogradle.subsystems.conventions.ide.idea.compiler-detection=false
```

This auto-detection will set the DSL property, disabling this property while running and compiling with idea means that you need to set this property yourself.

```
idea {
    project {
        runs {
            runWithIdea = true / false
        }
    }
}
```

note

This API is only available on the root project, since that is where the idea project model resides.

If for what ever reason IDEA does not properly register its model in your root project you can also use:

```
minecraft {
    idea {
        runWithIdea = true / false
    }
}
```

warning

This model is project specific, as apposed to the API shown above which is only available on the root project. It should as such be set on all projects at once. We suggest using a convention plugin or a similar Config Cache compatible mechanism.

##### Using Arg Files[​](#using-arg-files "Direct link to Using Arg Files")

Sometimes IDEA will generate a Config Cache incompatible init script when its command line arguments are shortened using an args file and running with Gradle is enabled (which is the default). To prevent gradle from being mad at idea for the incompatible script you can make NG generate a launch configuration that does not shorten the command line.

note

This might not work on all operating systems. Some configurations of paths might prevent you from launching the game because the launch command got too long.

You can either use the following convention:

```
neogradle.subsystems.conventions.ide.idea.use-args-file=false
```

Or the following model APIs:

```
idea {
    project {
        runs {
            useArgsFile = true / false
        }
    }
}
```

Will configure the system to use your preferred convention.

note

This API is only available on the root project, since that is where the idea project model resides.

If for what ever reason IDEA does not properly register its model in your root project you can also use:

```
minecraft {
    idea {
        useArgsFile = true / false
    }
}
```

warning

This model is project specific, as apposed to the API shown above which is only available on the root project. It should as such be set on all projects at once. We suggest using a convention plugin or a similar Config Cache compatible mechanism.

##### Post Sync Task Usage[​](#post-sync-task-usage "Direct link to Post Sync Task Usage")

By default, the import in IDEA is run during the sync task. If you want to disable this, and use a post sync task, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.ide.idea.use-post-sync-task=true
```

##### Reconfiguration of IDEA Unit Test Templates[​](#reconfiguration-of-idea-unit-test-templates "Direct link to Reconfiguration of IDEA Unit Test Templates")

By default, the IDEA unit test templates are not reconfigured to support running unit tests from the gutter. You can enable this behavior by setting the following property in your gradle.properties:

```
neogradle.subsystems.conventions.ide.idea.reconfigure-unit-test-templates=true
```

### Runs[​](#runs "Direct link to Runs")

To disable the runs conventions, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.runs.enabled=false
```

#### Automatic default run per type[​](#automatic-default-run-per-type "Direct link to Automatic default run per type")

By default, a run is created for each type of run. If you want to disable this, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.runs.create-default-run-per-type=false
```

#### DevLogin Conventions[​](#devlogin-conventions "Direct link to DevLogin Conventions")

If you want to enable the dev login tool for all client runs, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.runs.devlogin.conventionForRun=true
```

This will enable the dev login tool for all client runs, unless explicitly disabled.

#### RenderDoc Conventions[​](#renderdoc-conventions "Direct link to RenderDoc Conventions")

If you want to enable the render doc tool for all client runs, you can set the following property in your gradle.properties:

```
neogradle.subsystems.conventions.runs.renderdoc.conventionForRun=true
```

This will enable the render doc tool for all client runs, unless explicitly disabled.

## Tool overrides[​](#tool-overrides "Direct link to Tool overrides")

To configure tools used by different subsystems of NG, the subsystems dsl and properties can be used to configure the following tools:

### JST[​](#jst "Direct link to JST")

This tool is used by the parchment subsystem to apply its names and javadoc, as well as by the source access transformer system to apply its transformations. The following properties can be used to configure the JST tool:

```
neogradle.subsystems.tools.jst=<artifact coordinate for jst cli tool>
```

### DevLogin[​](#devlogin "Direct link to DevLogin")

This tool is used by the dev login subsystem in runs to enable Minecraft authentication in client runs. The following properties can be used to configure the DevLogin tool:

```
neogradle.subsystems.tools.devLogin=<artifact coordinate for devlogin cli tool>
```

More information on the relevant tool, its released version and documentation can be found here: [DevLogin by Covers1624](https://github.com/covers1624/DevLogin)

### RenderDoc[​](#renderdoc "Direct link to RenderDoc")

This tool is used by the RenderDoc subsystem in runs to allow capturing frames from the game in client runs. The following properties can be used to configure the RenderDoc tool:

```
neogradle.subsystems.tools.renderDoc.path=<path to the RenderDoc download and installation directory>
neogradle.subsystems.tools.renderDoc.version=<version of RenderDoc to use>
neogradle.subsystems.tools.renderDoc.renderNurse=<artifact coordinate for rendernurse agent tool>
```

More information on the relevant tool, its released version and documentation can be found here: [RenderDoc](https://renderdoc.org/) and [RenderNurse](https://github.com/neoforged/RenderNurse)

## Centralized Cache[​](#centralized-cache "Direct link to Centralized Cache")

NeoGradle has a centralized cache that can be used to store the decompiled Minecraft sources, the recompiled Minecraft sources, and other task outputs of complex tasks. The cache is enabled by default, and can be disabled by setting the following property in your gradle.properties:

```
net.neoforged.gradle.caching.enabled=false
```

You can clean the artifacts that are stored in the cache by running the following command:

```
./gradlew cleanCache
```

This command is also automatically run, when you run the clean task. The command will check if the stored artifact count is higher than the configured threshold, and if so, remove the oldest artifacts until the count is below the threshold. The count is configured by the following property in your gradle.properties:

```
net.neoforged.gradle.caching.maxCacheSize=<number>
```

### Debugging[​](#debugging "Direct link to Debugging")

There are two properties you can tweak to get more information about the cache:

```
net.neoforged.gradle.caching.logCacheHits=<true/false>
```

and

```
net.neoforged.gradle.caching.debug=<true/false>
```

The first property will log when a cache hit occurs, and the second property will log more information about the cache in general, including how hashes are calculated. If you are experiencing issues with the cache, you can enable these properties to get more information about what is happening.