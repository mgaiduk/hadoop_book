## Gradle + Kotlin Hello World
### About jvm and Kotlin
In around 1995, Java as a language was created. Apart from being a computer language, it also came with "jvm" - a lowlevel bytecode interpreter tool that allows you to launch jvm-compatible programs written in any jvm-based language (Java, Kotlin, Scala, Groovy) on any OS and cpu architecture. The architecture part is quite handy for a tool like Hadoop - we are going to write and compile some code on our laptop (mac m1, arm cpu architecture in my case) and launch it on many workers in the cloud, with potentially different architecture, without having to worry about recompilation and compatibility.  

Java is one of the most popular programming languages, used for backed, frontent (app & web) development. Together with other JVM compatible languages, it dominates a lot of fields, open source big data processing being one example with Apache Hadoop, Apache Spark, Apache Beam, Apache Flink and other Apache Something framework written in Java and having main client libraries in Java as well.  

Jvm-compatible languages are completely interoperable. That means that you can use Java library from your Kotlin code, you can write a project with Java and Kotlin classes intermingled, including inheriting classes from one language in the other - something I didn't see anywhere else, apart from maybe Swift and Objective-C pair, and you will face almost no challenges in making that work once you understand some basics.  

Kotlin seems to be gaining popularity in the recent years: it is cleaner and more modern than Java, with better async and fiber support. In addition, learning Kotlin is fun! 
### About gradle
Gradle positions itself as a "developer productivity and build automation tool", which tells you almost nothing about what it actually is and how to use it. Gradle official website is not helpful in that respect either.  

This is why I made an entire chapter mostly about getting started with Gradle.  

So what is Gradle? Gradle, the way I see and use it, is a build tool. Java/Kotlin default tools are already quite good for building project: they allow you to write code in an organized manner, spread across multiple files and folders. Gradle adds 2 important features: fetching external dependencies (like hadoop lib) that are not part of STL, and easy building of a fully-packed .jar files with all runtime dependencies included, like Kotlin runtime and STL.  

Gradle is language-agnostic, i.e., it can in theory be used to build c++ or Swift projects (although I have never seen anyone doing that). Because of that, we will need to be writing some extra configuration code to just tell Gradle that we are interested in building a Kotlin program.  

Major tools I know for building jvm projects are Maven (focuses on Java projects only), sbt (for scala projects), and Gradle (multi-language, which is perfect for Kotlin collaboration with Hadoop libs written in Java). Gradle is also the most popular build tool for android apps.
### Getting started - creating a gradle project
To get started, we need to jave Java/JVM and Gradle already installed. (google it!)  

I know 2 ways of creating a proper Kotlin project with all configuration code already in place: `gradle init` and creating a project in Intellij Idea.   
In `gradle init`:
- Select type of project to generate: 2 (application). Basic will not create any configuration for building.
- Select implementation language: 4 (kotlin)
- Split functionality across multiple subprojects 1 (no)
- Select build script DSL: Kotlin
- Generate build using new APIs and behavior: no
- Project name: ...
- Source package: ...  

In Intellij idea, click "new project" (without any "generators" selected), Language - Kotlin, build system - Gradle, Gradle DSL: Kotlin, add sample code: true.  
I prefer creating a project in Intellij Idea, because it generates less useless extenral dependencies (`gradle init` adds a dependency on `com.google.guava` for some reason).

**Gradle DSL** is the language in which the actual configuration code will be written. Options are Groovy (old style) and Kotlin (newer). We choose Kotlin because its newer, and because we want to write configuration in the same language as the project itself :)  

After creating a project in Intellij Idea, a whole bunch of files and folders were generated. Lets go over all of them:  

