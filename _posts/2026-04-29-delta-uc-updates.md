---
layout: post
title: "Delta Grows Up: Writes, Time Travel and Unity Catalog"
author: "Ben Fleis"
excerpt: "DuckDB's Delta and Unity Catalog extensions shed their experimental tags — now with writes, time travel, and catalog-managed commits."
tags: ["extensions"]
thumb:
image:
---

Welcome back! While we here at DuckDB Labs are typically of the quacking
persuasion, we’ve been busy as beavers, shoring up our Delta to prepare for
what’s next… Unity Catalog! Let’s look at how DuckDB’s Delta and Unity Catalog
extensions have grown up – enough to shed the experimental tag – and see what
has changed since our [last
update]({% post_url 2025-03-21-maximizing-your-delta-scan-performance %}).

## Time to Open the Delta

In that last update we highlighted performance wins, particularly file skipping
via filter pushdowns, and metadata caching with snapshot pinning. Now we build
on those with extended functionality, and even more performance gains!

### Building Up the Delta (Lake): Writes

What fun are reads without writes? The big addition since we last chatted is
`INSERT` support! It works as simply as you expect. Let's assume you have a delta
table ready to go. `INSERT` away, it's that simple:

```sql
-- Schema: (text VARCHAR, code BIGINT)
ATTACH './path/to/my_table' AS my_table (TYPE delta);

INSERT INTO my_table VALUES ('Question 2', 2), ('The Answer', 42);

-- Bulk insert from a query
INSERT INTO my_table FROM (SELECT text || ' (copy)', code + 100 FROM my_table);
```

Also worth calling out – multiple `INSERT`s within a `BEGIN`/`COMMIT` block are
stored as a single Delta version: one atomic commit, one new log entry. And,
as you'll see later, this works with catalogs too! UPDATE, MERGE, and DELETE
are not yet supported, but squarely in our
future work list.

### Time Travel

