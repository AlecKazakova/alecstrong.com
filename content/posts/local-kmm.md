---
title: "Testing a Shared KMM Repo Locally"
date: 2021-05-03T21:37:14-04:00
draft: false
---

There's a lot of different ways of organizing your Kotlin Multiplatform Mobile codebase, Ben Asher
and I gave an overview of some of those at [KotlinConf 2019](https://youtu.be/je8aqW48JiA?t=1891),
where I described a setup where the shared code lives in your iOS and Android codebase and is copied
between them. The pro of this was an easy local development setup, but there are a lot of cons. We
moved away from this style to a separate (third) repository for the multiplatform code last year.

The general model for working in this new third repository is:

 1. Write shared code in the repository with unit tests.
 2. Publish a local version of the artifact (.jar for Android, .framework for iOS)
 3. Use that published local artifact in the Android and iOS apps to develop your feature
 4. Once those are good to go, open a PR on the multiplatform repo
 5. Push a release version of the multiplatform
 6. Open a PR on the Android/iOS repos with the integrated change.

There's lots of room for optimization here, but this post will focus on steps 2/3. Ideally we want
to get back the ease of having the multiplatform code live in the app repos, and thankfully there
are mechanisms to do just that both in Gradle and CocoaPods.

### Android

Consider this Gradle module in an Android repo which uses your multiplatform codebase:

```groovy
apply plugin: "com.android.library"
apply plugin: "org.jetbrains.kotlin.android"

dependencies {
  implementation "app.cash.mobile.multiplatform:my-artifact:1.0.0"
}
```

If you have the mobile multiplatform repository checked out locally, you can just use that local
copy directly using [Gradle composite builds](https://docs.gradle.org/current/userguide/composite_builds.html).
In your `settings.gradle`:

```groovy
includeBuild('path/to/mobile-multiplatform') {
  dependencySubstitution {
    substitute module("app.cash.mobile.multiplatform:my-artifact") with project(":my-module")
  }
}
```

Now instead of downloading and using a maven artifact, it will build the included project and then
use the Gradle subproject specified. Now you can make changes in your local copy of the multiplatform
repo and see them reflected immediately.

### iOS

Our iOS setup uses CocoaPods which enables local development. Kotlin comes bundled with a CocoaPods
integration you can apply to get things working.

In your multiplatform Gradle project:

```groovy
apply plugin: "org.jetbrains.kotlin.multiplatform"
apply plugin: "org.jetbrains.kotlin.native.cocoapods"

kotlin {
  cocoapods {
    // Configure fields required by CocoaPods.
    summary = "Local development of the kotlin multiplatform module."
    homepage = "https://something.com/mobile-multiplatform"

    // You will want this to be the same name as your published .framework 
    frameworkName = "MyKmm"
  }
}
```

Applying the CocoaPods plugin also automatically configures some framework binaries. In our case
we were creating those manually so instead we had to switch to configure the ones CocoaPods made.

```groovy
kotlin {
  sourceSets {
    configure(listOf(iosArm32, iosArm64, iosX64)) {
      compilations.findByName("main")?.source(iosMain)
      listOf(
              binaries.getFramework(NativeBuildType.DEBUG),
              binaries.getFramework(NativeBuildType.RELEASE)
      ).forEach {
        it.isStatic = true
        configurations.getByName(iosMain.apiConfigurationName).dependencies.forEach(it::export)
      }
    }
  }
}
```

Finally you will want to generate the `.podspec`, which you can do by running `./gradlew :my-module:podspec`.
The output may need to be manually changed to adhere to your published artifact's naming; in our case
the generated `.podspec` referred to the module name `umbrella` where we instead want it to use
the framework name `CaskKt`, so we had to rename from `umbrella.podspec` to `CashKt.podspec` and change
the `spec.name` property inside as well.

Now in your actual iOS repository you can swap out this line of code in `Podfile`:


```ruby
pod 'MyArtifact', '~> 1.0.0', source: some_pod_repo
```

To this:

```ruby
pod 'MyArtifact', :path => 'path/to/mobile-multiplatform/my-module/'
```

And running the iOS app will first build your local version of the multiplatform code and then run it. Yay!