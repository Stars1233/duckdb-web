---
layout: post
title: "The Quack Protocol for DuckDB"
author: "Hannes Mühleisen"
thumb: "/images/blog/thumbs/quack-release.svg"
image: "/images/blog/thumbs/quack-release.jpg"
excerpt: "TODO"
tags: ["extension"]
---

...

## Introducing the Quack Protocol for DuckDB

What do two (or more) ducks do if they want to talk to each other? They [quack](https://en.wikipedia.org/wiki/Duck#Communication)! So it is quite natural that we need to call the protocol that two DuckDB instances can use to talk to each other “Quack”, too! We had the opportunity to design a database protocol from scratch in 2026 without having to consider any legacy, which is quite luxurious. We were able to learn from the existing protocols, including the more recent Arrow Flight SQL and others. Before we dive into how Quack works internally, let's see how it works from a user perspective. First, you need two DuckDB instances. That’s right, DuckDB will act both as a client and as a server! The two instances can be on different computers a world apart (or in space) or just two different terminal windows on your laptop. First, we need to install the Quack extension in both DuckDB instances. For now, Quack lives in the `core_nightly` repository and is available in [DuckDB 1.5.2]({% link install/index.html %}), the current release version:

<div class="duck-diagram" markdown="1">

<div class="duck-diagram-box" markdown="1">

#### <svg class="icon"><use href="#database-01"></use></svg> DuckDB #1

```sql
INSTALL quack FROM core_nightly;
LOAD quack;

CALL quack_serve(
    'quack:localhost',
    token = 'super_secret'
);

CREATE TABLE hello AS
    FROM VALUES ('world') v(s);
```

</div>

<div class="duck-diagram-arrow">quack:</div>

<div class="duck-diagram-box" markdown="1">

#### <svg class="icon"><use href="#database-01"></use></svg> DuckDB #2

```sql
INSTALL quack FROM core_nightly;
LOAD quack;

CREATE SECRET (
    TYPE quack,
    TOKEN 'super_secret'
);

ATTACH 'quack:localhost' AS remote;
FROM remote.hello;
```

</div>

</div>

This should show the content of the remote table hello, `world` in DuckDB #2. Witchcraft! We can also copy data from local to remote:

<div class="duck-diagram" markdown="1">

<div class="duck-diagram-box" markdown="1">

#### <svg class="icon"><use href="#database-01"></use></svg> DuckDB #1

```sql



-- step two
FROM hello2;
```

</div>

<div class="duck-diagram-arrow">quack:</div>

<div class="duck-diagram-box" markdown="1">

#### <svg class="icon"><use href="#database-01"></use></svg> DuckDB #2

```sql
-- step one
CREATE TABLE remote.hello2 AS
    FROM VALUES ('world2') v(s);
```

</div>

</div>

Similarly, you should see `world2` in the output on DuckDB \#1. Obviously those are the most minimal examples we can think of. Tables can be much more complex, queries can be much more complex, data volumes can be quite vast (see below). There is also a way to just ship an entire verbatim query to the remote side using the `query` function, which is better for very complex queries on large datasets and offers more control over what exactly is executed remotely:

<div class="duck-diagram" markdown="1">

<div class="duck-diagram-box" markdown="1">

#### <svg class="icon"><use href="#database-01"></use></svg> DuckDB #1

```text
```

</div>

<div class="duck-diagram-arrow">quack:</div>

<div class="duck-diagram-box" markdown="1">

#### <svg class="icon"><use href="#database-01"></use></svg> DuckDB #2

```sql
FROM remote.query(
    'SELECT s FROM hello'
);
```

</div>

</div>

Of course there is much more to see here. Please [consult our documentation]({% link docs/current/quack/overview.md %}) for more details.

...