DuckDB's Delta extension now supports [time
travel](https://delta.io/blog/2023-02-01-delta-lake-time-travel/). Any Delta
table can be queried as of a particular version. DuckDB supports binding to a
specific `VERSION` either at `ATTACH` time, or as part of an individual query.

Let's assume that we have built up the above `my_table` incrementally, with
versions 0, 1, and 2 containing:

| Version | Contents                                                                                   |
| ------- | ------------------------------------------------------------------------------------------ |
| 0       | `('Question 1', 1)`                                                                        |
| 1       | + `('Question 2', 2)`, `('The Answer', 42)`                                                |
| 2       | + `('Question 1 (copy)', 101)`, `('Question 2 (copy)', 102)`, `('The Answer (copy)', 142)` |

You can attach normally and query arbitrary versions inline as needed — the
most flexible approach:

```sql
ATTACH './path/to/my_table' AS my_table (TYPE delta);

SELECT count() FROM my_table AT (VERSION => 0);  -- → 1  (Question 1 only)
SELECT count() FROM my_table AT (VERSION => 1);  -- → 3  (after first insert)
SELECT count() FROM my_table;                    -- → 6  (latest)
```

Or attach, pinned to a specific version, which is useful when you want a stable
reference that never changes, regardless of future writes:

```sql
-- Always v1, no matter what gets written later
ATTACH './path/to/my_table' AS my_table_v1 (TYPE delta, VERSION 1);
SELECT count() FROM my_table_v1;   -- → 3

-- Locked to whatever was latest at attach time
ATTACH './path/to/my_table' AS my_table_pinned (TYPE delta, PIN_SNAPSHOT);
SELECT count() FROM my_table_pinned;  -- → 6
```

On top of that, snapshot loading will be performed incrementally when possible,
making time travel even faster. Consider a table where some initial queries are
made against version 16:

```sql
ATTACH './path/to/table' AS t (TYPE delta, VERSION 16);
SELECT count() FROM t;  -- → 17
```

And now some work needs to be done against version 20. If we peek under the
hood (warning: sneaky code follows), we'll see that none of the previously
loaded delta log metadata files were re-loaded:

```sql
SET enable_logging = true;
SET delta_kernel_logging = true;
CALL enable_logging('DeltaKernel', level = 'trace');

ATTACH './path/to/table' AS t (TYPE delta, VERSION 20);
SELECT count() FROM t;  -- → 21

-- Delta kernel logs 'Provisionally selecting ... <version>.json' whenever it reads
-- a log file from scratch. We search for any such message referencing a
-- zero-padded log filename — zero matches means the cached v16 snapshot was
-- extended incrementally rather than rebuilt.
SELECT count() FROM duckdb_logs
WHERE type = 'DeltaKernel'
  AND message LIKE '%00000000000000000%.json%';
-- → 0
```

In delta lakes with thousands or millions of snapshots, incremental loading
provides a big win when working across multiple versions. (Note: At time of
writing, incremental snapshot loading is supported in nightly builds, and will
be included in the next release, v1.5.3.)

### Growing Up: No Longer a [Kit](https://duckduckgo.com/?q=what+is+a+baby+beaver+called%3F)

The DuckDB Delta extension has grown up quite a bit since a year ago. Atop
previous performance improvements, it now supports writes and time travel. So,
we got rid of the experimental tag! And even better, all these features open
the door to something bigger: Unity Catalog coordination.

## Unity Catalog Support atop the Delta

Data lake systems excel at scale, but raw scale creates its own challenges. As
your data assets multiply, you need a way to discover what exists, control who
can access it, audit how it's being used, and coordinate writes across multiple
engines. Data catalogs have evolved to address exactly these needs — sitting
above the storage layer to manage the metadata, governance, and transactional
bookkeeping that make large-scale data lakes effective. The OSS
Unity Catalog team has a [good
overview](https://www.unitycatalog.io/blogs/data-catalog) if you'd like to go
deeper; the concepts apply broadly regardless of which catalog you use.

### What is Unity Catalog?

Unity Catalog (UC for short) is an open standard for governing data and AI
assets — tables, volumes, models, and functions — across engines and clouds. It
gives you a single place to discover, audit, and control access to your data,
regardless of what's reading or writing it. There are two main flavors: OSS
Unity Catalog, which you can self-host (and Docker-ify in minutes), and
Databricks Unity Catalog, the managed version. Like Delta, the DuckDB Unity
Catalog extension has shed its experimental tag — let's put both to work.

### Getting Started: OSS Unity Catalog

We've put together a [playground Docker image — OSS Unity Catalog and DuckDB
bundled together](https://github.com/benfleis/duckdb-unitycatalog-playground/),
so you can follow along without any setup beyond docker build-and-run. Grab it
if you would like to walk through the samples or experiment on your own. (If
you'd prefer to run OSS UC directly, the official image is the upstream of our
playground.)

Let's start with Docker. Assuming you now have the image running, it
already executed (roughly) the following steps in the build phase to prepare
our playground:

```bash
# Create a schema
/home/unitycatalog/bin/uc schema create --catalog unity --name my_schema

# Create the "pets" table
/home/unitycatalog/bin/uc table create \
    --full_name        unity.my_schema.pets \
    --columns          "uuid STRING, name STRING, age INT, adopted BOOLEAN" \
    --format           DELTA \
    --storage_location file:///home/unitycatalog/etc/data/external/unity/my_schema/tables/pets
```

After that, we can test things out from DuckDB. To see for
yourself, `docker exec -it duckdb-playground duckdb` will give you a DuckDB shell
inside the container.

Before doing anything meaningful we'll need to set up a secret. In this case
the token is ignored because of our local config, but the structure is
identical in a live production deployment. After creating the secret, you can
immediately attach and read:

```sql
CREATE SECRET (
    TYPE     unity_catalog,
    TOKEN    'demo-ignored-token',
    ENDPOINT 'http://unitycatalog:8080'
);

ATTACH 'unity' AS my_catalog (TYPE unity_catalog, DEFAULT_SCHEMA 'my_schema');

SELECT name, age, adopted FROM my_catalog.pets ORDER BY name;
-- -> single 'Seed' row
```

That's it — you just queried Unity-Catalog-managed, Delta-stored pets data.

Next, let's complete the circle and write some data into our pets table:

```sql
INSERT INTO my_catalog.pets (uuid, name, age, adopted)
SELECT
    gen_random_uuid()::VARCHAR,
    ['Luna', 'Milo', 'Bella', 'Charlie', 'Max', 'Lucy', 'Cooper', 'Daisy', 'Buddy', 'Lily',
     'Rocky', 'Molly', 'Bear', 'Lola', 'Duke', 'Sadie', 'Tucker', 'Zoe', 'Oliver', 'Stella'][1 + (random() * 19)::INT],
    (1 + (random() * 14)::INT)::INT,
    random() > 0.5
FROM range(10);

SELECT count() FROM my_catalog.pets;
```

You can also easily find and see the created files; check `data`
(bind-mounted in Docker), and you should find both existing files, and 1 new
parquet file containing the new rows. In my case it looks like this:

```batch
tree data
```

```text
data
└── external
    └── unity
        └── my_schema
            └── tables
                └── pets
                    ├── _delta_log
                    │   ├── 00000000000000000000.json
                    │   ├── 00000000000000000001.json
                    │   └── 00000000000000000002.json
                    ├── duckdb-19cb47ae-9f35-4126-b67d-c94fcade68cc.parquet
                    └── duckdb-e3bb0336-f16a-4d21-9495-0fbf55c6cba8.parquet

7 directories, 5 files
```

### Managing the Stream: Catalog Managed Commits

With the basics out of the way, we can talk about Catalog Managed Commits
(CMC). This is available today in both [OSS](https://www.unitycatalog.io/) and
[Databricks](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
Unity Catalog.

The headline feature is catalog-coordinated concurrent writes. Without CMC,
DuckDB writes go directly to the Delta log — and while modern storage backends
prevent outright lost writes, UC is left out of the loop entirely. Its
metadata, audit trail, and statistics fall out of sync with the actual table
state, and other engines querying through UC may see a stale view.

CMC fixes this: every write is staged and registered through UC before it
becomes visible. UC acts as the commit arbiter — a second writer arriving at
the same version receives a conflict error and can cleanly retry. This
matters wherever multiple DuckDB processes are appending simultaneously —
parallel ETL pipelines, partitioned bulk loads, concurrent analytical
inserts. Each writer works independently; UC ensures exactly one commit
lands per version and keeps its own catalog in sync with every one of them.

A few things worth noting: consistent reads and audit history are already
inherent to Delta and UC respectively — CMC doesn't add those, it just ensures
UC stays in sync with every commit. And CMC coordinates commits per table;
there is no cross-table atomicity. If you write to two tables in the same
`BEGIN`/`COMMIT` block, each table commits independently.

To opt a table into CMC, set the `delta.feature.catalogManaged` table property
at creation time. This is done via Spark or the UC CLI — DuckDB does not yet
support `CREATE TABLE` DDL:

```sql
-- Via Spark
CREATE TABLE my_catalog.my_schema.concurrent_tbl (
    uuid    STRING  NOT NULL,
    name    STRING  NOT NULL,
    age     INT     NOT NULL,
    adopted BOOLEAN NOT NULL
)
TBLPROPERTIES ('delta.feature.catalogManaged' = 'supported');
```

Once CMC-enabled, DuckDB writes go through UC's commit staging automatically —
the `INSERT` syntax is unchanged:

```sql
INSERT INTO my_catalog.my_schema.concurrent_tbl (uuid, name, age, adopted)
VALUES (gen_random_uuid()::VARCHAR, 'Luna', 3, true);
```

Now each DuckDB writer stages its commit to a `_staged_commits/` directory and
registers it with UC before that data becomes visible. UC arbitrates: exactly
one writer wins each version in a race, the rest get a conflict error and can
retry.

Let's put that to the test. We launched 20 competing DuckDB
writers, 8 at a time, all inserting into the same CMC table:

```bash
seq 1 20 | xargs -P 8 -I{} scripts/unity/05-cmc/write-single {}
[worker 6] OK — inserted 5 rows
[worker 5] CONFLICT — another writer won this version, retry needed
[worker 2] CONFLICT — another writer won this version, retry needed
[worker 8] CONFLICT — another writer won this version, retry needed
[worker 7] CONFLICT — another writer won this version, retry needed
[worker 3] CONFLICT — another writer won this version, retry needed
[worker 1] OK — inserted 5 rows
[worker 4] CONFLICT — another writer won this version, retry needed
[worker 16] OK — inserted 5 rows
[worker 13] CONFLICT — another writer won this version, retry needed
[worker 15] CONFLICT — another writer won this version, retry needed
[worker 11] CONFLICT — another writer won this version, retry needed
[worker 14] CONFLICT — another writer won this version, retry needed
[worker 12] OK — inserted 5 rows
[worker 9] CONFLICT — another writer won this version, retry needed
[worker 10] CONFLICT — another writer won this version, retry needed
[worker 17] CONFLICT — another writer won this version, retry needed
[worker 20] CONFLICT — another writer won this version, retry needed
[worker 18] OK — inserted 5 rows
[worker 19] CONFLICT — another writer won this version, retry needed
```

5 writers committed, and 15 got a clean conflict error to signal a retry. No
silent overwrites, no corruption — the table's row count confirms every
successful insert landed exactly once:

```sql
SELECT count() AS total_rows FROM my_catalog.my_schema.concurrent_tbl;
┌────────────┐
│ total_rows │
│   int64    │
├────────────┤
│         35 │
└────────────┘
```

10 seeded rows + (5 writes × 5 rows each) = 35 total rows. Successful
concurrent writes. In a real workload, you would retry the
conflicted writes and land all 20 inserts.

## Conclusions

A year ago, DuckDB could read Delta tables. Today it can insert data into them,
travel through their history, and query and write through a governed catalog —
without the experimental caveat on any of it. The combination of Delta for open
storage, Unity Catalog for governance and coordination, and DuckDB for fast
analytical queries is a stack you can build on.

There's more to come: DDL support to create and manage tables directly,
delete/update/merge support, and multi-table atomicity for writes that span
more than one table. In the meantime, the playground image linked above has
everything you need to kick the tires — and as always, feedback and bug reports
are welcome on GitHub.

---
