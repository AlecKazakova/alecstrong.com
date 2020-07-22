# SQLite Helper Queries


In programming one of the earliest tools we're given for simplifying complex code is helper functions.
Splitting code into smaller more digestible blocks makes it easier to maintain and read, which is
usually more important than code that's easier to write. In SQL we have tools for accomplishing
the same thing! One of the bigger ones is views which
can help share an underlying query between multiple other ones. Leandro Favarin wrote a great
post on how to put views into practice in SQLite [here](https://leandrofavarin.com/sqldelight-powerful-codegen).

Another strategy for simplifying a query is common table expressions. Take for example a query like this:

```sql
SELECT *
FROM album
WHERE year_of_release = (
  SELECT max(year_of_release)
  FROM album AS artistAlbum
  WHERE artistAlbum.artist = album.artist
);
```

The query finds all the latest albums from every artist (meaning if an artist releases 2 albums in 2019 but none afterwards, those albums appear).
It's already pretty complicated so lets write a common table expression which we can use later in the query:

```sql
WITH artistInfo(name, activeUntil) AS (
  SELECT artist, max(year_of_release)
  FROM album
)

SELECT *
FROM album
JOIN artistInfo ON (artist = artistInfo.name)
WHERE year_of_release = artistInfo.activeUntil;
```

More verbose, but hopefully you can see how this would scale a bit nicer. If we assume the data populating `artistInfo`
gets more complex, the query below doesn't need to grow in complexity.
 
---
 
The catch here is now we're using joins,
but this actually has another added benefit to it - which we'll illustrate by explaining the query plan on these two queries.

```sql
EXPLAIN QUERY PLAN
SELECT *
FROM album
WHERE year_of_release = (
  SELECT max(year_of_release)
  FROM album AS artistAlbum
  WHERE artistAlbum.artist = album.artist
);
```

| id | parent | detail                            |
|----|--------|-----------------------------------|
| 2  | 0      | SCAN TABLE album                  |
| 5  | 0      | CORRELATED SCALAR SUBQUERY 1      |
| 10 | 5      | SEARCH TABLE album AS artistAlbum |

Because the subquery is `CORRELATED` it runs once per row of album, which is expensive for large data sets.
The subsequent `SEARCH` comes from the `WHERE artistAlbum.artist = album.artist`, there's no index on the `album.artist`
column so it just has to search the table for a match. Now we'll check out the one using a common table expression.

```sql
EXPLAIN QUERY PLAN
WITH artistInfo(name, activeUntil) AS (
  SELECT artist, max(year_of_release)
  FROM album
)

SELECT *
FROM album
JOIN artistInfo ON (artist = artistInfo.name)
WHERE year_of_release = artistInfo.activeUntil;
```

| id | parent | detail             |
|----|--------|--------------------|
| 3  | 0      | MATERIALIZE 1      |
| 7  | 3      | SEARCH TABLE album |
| 23 | 0      | SCAN SUBQUERY 1    |
| 25 | 0      | SCAN TABLE album   |

Now instead of the correlated subquery, we're materializing that subquery to start and then joining it with album. Materializing
just happens once so this query becomes a lot more performant.

---
 
There's one other huge benefit to common table expressions, which is that you can do
recursion! They're harder to follow and potentially defeat "easier to read and maintain" but sometimes they're the simplest solution and worth
knowing. Here's a recursive query that comes in handy for splitting a single string into a table where each row is one word:

```sql
WITH RECURSIVE
split_string(word, str) AS (
    SELECT '', :input || ' '
    UNION ALL SELECT
      substr(str, 0, instr(str, ' ')),
      substr(str, instr(str, ' ') + 1)
    FROM split_string WHERE str != ''
)

SELECT ...
```

To show how this works lets run it with `:input` = `"hello recursive world"`:

| word      | str                    |
|-----------|------------------------|
|           | hello recursive world  |
| hello     | recursive world        |
| recursive | world                  |
| world     |                        |

We're making heavy use of the `instr` function to find the next space, take the word before that and leave the remaining string. Recursive
queries work like a queue, the first part of the query (everything before `UNION ALL`) populates the queue initially, and
then the sqlite runtime pops off a row from the queue, runs the query after the `UNION ALL` using only that row, and pushes the results onto the
queue. Eventually `str` is empty so the query after `UNION ALL` pushes nothing onto the queue, so it ends. The results above are both the queue forming
and the table created at the end.
 
Since we only care about the words you just have to modify how you select from the query:

```sql
SELECT word
FROM split_string
WHERE word != '';
```

| word      |
|-----------|
| hello     |
| recursive |
| world     |

Nice! Sometimes you want to be able to `JOIN` a runtime parameter for sqlite, and this is how I do it. Again it has the benefit of running once,
which can be a lot more performant than writing the query to use `WHERE ___ IN ____` sometimes.


