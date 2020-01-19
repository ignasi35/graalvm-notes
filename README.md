# Graal Notes

## Install

Using `sdkman` is dead simple to install a CE of a Graal-ready JDK:

```bash
sdk list java | grep grl
sdk install java 19.3.1.r8-grl
```

Once you have a JDK with Graal you still need to install `native-image` for it.

```bash
## First make sure you use the appropriate JDK 
sdk use java 19.3.1.r8-grl
## then install the native-image binary
gu install native-image
```

## Usage

### sbt install 

Use `sbt-native-packager`'s `GraalVMNativeImagePlugin` plugin:

```scala
    .enablePlugins(GraalVMNativeImagePlugin)
```
```bash
sbt> graalvm-native-image:packageBin
```

### sbt usage


There are a couple of dependencies only used at compile time which I'm not entirely sure what they are for:

```scala
  ## val GraalVersion = "19.2.1" // or "19.3.1"
  "org.graalvm.sdk" % "graal-sdk" % GraalVersion % "provided", 
  "com.oracle.substratevm" % "svm" % GraalVersion % "provided",
```

### Configuring the `native-image` command

#### Reflection/Resources/JNI/Proxy 

The code using reflection, resources, JNI or Java Proxy classes needs special settings. You may:

1. create those files manually
2. use files provided by the library creators
3. use the `native-image` agent to create those files on-the-fly from your running application.

***NOTE:*** the `***-config.json` files required by `native-image` may have content that's JDK specific. For example, if your application uses `sun.misc.Unsafe` (directly or via a library) then that class will be on the `***-config.json` files. Therefore, your `***-config.json` will only be usable from JDK 8 since that class was removed in JDK 9. If you use option `3.` aboce, make sure to use the same JDK version across all the involved environments (dev, CI, build, prod,...)

Once you've created all four `***-config.json` files (some of them may be empty) you need to keep them under source control management (SCM). You have two options:

***Option 1: Use `META-INF/graal`***

Simply place your `{resource,reflect,proxy,jni}-config.json` on `src/main/resources/META-INF/graal` and they'll be automatically packaged and accessible by `native-image`.

***Option 1: Use a custom folder***

You may also place your `{resource,reflect,proxy,jni}-config.json` on an arbitrary folder (e.g. `src/main/resources/my-files` and then set it up in `build.sbt`:

```scala
val graalConfigurationDirectory = settingKey[File]("The directory where Graal configuration lives")
val graalResourcesConfigurationFile = settingKey[File]("The file for Graal's resource-config.json")
val graalReflectConfigurationFile = settingKey[File]("The file for Graal's reflect-config.json")

val graalSettings = Seq(
    graalConfigurationDirectory := (Compile / resourceDirectory).value / "graal",
    graalResourcesConfigurationFile := graalConfigurationDirectory.value / "resource-config.json",
    graalReflectConfigurationFile := graalConfigurationDirectory.value / "reflect-config.json",
    graalVMNativeImageOptions ++= Seq(
        ... 
        "-H:ResourceConfigurationFiles=" + graalResourcesConfigurationFile.value.getAbsolutePath,
        "-H:ReflectionConfigurationFiles=" + graalReflectConfigurationFile.value.getAbsolutePath,
        ...
    )
)
```

When creating the `***-config.json` files you should consider composability (since `native-image` will merge all the files in the classpath). For example, if you are using Akka 2.5 (at the moment no later version is supported) you may use the libraries provided by `"com.github.vmencik" %% "graal-***" % "0.5.0`. These artifacts provide the appropriate configs:

```scala
  ## latest version when writing this was "0.5.0" and only worked for Akka 2.5 (not 2.6)
  "com.github.vmencik" %% "graal-akka-actor" % graalAkkaVersion,
  "com.github.vmencik" %% "graal-akka-stream" % graalAkkaVersion,
  "com.github.vmencik" %% "graal-akka-http" % graalAkkaVersion,
  "com.github.vmencik" %% "graal-akka-slf4j" % graalAkkaVersion
```

### Build-time vs Run-time initialization

When `native-image` is compiling your bytecode into a native binary it'll crawl the bytecode encountering uses of reflection, JNI or loading of resources that it will be able to deal with thanks to the `***-config.json` files you've configured using `-H:ReflectionConfigurationFiles` (and related paramters). But there are other situations where a new settings is required. Some classes must be eagerly initialized at build time otherwise they'll be missing on the binary and the execution of the application will fail.

???