# What's the big IDEA? Writing IntelliJ plugins for Kotlin

_Presented with Egor Andreevich_

With Kotlin having full interop with Java, mixed codebases have become common and effective - but have made writing developer tools more challenging. How do you support multiple languages with a single tool? How do you convert existing plugins from Java to Kotlin and is there a way to avoid having to?

This talk will cover UAST (Universal Abstract Syntax Tree), an API for working with languages generically. With UAST you can write a single tool that will work for both Java and Kotlin - no special casing needed. We will talk about how to setup a plugin to use UAST and walk through a sample that works on mixed codebases.

The talk will also dive into the types of problems you can solve by writing an IntelliJ plugin, as well as other applications for UAST outside of IntelliJ IDEA.

{{< youtube j2tvi4GbOr4 >}}

---

{{< speakerdeck baaab3df5420471293440d924bfc81ae 1.77777777777778>}}