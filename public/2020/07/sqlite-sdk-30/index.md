# Mysterious SQLite bugs and how to solve them.


TLDR: Android 30 upgrades SQLite from 3.22.0 -> 3.28.0, this introduces new alter table
behavior which will potentially cause runtime exceptions when `ALTER TABLE` statements are ran on
tables which are used in a view. To preserve old behavior turn on `PRAGMA legacy_alter_table=ON`
before running your migrations.

---

Late last week we experienced a weird bug on Cash App happening to a few of our developers - 
reproducibly but not for everyone. The crash pointed to something happening in our SQLite migrations:

```
Caused by: android.database.sqlite.SQLiteException: error in view activityRecipient: no such table: main.instrumentLinkingConfig (code 1 SQLITE_ERROR)
        at android.database.sqlite.SQLiteConnection.nativeExecute(SQLiteConnection.java:-2)
        at android.database.sqlite.SQLiteConnection.execute(SQLiteConnection.java:707)
        at android.database.sqlite.SQLiteSession.execute(SQLiteSession.java:621)
        at android.database.sqlite.SQLiteStatement.execute(SQLiteStatement.java:46)
        at com.squareup.sqldelight.android.AndroidPreparedStatement.execute(AndroidSqliteDriver.kt:2)
        at com.squareup.sqldelight.android.AndroidSqliteDriver$execute$2.invoke(AndroidSqliteDriver.kt:2)
        at com.squareup.sqldelight.android.AndroidSqliteDriver.execute(AndroidSqliteDriver.kt:4)
        at com.squareup.sqldelight.android.AndroidSqliteDriver.execute(AndroidSqliteDriver.kt:10)
        at com.squareup.scannerview.R$layout.execute$default(Unknown:1)
        at com.squareup.cash.db.db.CashDatabaseImpl$Schema.migrate(CashDatabaseImpl.kt:819)
```

We use SQLDelight's migration verification so getting a runtime error is a bad look, but even worse
this stacktrace is incorrect. This line is a dead giveaway:

```
at com.squareup.scannerview.R$layout.execute$default(Unknown:1)
```

Nothing in our generated SQLDelight code is referencing that class so something is clearly wrong
here. The line in `CashDatabaseImpl$Schema` is also incorrect, which is even worse because it means
we don't know which line of code is crashing. There's no obvious way to understand what's going on
unless you already know what's going on, so here's what's going on: bytecode optimizers break 
stacktraces because they're changing the names and locations of symbols. A great example is an
optimizer that inlines code - it's moving the code around to a new location, when it does crash its crashing in bytecode
that doesn't decompile to source code. To solve this optimizers give you a mapping file which tells
other tools how to go from a stack frame for the compiled code to one which accurately reflects your
source code.

The conditions for having one of these mangled stack traces are pretty rare. The crash needs to come
from an optimized app - for development we don't optimize Cash App so local stack traces are never
obfuscated. For our internal and production releases the mapping file gets uploaded automatically
to bugsnag using the [gradle plugin](https://github.com/bugsnag/bugsnag-android-gradle-plugin) and
any crash reporter will have a similar tool. Unfortunately there's an [issue](https://github.com/bugsnag/bugsnag-android-gradle-plugin/issues/199)
causing some of our mapping files to not upload, which is how we wound up with this obfuscated stack trace.

There's no obvious way to tell if your stack trace is incorrect other than manually checking to see
if it looks better mapped. We can map the stacktrace using
[ReTrace](https://www.guardsquare.com/en/products/proguard/manual/retrace) which is a tool part of
ProGuard to retrieve the original stacktrace based on an obfuscated one + the mapping file. You can
generate the mapping.txt file by assembling the obfuscated app and grabbing the mapping file from
`build/outputs/mapping`. Once we use this we get a better stacktrace:

```
com.squareup.cash.db.db.CashDatabaseImpl$Schema.migrate(CashDatabaseImpl.java:7600)
```

And looking at that line of code we see:

```sql
driver.execute(null,
            "ALTER TABLE newInstrumentLinkingConfig RENAME TO instrumentLinkingConfig", 0)
```

Which given the original error message is pretty confusing:

```
error in view activityRecipient: no such table: main.instrumentLinkingConfig
```

Given it's working for some of us and not working for others, my hunch was that it was dependent
on the version of SQLite we were on, so I checked the
[versions of SQLite on Android](https://stackoverflow.com/questions/2421189/version-of-sqlite-used-in-android)
and found that SDK 30 introduced a pretty big version bump for SQLite. Next was to check the 
[SQLite Releases](https://www.sqlite.org/changes.html#version_3_28_0) for anything that references
"View" (literally did a cmd+f of "View" on that page) and found this:

```
2018-09-15 (3.25.0)
2. Enhancements the ALTER TABLE command:
   a. Add support for renaming columns within a table using ALTER TABLE table RENAME COLUMN oldname TO newname.
   b. Fix table rename feature so that it also updates references to the renamed table in triggers and views.
```

2b) is the one we care about. When SQLite does an ALTER TABLE command as of version 3.25.0, it finds
references to that table in VIEWS/TRIGGERS and updates them. The problem in our case was that our
migration file looked like this:

```sql
DROP TABLE instrumentLinkingConfig;
ALTER TABLE newInstrumentLinkingConfig RENAME TO instrumentLinkingConfig;
```

So when it went to go look at views during the `ALTER TABLE` statement, it would find ones
that reference the (now non-existent) `instrumentLinkingConfig`, and throw an error.

This is only really an issue because with Android source code you're targeting multiple versions
of SQLite (depending on the OS version), if you ship unbundled SQLite (like the 
[requery one](https://github.com/requery/sqlite-android)) what you should really be doing is using
the `ALTER TABLE ADD COLUMN` support which was added in the same SQLite version to avoid doing the
`DROP TABLE old/ALTER TABLE new RENAME TO old` dance we had to do on old SQLite for adding a column.
This would avoid issues around temporarily invalid views. For those of us still using the bundled
SQLite in android, the workaround is to enable the PRAGMA added in `3.25.2`: 
`PRAGMA legacy_alter_table=ON`.

On Android we're pretty used to testing on a different SDK to reproduce UI bugs and glitches, really
the only recommendation here is to do the same for SQLite issues unless you're shipping unbundled.
Outside of that do a thorough read of the release notes, and always double check your tools to make
sure they're not giving you red herrings.
