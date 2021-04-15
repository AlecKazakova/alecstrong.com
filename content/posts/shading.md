---
title: "What we do in the shadows"
date: 2021-04-03T17:24:07-04:00
draft: false
---

If you're in the business of publishing JVM libraries its possible you've needed to deal with
dependency shading - bundling external jars in with your library and renaming the packages along
the way to prevent a [diamond dependency conflict](https://jlbp.dev/what-is-a-diamond-dependency-conflict#:~:text=A%20diamond%20dependency%20conflict%20is,features%20that%20the%20consumers%20expect.).

If your published JVM library is built with Gradle it's then likely you've used [John Engleman's 
shading plugin 'shadow'](https://github.com/johnrengelman/shadow). It's really a fantastic plugin
to solve this problem, thank you mr engleman. There's thorough documentation including some on
how to [shade Gradle plugins](https://imperceptiblethoughts.com/shadow/plugins/), which I needed to
do for SQLDelight and subsequently failed at for the past year until finally getting it working
recently. I do not
blame the plugin or the documentation, everything you need is there it's just an incredibly hard
problem that took me a long time to wrap my head around, so I am writing this for future JVM library
authors bundling IntelliJ IDEA in a Gradle plugin of which I am sure there will be many.

One thing that's incredibly useful to know: `.jar` files are just `.zip` files. You can just rename
the file to have the `.zip` extension, unzip that file and then peruse all the classes and whatnot
inside. I found this to be the easiest way to debug why my shadow jar was causing issues.

### Setup

As of writing the shadow documentation is slightly out of date (it's still using the compile 
configuration) and so on first attempt shading the SQLDelight Gradle plugin looked something like
this:

```groovy
// sqldelight-gradle-plugin
apply plugin: 'com.github.johnrengelman.shadow'

dependencies {
  implementation project(':sqlite-migrations')
  implementation project(':sqldelight-compiler')
  
  implementation deps.intellij.core
  implementation deps.intellij.java
  implementation deps.intellij.lang
  implementation deps.intellij.testFramework

  compileOnly gradleApi()
  implementation deps.plugins.kotlin
  compileOnly deps.plugins.android
}

task relocateShadowJar(type: ConfigureShadowRelocation) {
  target = tasks.shadowJar
}

tasks.shadowJar.dependsOn tasks.relocateShadowJar
```

Some of these dependencies require explanation - SQLDelight uses the IntelliJ APIs for its compiler,
as seen in it's `build.gradle`:

```groovy
// sqldelight-compiler
dependencies {
  implementation deps.sqliteJdbc

  compileOnly deps.intellij.core
  compileOnly deps.intellij.lang
  compileOnly deps.intellij.java
  compileOnly deps.intellij.testFramework
}
```

Marking a dependency as `compileOnly` means that those classes will be used during
compilation, but someone else will provide them at runtime. We do this because the IntelliJ plugin
also uses the compiler but already has the IntelliJ APIs on its runtime classpath.

However the Gradle plugin is not running in IntelliJ - we need to include those artifacts as
dependencies and can do that by marking them as `implementation` as shown in the Gradle plugins
dependencies block above. `gradleApi()` and `deps.plugins.android` are both marked `compileOnly` because we also
expect those dependencies to be available at runtime without us asking for them.

### Problem 1: Jar too big

This doesn't work. IntelliJ is enormous and bloats the shadow jar to be bigger than the 65k limit
jars have on class files[^1]. Android developers have probably had nightmares about that number.
Easy enough to fix! minimize the shadow jar:

```groovy
tasks.getByName("shadowJar").configure {
  dependsOn("relocateShadowJar")
  minimize()
}
```

### Problem 2: Transitive Dependencies

Now our jar is small enough, but when we run the SQLDelight tests they crash with this:

```
No suitable driver found for jdbc:sqlite
```

Which after way too much sitting and staring at my setup I realized was caused by [this](https://github.com/xerial/sqlite-jdbc/blob/master/src/main/resources/java.sql.Driver),
and `minimize()` stripping the actual `Driver` implementation since it isn't _technically_ used in
my source code anywhere, it was just referenced in a Java resources file. No problem, you can make
sure minimize avoids certain packages:

```groovy
tasks.getByName("shadowJar").configure {
  minimize {
    exclude(dependency('org.xerial:sqlite-jdbc:*'))
  }
}
```

Now when we go to run the SQLDelight tests we get something about joda time not being able to figure
out the time zone. "Ahhhhhhhhhhhh" this is so far from the problem I'm trying to solve, and I certainly don't want
to be leaking transitive dependencies into the implementation details of the Gradle plugin,
so a new route needed to be formed.

I really _only_ want to shade the IntelliJ dependencies, because those are the
ones that cause [major problems](https://github.com/cashapp/sqldelight/issues/1998).
The tricky thing about this is that shadow will not filter [transitive dependencies](https://imperceptiblethoughts.com/shadow/configuration/dependencies/#filtering-dependencies)
when shadowing, so if you shade a project dependency (in this case my `:sqlite-migrations` dependency),
you _have_ to shade all of it's dependencies (in this case, xerial and transitively jodatime). The trick
to get around this goes back to `compileOnly` vs `implementation`, we can control which dependencies are
shaded at the `sqldelight-gradle-plugin` level, so if we just mark all transitive dependencies as `compileOnly`
we can pick and choose which ones we want shaded.

Before I show the resulting Gradle (it's not pretty), I'll recap:
 1. We need to pick and choose which dependencies to shade, in this case the IntelliJ dependencies
    and any dependencies that use IntelliJ.
 2. `compileOnly` dependencies do not get shaded.
 3. `shade` dependencies will be bundled into the resulting jar
 4. we still need `implementation` dependencies (like xerial:sqlite-jdbc) to be a normal (non-shaded) dependency.

Okay here's the Gradle file!

```groovy
apply plugin: 'com.github.johnrengelman.shadow'

configurations {
  shade
}

configurations.compileOnly.extendsFrom(configurations.shade)

dependencies {
  implementation deps.sqliteJdbc
  implementation deps.objectDiff
  implementation deps.schemaCrawler.tools
  implementation deps.schemaCrawler.sqlite

  shade deps.sqlitePsi
  shade project(':sqlite-migrations')
  shade project(':sqldelight-compiler')
  shade deps.intellij.core
  shade deps.intellij.java
  shade deps.intellij.lang
  shade deps.intellij.testFramework

  compileOnly gradleApi()
  implementation deps.plugins.kotlin
  compileOnly deps.plugins.android
}


tasks.register("relocateShadowJar", ConfigureShadowRelocation.class) {
  target = tasks.shadowJar
  prefix = "sqldelight"
}

tasks.getByName("shadowJar").configure {
  dependsOn("relocateShadowJar")
  minimize()
  configurations = [project.configurations.shade]
}
```

We now have three configurations:
 1. `compileOnly` for dependencies that will be available at runtime that we do not need to depend on at all.
 2. `implementation` for dependencies we do not want shaded (ie just use normal maven coordinates).
 3. `shade` for dependencies that we want shaded (bundled into the artifact jar with package names changed so we dont get conflicts).

The core shaded dependencies are all the `deps.intellij` ones, the other three (`:sqlite-migrations`, `:sqldelight-compiler`, and `deps.sqlitePsi`)
also need to be shaded because they transitively depend on the intellij apis - if they didn't get shaded they would still
reference `org.jetbrains.intellij` instead of `sqldelight.org.jetbrains.intellij` (the new shaded dependencies)
and get NoClassDef errors. Importantly the `implementation` dependencies in this module must be marked as `compileOnly` in
those downstream modules (like `:sqlite-migrations`) otherwise they would get shaded.

### Problem 3: The Gradle plugin is missing (relocating project dependencies)

I wish we were done. Still loads to go. Now we run our Gradle plugin and find that it is missing:

```
NoClassDefFoundError: Cannot find class com.squareup.sqldelight.SqlDelightPlugin
```

Whenever you see a NoClassDefFoundError in this process, the easiest thing to do is the `.zip` trick
from above and start searching. What I found in my `.jar` was that `SqlDelightPlugin` was there but 
had been moved to `sqldelight.com.squareup.sqldelight.SqlDelightPlugin`: it was relocated
as part of the shading process. The `ConfigureShadowRelocation` task is pretty straightforwad and
looking through it's [source code](https://github.com/johnrengelman/shadow/blob/master/src/main/groovy/com/github/jengelman/gradle/plugins/shadow/tasks/ConfigureShadowRelocation.groovy)
we can see whats happening:

```groovy
JarFile jf = new JarFile(jar)
jf.entries().each { entry ->
    if (entry.name.endsWith(".class")) {
        packages << entry.name[0..entry.name.lastIndexOf('/')-1].replaceAll('/', '.')
    }
}
```

For every dependency in the `shade` configuration we locate all its packages and register them to
be relocated. Unfortunately because we shade a project dependency (`:sqldelight-compiler`),
the task is finding the `com.squareup.sqldelight` package in that project dependency and asking that it
be relocated. This also means the root project that shares the same package will have **its** dependencies
relocated too, and in this case the Gradle plugin itself. There is no officially documented way to remove
packages to be relocated but I was able to plug in to the task directly like this:

```groovy
tasks.getByName("shadowJar").configure {
  doFirst {
    relocators = relocators.grep {
      !it.getPattern().startsWith("com.squareup.sqldelight")
    }
  }
}
```

### Problem 4: The Jar is still Too Damn Big

We're no longer hitting limits for the size of our jar, but looking into its contents we still see
a lot of extras from intellij: including things like svg and png files, which are definitely not
needed for a Gradle plugin to function. The only things really needed for a Gradle plugin are the
class files, so we can configure the shadowJar to only include those:

```groovy
tasks.getByName("shadowJar").configure {
  ...
  include '*.jar'
  include '**/*.class'
  include 'META-INF/gradle-plugins/*'
}
```

We need to `include '*.jar'` because the shadow plugin works in phases, and on its first phase it
is dealing purely with `.jar` files and not `.class` files yet. We also need to make sure we include
the Gradle plugin registration files.

### Problem 5: Don't shade entire programming languages

SqlDelight makes heavy use of kotlinpoet to do it's codegen, which in turn inspects parts of the
kotlin stdlib in order to know what to generate. Because of that transitive dependency this setup
was shading the kotlin stdlib and then generating code against that shaded dependency, and not the
actual real kotlin stdlib. This resulted in some REALLY weird compile time exceptions that looked
something like

```
Unexpected type FunctionN, expected FunctionN
```

The same happened with groovy, as our Gradle plugin uses some groovy reflection in its implementation.
The type that SQLDelight was given was a `groovy.Closure`, but it was looking for a `sqldelight.groovy.Closure`.

If you are inspecting your shadow jar `.zip` and see that it has and programming languages in there,
you definitely want to exclude those. Similar to the Gradle API we're guaranteed something like
groovy is going to be there, and transitively we'll pick up kotlin as well. Similar to our
includes we can choose which classes not to include in our shadowJar, but we also need to make sure
we dont relocate the packages for those standard libs:

```groovy
doFirst {
  relocators = relocators.grep {
    !it.getPattern().startsWith("com.squareup.sqldelight") &&
            !it.getPattern().startsWith("groovy") &&
            !it.getPattern().startsWith("kotlin")
  }
}

exclude '/groovy**'
exclude '/kotlin/**'
```

### Finished

So end the woes of shading our Gradle plugin. [Here](https://github.com/cashapp/sqldelight/blob/master/sqldelight-gradle-plugin/build.gradle#L83-L112) is the final setup
for SQLDelight with all the steps outlined above. If you're a user of SQLDelight the only thing thats changing
is that Gradle plugin shrank from 56mb to 13mb (!!!) and hopefully you will encounter no classpath issues
using the project. I hope I never have to think about this again.

  [^1]: I cannot remember if class files were the thing limited by 65k. It might have been raw resources.