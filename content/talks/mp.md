# Multiplatform Persistence

A lot of client code is already multiplatform - the tools underneath client backends like SQLite, protobufs, and HTTP are themselves platform agnostic. Codebases for Android and iOS already look similar on top of these tools until exposed to platform specific interfaces. Realistically how much could be shared across multiple platforms without affecting any codebase significantly?

This talk will go over SQLDelight 1.0 - a recent release which has enabled multiplatform SQLite development using Kotlin Native, as well as how to architect your app’s data layer to be shared across platforms. It will focus on how SQLDelight can be best used from both Android and iOS while feeling natural to Kotlin and Swift users alike, and go over features introduced in the release that affect all platforms. Finally we will explore the future of multiplatform including other components of your app which are good or bad candidates for making platform agnostic and what future releases of SQLDelight will include.

{{< youtube gyCrZ3z85xk >}}

---

{{< speakerdeck 2c009013af2a4b89b3df7cf59820ab6f 1.77777777777778>}}