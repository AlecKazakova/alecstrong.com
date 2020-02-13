# Supporting Multiple Dialects in SQLDelight


With SQLDelight 1.3.0 and above we now support multiple dialects of SQL (at
time of writing: SQLite 3.18, SQLite 3.24, and MySQL). They're not complete implementations of any of
those dialects but they support necessary features and the real meat of the release is a straightforward
framework for adding parts of the dialects incrementally. For example, SQLite 3.24 is just SQLite 3.18 with upsert
on top (thank you [Angus Holder](https://github.com/angusholder)!).

In this post I'm going to go over how it's all possible. I'll start by introducing Grammar-Kit from JetBrains
as the underlying tool for it and then I'll talk about how I abuse it to accomplish composable parsing.

---

## Grammar Kit

SQLDelight is a compiler, so it relies heavily on a lexer which takes text and turns them into tokens, and a parser
which takes those tokens and turns them into a tree of rules (abstract syntax tree or AST). ASTs become the API
that your compiler or any other tools use to interact with source code. This is how IntelliJ works, but it requires
strict adherence to its own AST interface called PSI. JetBrains definitely has extremely comprehensive parsers for
all kinds of SQL dialects which produce PSI, but since DataGrip is proprietary it's all closed source and I have to
write my own.

For one SQL dialect it's manageable. There's an open source [ANTLR Grammar](https://github.com/bkiers/sqlite-parser) which
is what most modern SQLite compilers I know of are using. There's also a tool for going between [ANTLR's AST and PSI](https://github.com/antlr/antlr4-intellij-adaptor), enabling
IntelliJ. This was what the first version of SQLDelight did and it worked fine, but managing two separate ASTs was awful.

JetBrains has their own parser generator called [Grammar Kit](https://github.com/JetBrains/Grammar-Kit)
which outputs its AST as PSI, which is fantastic if you're maintaining an IntelliJ plugin in addition to your
compiler. The only issue is that the PSI runtime is deeply embedded in IntelliJ and so running it outside of
IntelliJ is challenging. The SQLDelight compiler effectively runs headless IntelliJ which is also awful, but I've
since appreciated that it is less awful than maintaining two ASTs. If you've wondered why the SQLDelight gradle plugin
binary is 50mb - this is why. It's kind of running IntelliJ.

### The Grammar

To power Grammar Kit we need a compatible grammar in BNF format, which didn't exist at the time (~2 years ago). Back then SQLite published 
a [bnf](https://www.sqlite.org/docsrc/doc/trunk/art/syntax/all-bnf.html) that has since been taken down because it wasn't being maintained.
It was really the only starting point so I wrote a [python script](https://github.com/AlecStrong/sqlite-bnf) that reads from that site
and outputs a Grammar Kit compatible `.bnf`. 

BNF grammars are pretty straightforward, its a list of rules and definitions for those rules. As an example here's
what a simplified column rule would look like in BNF

```
column_def ::= column_name type_name (column_constraint)*
```

A name, a type, and a list of constraints. There's a ton of rules in addition to this one but we'll roll with it to illustrate
how multiple dialects come in. Here's the definition of the `type_name` rule.

```
type_name ::= 'REAL' | 'INTEGER' | 'TEXT' | 'BLOB'
```

This is SQLite specific, though not truthfully since SQLite lets you write anything for the column type. We intentionally
restrict it down because it's [less confusing](https://www.sqlite.org/datatype3.html). Works great for SQLite but now we want 
to support MySQL so this doesn't work.

### External rules

The high level premise of how we'll accomplish this is that when we go to parse the `type_name` rule, depending on the dialect we're in
we'll use a different definition. Grammar Kit wants to generate super deterministic code and wants all the rules in the same file, but you can
cheat the system by providing an "external" rule. An external rule is one that you are providing *in code* instead of in the grammar, and they look like this.

```
column_def ::= column_name <<typeNameExt>> (column_constraint)*
```

As long as there's a static function named `typeNameExt`, that's what will be invoked for that step of parsing. Here's what
that static function looks like:

```kotlin
@JvmStatic
fun typeNameExt(
  builder: PsiBuilder,
  level: Int
): Boolean
```

In order to support different implementations of this rule, we have a single static field for the parser to use:

```kotlin
object SqlParserUtil {
  var type_name: GeneratedParserUtilBase.Parser? = null

  @JvmStatic
  fun typeNameExt(
    builder: PsiBuilder,
    level: Int,
    type_name: GeneratedParserUtilBase.Parser
  ): Boolean = (this.type_name ?: type_name).parse(builder, level)
}
```

Notice too that we've added a parameter `type_name` to the static function; it's actually the original rule that we might be overriding
if someone sets the static `type_name` variable. Grammar Kit external rules can actually be *passed* another rule within the grammar, which looks
like this:

```
column_def ::= column_name <<typeNameExt type_name>> (column_constraint)*

type_name ::= identifier // ANSI SQL lets you write anything for the type
```

Lets recap: when the generated parser goes to parse the `column_def` rule, it will first parse `column_name` normally,
then it will parse `type_name` by calling `typeNameExt` (code I wrote), passing it the original rule `type_name`, and then in
my code I will invoke the original rule or an override if one was set.

This means in a separate bnf file I can create the MySQL rule:

```
type_name ::= VARCHAR | INTEGER | VARBINARY
```

Which generates its own `MySQLParser`, and then at runtime I can override the rule:

```kotlin
SqlParserUtil.type_name = MySQLParser.type_name
```

Wow! Now everything works. 

No

### Shaping the AST

The problem with this approach is that the AST we're given won't actually have a method on it for the type_name, since it doesn't know what to expect.

```kotlin
interface ColumnDef {
  fun getColumnName(): ColumnName
  
  fun getColumnConstraints(): List<ColumnConstraint>
}
```

Notice there's no type name. This is a problem since now we're losing the type safety we cared about in the first place. Alright back to the grammar...

We can actually instruct a rule to take the shape of a different rule if we tell it to, using a Grammar Kit attribute called `elementType`. It looks like this:

```
fake column_def ::= column_name type_name (column_constraint)*

column_def_real ::= column_name <<typeNameExt type_name>> (column_constraint)* {
  elementType = column_def
}
```

What happens here is that when we parse the `column_def_real` rule, it will put the resulting element in the type for the `column_def` rule. We
mark the `column_def` rule as `fake` so that we don't actually generate a parser for it. It's not actually being used as a rule so we don't need
to generate any parsing code for it.

So here's the type we end up getting when the `column_def_real` rule is parsed:

```kotlin
interface ColumnDef {
  fun getColumnName(): ColumnName

  fun getTypeName(): TypeName

  fun getColumnConstraints(): List<ColumnConstraint>
}
```

This is what we want!

### What about those other rules!

Yea good question. The hypothesis here is that I can apply this strategy to each rule and have a completely composable grammar, which future
proofs me for other SQL dialects or features within a dialect. So lets just do that! Here's the grammar for that rule:

```
fake column_def ::= column_name type_name (column_constraint)*

column_def_real ::= <<columnNameExt column_name_real>> <<typeNameExt type_name_real>> (<<columnConstraintExt column_constraint_real>>)* {
  elementType = column_def
}
```

```kotlin
object SqlParserUtil {
  var column_name: GeneratedParserUtilBase.Parser? = null
  var type_name: GeneratedParserUtilBase.Parser? = null
  var column_constraint: GeneratedParserUtilBase.Parser? = null

  @JvmStatic
  fun columnNameExt(
    builder: PsiBuilder,
    level: Int,
    column_name: GeneratedParserUtilBase.Parser
  ): Boolean = (this.column_name ?: column_name).parse(builder, level)

  @JvmStatic
  fun typeNameExt(
    builder: PsiBuilder,
    level: Int,
    type_name: GeneratedParserUtilBase.Parser
  ): Boolean = (this.type_name ?: type_name).parse(builder, level)

  @JvmStatic
  fun columnConstraintExt(
    builder: PsiBuilder,
    level: Int,
    column_constraint: GeneratedParserUtilBase.Parser
  ): Boolean = (this.column_constraint ?: column_constraint).parse(builder, level)
}
```

Great, and now we'll just repeat this for all the SQL rules!

```
create_trigger ::= CREATE [ TEMP | TEMPORARY ] TRIGGER [ IF NOT EXISTS ] [ database_name '.' ] trigger_name [ BEFORE | AFTER | INSTEAD OF ] ( DELETE | INSERT | UPDATE [ OF column_name ( ',' column_name ) * ] ) ON table_name [ FOR EACH ROW ] [ WHEN expr ] BEGIN ( (update_stmt | insert_stmt | delete_stmt | compound_select_stmt ) ';' ) + END
```

😱

Not happening. Thankfully this is a great opportunity for some codegen, so I wrote [Grammar Kit Composer](https://github.com/AlecStrong/Grammar-Kit-Composer)
to do all the heavy lifting for you. You just write your Grammar Kit grammars normally:

```
column_def ::= column_name type_name (column_constraint)*
```

Apply the gradle plugin:

```groovy
buildscript {
  dependencies {
    classpath "com.alecstrong:grammar-kit-composer:0.1.3"
  }
}

apply plugin: "com.alecstrong.grammar.kit.composer"
```

And then running `build` will first generate the composable grammar, then the parser util, then the normal Grammar Kit outputs (your PSI tree and the parser).

## Grammar Kit Composer

Grammar Kit Composer lets you indicate what parser you're overriding and then referencing rules or overriding rules:


```
//MySQL.bnf
{
  overrides="com.alecstrong.sql.psi.core.SqlParser"
}
type_name ::= VARCHAR | INTEGER | VARBINARY {
  overrides=true
}
```

Then the generated parserUtil comes with a setup method:

```kotlin
MySQLParserUtil.overrideSqlParser()
```

### sql-psi

SQLDelight is all about codegen, not actually parsing sql or all this composable grammar nonsense, so that all lives in a separate repo 
[sql-psi](https://github.com/AlecStrong/sql-psi). If you're interested in seeing more dialects or language features supported downstream in
SQLDelight, sql-psi is where it needs to happen.