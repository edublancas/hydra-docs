---
description: >-
  The Hydra columnar table engine gives Postgres the ability to increase
  performance for append-only datasets.
---

# 📊 Columnar

Hydra offers a columnar table engine for Postgres in addition to Postgres' built-in (default) Heap tables. Heap tables are ideal for row-based access, where individual pieces of data is found in the heap by using an index. Columnar is ideal for analytical and historical data that will not change.

The columnar engine stores data in stripes of up to 150,000 rows of data. Each stripe of data stores data from each column together. This makes it considerably faster to read all data from a single column. Stripes are created during each transaction, so data needs to be inserted in bulk, rather than one row at a time.

Data in columnar tables is append-only; it cannot be updated or deleted. One pattern is to move data to columnar tables once it has been finalized. Alternatively, columnar tables can be periodically rebuilt in order to update them.

Data in columnar tables are compressed, which can assist in reducing your long-term storage requirements.

Columnar is available on all Hydra plans.

## Using a columnar table

Create a Columnar table by specifying `USING columnar` when creating the table.

```sql
CREATE TABLE my_columnar_table
(
    id INT,
    i1 INT,
    i2 INT8,
    n NUMERIC,
    t TEXT
) USING columnar;
```

Insert data into the table and read from it like normal (subject to the limitations listed below). Note that columnar supports only btree and hash indexes and their associated constraints.

## Limitations

* Append-only (no `UPDATE`/`DELETE` support)
* No space reclamation (e.g. rolled-back transactions may still consume disk space)
* No support for `gist`, `gin`, `spgist`, and `brin` indexes.
* No bitmap index scans
* No tidscans
* No sample scans
* No TOAST support (large values supported inline)
* No support for [`ON CONFLICT`](https://www.postgresql.org/docs/12/sql-insert.html#SQL-ON-CONFLICT) statements (except `DO NOTHING` actions with no target specified).
* No support for tuple locks (`SELECT ... FOR SHARE`, `SELECT ... FOR UPDATE`)
* No support for serializable isolation level
* No support for foreign keys, unique constraints, or exclusion constraints
* No support for logical decoding
* No support for intra-node parallel scans
* No support for `AFTER ... FOR EACH ROW` triggers
* No `UNLOGGED` columnar tables

## Converting From Row to Columnar

Hydra has a convenience function that will copy your row table to columnar.

```sql
CREATE TABLE my_table (i INT8);
-- convert to columnar
SELECT alter_table_set_access_method('my_table', 'columnar');
```

Data can also be converted manually by copying. For instance:

```sql
CREATE TABLE table_heap (i INT8);
CREATE TABLE table_columnar (LIKE table_heap) USING columnar;
INSERT INTO table_columnar SELECT * FROM table_heap;
```

## Partitioning

Columnar tables can be used as partitions; and a partitioned table may be made up of any combination of row and columnar partitions. You can use this feature to have archived data from previous months or years stored in columnar tables while active data is added to a heap table.

```sql
CREATE TABLE parent(ts timestamptz, i int, n numeric, s text)
  PARTITION BY RANGE (ts);

-- columnar partition
CREATE TABLE p0 PARTITION OF parent
  FOR VALUES FROM ('2020-01-01') TO ('2020-02-01')
  USING COLUMNAR;
-- columnar partition
CREATE TABLE p1 PARTITION OF parent
  FOR VALUES FROM ('2020-02-01') TO ('2020-03-01')
  USING COLUMNAR;
-- row partition
CREATE TABLE p2 PARTITION OF parent
  FOR VALUES FROM ('2020-03-01') TO ('2020-04-01');

INSERT INTO parent VALUES ('2020-01-15', 10, 100, 'one thousand'); -- columnar
INSERT INTO parent VALUES ('2020-02-15', 20, 200, 'two thousand'); -- columnar
INSERT INTO parent VALUES ('2020-03-15', 30, 300, 'three thousand'); -- row
```

When performing operations on a partitioned table with a mix of row and columnar partitions, take note of the following behaviors for operations that are supported on row tables but not columnar (e.g. `UPDATE`, `DELETE`, tuple locks, etc.):

* If the operation is targeted at a specific row partition (e.g. `UPDATE p2 SET i = i + 1`), it will succeed; if targeted at a specified columnar partition (e.g. `UPDATE p1 SET i = i + 1`), it will fail.
* If the operation is targeted at the partitioned table and has a `WHERE` clause that excludes all columnar partitions (e.g. `UPDATE parent SET i = i + 1 WHERE ts = '2020-03-15'`), it will succeed.
* If the operation is targeted at the partitioned table, but does not exclude all columnar partitions, it will fail; even if the actual data to be updated only affects row tables (e.g. `UPDATE parent SET i = i + 1 WHERE n = 300`).

Note that the columnar engine supports `btree` and `hash` indexes (and the constraints requiring them) but does not support `gist`, `gin`, `spgist` and `brin` indexes. For this reason, if some partitions are columnar and if the index is not supported by columnar, then it's impossible to create indexes on the partitioned (parent) table directly. In that case, you need to create the index on the individual row partitions. Similarly for the constraints that require indexes, e.g.:

```sql
CREATE INDEX p2_ts_idx ON p2 (ts);
CREATE UNIQUE INDEX p2_i_unique ON p2 (i);
ALTER TABLE p2 ADD UNIQUE (n);
```

## Options

Set options using:

```sql
alter_columnar_table_set(
    relid REGCLASS,
    chunk_group_row_limit INT4 DEFAULT NULL,
    stripe_row_limit INT4 DEFAULT NULL,
    compression NAME DEFAULT NULL,
    compression_level INT4)
```

For example:

```sql
SELECT alter_columnar_table_set(
    'my_columnar_table',
    compression => 'none',
    stripe_row_limit => 10000);
```

The following options are available:

* **compression**: `[none|pglz|zstd|lz4|lz4hc]` - set the compression type for _newly-inserted_ data. Existing data will not be recompressed/decompressed. The default value is `zstd`.
* **compression\_level**: `<integer>` - Sets compression level. Valid settings are from 1 through 19. If the compression method does not support the level chosen, the closest level will be selected instead.
* **stripe\_row\_limit**: `<integer>` - the maximum number of rows per stripe for _newly-inserted_ data. Existing stripes of data will not be changed and may have more rows than this maximum value. The default value is `150000`.
* **chunk\_group\_row\_limit**: `<integer>` - the maximum number of rows per chunk for _newly-inserted_ data. Existing chunks of data will not be changed and may have more rows than this maximum value. The default value is `10000`.

View options for all tables with:

```sql
SELECT * FROM columnar.options;
```

You can also adjust options with a `SET` command of one of the following configuration variables:

* `columnar.compression`
* `columnar.compression_level`
* `columnar.stripe_row_limit`
* `columnar.chunk_group_row_limit`

These settings only affect newly-created tables, not any newly-created stripes on an existing table.

## Source Code

Hydra's columnar engine is a fork of the Citus columnar access method. [The source code is available on GitHub](https://github.com/hydradatabase/hydra/tree/main/columnar).

## Additional information

* [Archiving with columnar storage (microsoft.com)](https://docs.citusdata.com/en/stable/use\_cases/timeseries.html#archiving-with-columnar-storage)