- **gradlew**: gradlew, gradle.bat, gradle/wrapper/gradle-wrapper.properties and gradle/wrapper/gradle-wrapper.jar. `gradlew` is a tool that is used to "bootstrap" gradle on different systems to make sure that the project is built with the same version of gradle. The version of gradle to use is written in `gradle/wrapper/gradle-wrapper.properties`. The way it works is like this: we only use `gradle` tool for the initial `gradle init` command. After that we only use `gradlew`: it looks in the wrapper config, downloads the proper version of gradle if necessary, and then uses it to build the project or run other tasks. Unfortunately, Gradle official website uses plain `gradle` tool in examples, and Gradle did a lot of breaking changes over the years, so it is common to face build issues in case you are trying to build the project with the wrong gradle version.   

- **configuration files**: gradle.properties, settings.gradle.kts, build.gradle.kts. I am not sure why are there so many of them. `build.gradle.kts` is the main build file with all the logic; other files just seem to have a bunch of options like project and package name. This is what the content of `build.gradle.kts` look like:
```
plugins {
    kotlin("jvm") version "1.8.0"
    application
}

group = "org.example"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(kotlin("test"))
}

tasks.test {
    useJUnitPlatform()
}

kotlin {
    jvmToolchain(11)
}

application {
    mainClass.set("MainKt")
}
```
So what is going on here? First and foremost, Gradle is a tool that revolves around running Tasks. Examples of a task: build and run a project (a `run` task), compile a project into a jar (the `jar` task). You can run `gradlew tasks` to find out what tasks there are for the current project. The next important entity is a `plugin`. Since Gradle itself is language agnostic, it can do almost nothing without external plugins. Here, we have announced that we are going to use `kotlin` plugin, `1.8.0` version (this version is the Kotlin language version). This plugin allows us to compile Kotlin code. Then, we use `application` plugin. Here is the [documentation](https://docs.gradle.org/current/userguide/application_plugin.html) for the application plugin. This plugin is made for compiling and running JVM programs.  

After declaring the usage of plugins, we later configure them:
```
kotlin {
    jvmToolchain(11)
}   
```
Here, we tell Kotlin plugin which version of the JVm to use.
```
application {
    mainClass.set("MainKt")
}
```
Here, we tell the Application plugin what the main class of the application must be. This is the JVM thing - actual classes are build entities, and to know how to run a program we need to know the entrypoint class.  Note that in Java, directory structure and class package structure must match, so we would have to put `src.main.Java.Main` here. There is no such requirement in Kotlin, instead, our `Main.kt` kotlin file is translated into a `MainKt` java class.  

`repositories` section defines repositories to fetch external dependencies from. Hadoop library is hosted on `mavenCentral` repository.  

`dependencies` section specifies build dependencies. Here, we just defined "testImplementation" that will only be used by tests, and will not be compiled into release build. Later on, we would add some release dependencies as well.  

Finally, `tasks.test` defines a `test` task that can be run using `gradlew test`. It just proxies testing to JUnit lib.

In `src/main/kotlin/Main.kt` we see the actual code of our program:

```
fun main(args: Array<String>) {
    println("Hello World!")

    // Try adding program arguments via Run/Debug configuration.
    // Learn more about running applications: https://www.jetbrains.com/help/idea/running-applications.html.
    println("Program arguments: ${args.joinToString()}")
}
```
We can run it with `gradlew run`. To package it in a jar, we will need to add the following to `build.gradle.kts`:
```
tasks.jar {
    manifest.attributes["Main-Class"] = "MainKt"
    val dependencies = configurations
        .runtimeClasspath
        .get()
        .map(::zipTree)
    from(dependencies)
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```
Here, we override the `jar` task, telling gradle to set the main class to MainKt, and to iterate over all dependencies and add them to the jar. This is necessary because we need to have kotlin runtime and stl included in the jar. If we run `gradlew jar` now, a jar appears at `build/libs` directory that we can run like this: `java -jar build/libs/hello_world-1.0-SNAPSHOT.jar` 

This concludes gradle/kotlin introduction. In the next chapter, we will start writing an actual Hadoop program.