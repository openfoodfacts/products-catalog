# Explanation of SQL database schema

## MariaDB

We use MariaDB with **InnoDB** storage engine.

## Products tables

The products table is divided in two tables:
* the table with the latest revisions: `products`
* the table with the history: `products_revisions`

Both tables have the same fields.

Having latest revisions in `products` is sensible because
most of the requests we expect will be to query latest revision.

**QUESTION:** should latest revision in `products`
and also in `products_revisions` at the cost of an overhead ?
This would simplify request on full history at the cost of a bigger database
(mean revision number in database seems to be around 10, so it's about 10% increase).

### The Products table fields

`products` and `products_revisions` contains following fields:

* `code` - is product code (varchar)
* `product_type` - is the product type (at this revision) mapped to a tinyint
* `revision` - is the revision number (int)
* `lifecycle` - state of the product at this revision
  * it is either "deleted", "active" or "obsolete" represented by a TINYINT (0, 1, 2)
* `created` - the date the revision was created
* `updated` - the date of update (different from created in case of in-place update)
* `changes` - is a JSON containing data about the change (LONGTEXT + JSON_VALID check)
* `data` - is the product data as JSON (LONGTEXT + JSON_VALID check)
* `data_schema_version` - is the schema version number for data.
  It's important to know what to expect,
  (and to eventually conduct in place migrations, see [Migrating schemas](#migrating-schemas))
* `changes_schema_version` - is the schema version number for changes.
* `data_hash` - a sha256 sum some data (see below, meaning given by service)

In products, the natural key is `code`.
In products_revisions, the natural key of this table is `(code, revision)`.

`lifecycle` defaults to 1,
so that if it's accidentally forgotten it does not result in a silent deletion.

`data` and `changes` schemas are under the control of Product Opener,
but `data_schema_version` and `changes_schema_version` help have an indication
on their schema version.

`data` contains the full product information, `code`, `product_type`, `revision`, `lifecycle` columns are duplicate of the information in `data`.

`data_hash` is a column that can help verify if a modification is a replay of a previous update (duplicate POST), or a blank modification.
The md5sum is computed on a canonicalized json (details at implementation time),
maybe with primary (non inferred) data + changes,
with eventual removal of canonical data that interfere (timestamps, minor computed fields)…
It's up to the service to determine the logic
(and for example if it wants to allow "empty modifications")

**QUESTION:** is this data_hash really desirable ?
**QUESTION:** Is this up to the Product Catalog service to compute this hash,
or maybe Product Opener (it's business driven).
The latter option might be a problem
if other services can update the data directly through Products Catalog.

**QUESTION:** an `archived_at` field
correponding to the time the product revision was overridden by a new revision
in `products_revisions` might make sense.
See discussion in [Table partitioning](#table-partitioning) below.

### Indexes

We will start with minimal useful indexes:

On both tables:
  * `code`
    * note: on `products` it's the (primary key)
    * on `products_revisions` it helps fetch all revisions for a single product
  * `created`
  * `updated`

on `products`:
  * `(product_type,lifecycle, code)`
    * to iterate products filtering by product_type and lifecycle

on `products_revisions`:
  * `(code,revision)` (primary key)

After running performances tests, and developing the API,
we might add more indexes if needed.

### Other settings

In `products_revisions` the `data` field must be compressed.
It's a one time write, many read,
and this will save disk space and memory footprint.

## Operations

Some guidelines for operations on the database.

### Inserting a product revision

If it's a new product it just lands in `products`.

If it's a revision of an existing product (be it deleted or obsolete),
it must:
* copy last revision to `products_revisions` table
* update the `products` row for this `code` with the new values


### Deleting a product

Deleting a product inserts a new revision with lifecycle set to the right value.

The `data` field might not be empty
as it contains information like revision, etc. (metadata)
But it should normally be quite small (it's up to the service to ensure that).

### Hard deleting a product

Product Opener is completely removing products
when the information is almost empty and we have no previous revision.

This correspond to a real removal of a line in `products` table.
But we must ensure revision is `1`.

### About in place updates

In place updates happens when Product Opener re-process a product.
That is, it should never touch the product primary (non inferred) data,
but might touch the product inferred data
(*_tags, ingredient analysis, scores, etc.)

Also it should not touch `changes` field.

### Find a product revision / iterate on revisions

Beware of using `UNION ALL` when searching for product revisions
that might either be in `products` or `products_revisions`
to ensure consistency of results
([keep a consistent MVCC snapshot](https://mariadb.com/docs/server/server-usage/storage-engines/tidesql/tidesql-transactions-and-isolation),
avoid race condition with an update).

### Fetching a product revision history

Beware that a product, `product_type` might change between revisions.
Your query must cope with it.

## Invariants and consistency

### data_hash idempotency

If an update comes with same data_hash as current revision,
it must end with a `conflict` instead of creating a revision
(eg. http code 409).

### Revision consistency

It's up to the Product catalog service to ensure consistency in the revision list.
The concurrency models is part of its definition.

It must ensure that
* revisions for a single product start at 1, are consecutive,
  and follow in time (`created` is always increasing)
  (if a revision is problematic for some reason,
  keep it but remove problematic data).
* latest revision is in `products`

A way to check that is to have:
`products.revision = 1 + COALESCE(MAX(products_revisions.revision), 0) … WHERE products_revisions.code = products.code`

### Time consistency

All timestamps must be UTC (it's up to the API to decide if it only accept UTC),
the schema definition of the database and query design
might help handle this smoothly.

`created`, `updated` and `archived_at` must come from the database clock
(eg `NOW(4)` in the insert statement)

### Tables are not redundant

`products` can't be rebuilt from `products_revisions`
so back-ups must include both tables.


### Migrating schemas

This is an open question.

Should we migrate old revisions schema to newer schema versions ?

This seems a good idea at least for products in `products` table,
as data exports will lean on that,
and also it would avoid version transformation on Product Opener side
and enable tools to read directly the catalog
(eg. robotoff / search-a-licious / openfoodfacts-query ),
making them more efficient (and saving Product Opener queries).

This is also too be considered for `products_revisions` table.
This would ease re-users working with old version to have a consistent schema,
and there again avoid Product Opener conversions.
This might happen at a lower pace than for `products` table.
Non consistent revisions that Product Opener can't convert
could be left has is.

However, this is a decision left to Product Opener.
The API should allow modifying historical data schema,
but is kept agnostic of it (for now).

## Table partitioning

The main benefits in partitioning with MariaDB (see [partitioning overview](https://mariadb.com/docs/server/server-usage/partitioning-tables/partitioning-overview))
(which is quite different from PostgresSQL in this point of view)
would be:
1. optimizing queries on subsets
2. distributing over multiple storage
3. smarter backups

(Also it could be for easy removal of historical data, but it's irrelevant here as we never really remove data)

The point 1 is not that relevant because our table as few indexed fields.
So choosing good index should be enough.

The distribution over multiple storage could be of use for `products_revisions`,
but it really depends on usage and options.

* cons:
  * at the moment we expect a db size < 2T,
    which is bearable for a reasonable SSD size
  * the `products` table can already be assigned to a different storage
    that `products_revisions`
  * the partitioning key is not evident:
    * if we partition on: `created` which seems a natural choice
      * if people needs historical view of current products,
        `created < SOME_POINT_IN_TIME` can leverage partitioning
        but we still have to check
        all partitions with `created` before this point in time,
        because some products might have
        their last version created a long time ago.
        So for recent dates, optimization is really marginal.
      * if we want old partition to be frozen,
        we need to duplicate current product revision in `products_revisions`
        (which might be reasonable though, see above)
    * `product_type` could also be a discriminant, but currently it would be marginal,
      as 90% is product_type food. Also `product_type` can change for a product.
    * hashing on `code` could also be a way to partition,
      * same products revisions would 
      * the drawback is that partitions all grows over time (less predictable behaviour)
  * Partitioning impose several limitations in MariaDB see [Partitioning Limitations](https://mariadb.com/docs/server/server-usage/partitioning-tables/partitioning-limitations)
    The most important to consider here:

    * > All columns used in the partitioning expression for a partitioned table must be part of every unique key that the table may have

      If we don't plan to use `code` or `revision` to partition,
      this adds a new column to the table key.
      It does not prevent from having a `UNIQUE` constraint on `code` and `revision`

    * > A partitioned table cannot contain, or be referenced by, foreign keys.

      As we have no foreign key at this point,
      it does not seems to have a big impact

* pros:
  * if at some point we want to partition `products_revisions`
    (because it's too big to fit a single disk) it is a very expensive operation,
    (maybe several days of read only operations)
    because it's equivalent to a full table rewrite.
    So doing it in anticipation seems a fair choice.
  * we could add a `archived_at` field
    (the date at which the revision was no more the last)
    and partition on that.
    * In this case, we don't need to duplicate current `products` revision
      in `products_revisions`
    * if people needs historical view of current products,
      they need to get lines for `products` where `created < SOME_POINT_IN_TIME`
      and lines for `products_revisions`
      where `achived_at > SOME_POINT_IN_TIME`
      (and `created ⩽  SOME_POINT_IN_TIME`)
      which can leverage partitioning,
      with better performances when `SOME_POINT_IN_TIME` is more recent.
    * We can place tables partitions with older partitions on HDD
      and newer partitions on SSD
    * partitioning this way enables inserts to go into the newest partition,
      giving a more sequential insert at the right edge instead of random page

**FIXME:** decide what to do
  (Alex: I really lean toward partitioning on `archived_at` by ~year since 2018)

## Some database considerations

### InnoDB Engine

MyIsam is not ACID, Aria is not yet ACID.
RockDB is for high number of writes (which is not the case here).

InnoDB is a good choice here: it's ACID,
it supports partitioning and media repartition and
it's the most widely used engine (conservative choice).

### Partial replication

MariaDB offers a way to replicate only chosen table to another instance.

It seems tempting to have read only MariaDB
that only replicate the `products` table.

But beware that there are some drawbacks:
* consistency is not ensured
  because statements with multiples tables
  are not necessarily replicated, so there can be a drift, it's brittle.
  In particular in our case, most updates
  will use `products` and `products_revisions` at the same time
* the target instance still receive the whole replication stream,
  and filtering happens at this end of the network,
  so it does not save network bandwidth.

