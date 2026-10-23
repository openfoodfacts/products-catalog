# Explanation of SQL database schema

## MariaDB

We use MariaDB with **InnoDB** storage engine.

## Product tables

The product catalog is divided in three tables:
* the table with the latest revisions: `product`
* the table with the history: `product_revision`
* the table with the changes: `change`

`product` and `product_revision` have the same fields.
The `change` table has no current/history split:
it stores the changes of every revision in a single table,
keyed by `(code, rev)`.

Having latest revisions in `product` is sensible because
most of the requests we expect will be to query latest revision.

**QUESTION:** should latest revision in `product`
and also in `product_revision` at the cost of an overhead ?
This would simplify request on full history at the cost of a bigger database
(mean revision number in database seems to be around 10, so it's about 10% increase).
**Note:** the rest of this document currently assuming
we only have current version in `product`


### The product table fields

`product` and `product_revision` contains following fields:

* `code` - is product code (varchar)
* `product_type` - is the product type (at this revision) mapped to a tinyint
* `rev` - is the revision number (int)
* `lifecycle` - state of the product at this revision
  * it is either "deleted", "active" or "obsolete" represented by a TINYINT (0, 1, 2)
* `last_modified_t` - the date the revision was created
   (corresponding to `last_modified_t` in the product data)
* `last_updated_t` - the date of update,
   corresponding to `last_updated_t` in the product data
  (different from `last_modified_t` in case of in-place update)
* `data` - is the product data as JSON (LONGTEXT + JSON_VALID check)
* `data_schema_version` - is the schema version number for data.
  It's important to know what to expect,
  (and to eventually conduct in place migrations, see [Migrating schemas](#migrating-schemas))
* `rev_archived_t` - the time the revision was archived,
  that is, was replaced by a newer revision.
  In `product` table `rev_archived_t`
  is always the same and it is the maximum date that can be set.



In `product`, the natural key is `code`.
In `product_revision`, the natural key of this table is `(code, rev)`.

`lifecycle` defaults to 1,
so that if it's accidentally forgotten it does not result in a silent deletion.

<a id="schema-version"></a>
`data` and `changes` schemas are under the control of Product Opener,
but `data_schema_version` (in `product` and `product_revision`)
and `changes_schema_version` (in `change`) help have an indication
on their schema version.

`data` contains the full product information, `code`, `product_type`, `rev`, `lifecycle` columns are duplicate of the information in `data`.

The `rev_archived_t` enables partitioning the `product_revision` table
(see [Table partitioning](#table-partitioning), below).
It also helps querying  run a query like "get me all products that were active on xxx date".

### The change table fields

`change` contains the changes associated with each product revision:

* `code` - is product code (varchar)
* `rev` - is the revision number (int)
* `changes` - is a JSON containing data about the change (LONGTEXT + JSON_VALID check)
* `changes_schema_version` - is the schema version number for changes
  (see [the explanation above for `data_schema_version`](#schema-version))

The natural key of this table is `(code, rev)`.
There is no current/history split for changes:
`change` stores one row per product revision,
including the latest revision.

### Indexes

We will start with minimal useful indexes:

On `product` and `product_revision`:
  * `code`
    * note: on `product` it's the (primary key)
    * on `product_revision` it helps fetch all revisions for a single product
  * `last_modified_t`
  * `last_updated_t`

on `product`:
  * `(product_type,lifecycle, code)`
    * to iterate products filtering by product_type and lifecycle

on `product_revision`:
  * `(code,rev)` (primary key)

on `change`:
  * `(code,rev)` (primary key)

After running performances tests, and developing the API,
we might add more indexes if needed.

### Other settings

In `product` and `product_revision`
the `data` field must be compressed.
It's a one time write, many read,
and this will save disk space and memory footprint.

## Operations

Some guidelines for operations on the database.

### Inserting a product revision

If it's a new product it just lands in `product`,
and its changes are added to the `change` table,
keyed by `(code, rev)`.

If it's a revision of an existing product (be it deleted or obsolete),
it must:
* copy last revision data to the `product_revision` table
  (its changes are already in the `change` table)
* update the `product` row for this `code` with the new values
* add the changes for this revision to the `change` table,
  keyed by `(code, rev)`


### Deleting a product

Deleting a product inserts a new revision with lifecycle set to the right value.

The `data` field might not be empty
as it contains information like revision, etc. (metadata)
But it should normally be quite small (it's up to the service to ensure that).

### Hard deleting a product

Product Opener is completely removing products
when the information is almost empty and we have no previous revision.

This correspond to a real removal of a line in `product` table.
But we must ensure `rev` is `1`.

### About in place updates

In place updates happens when Product Opener re-process a product.
That is, it should never touch the product primary (non inferred) data,
but might touch the product inferred data
(*_tags, ingredient analysis, scores, etc.)

Also it should not touch the `changes` field in the `change` table.

### Find a product revision / iterate on revisions

Beware of using `UNION ALL` or a transaction
when searching for product revisions
that might either be in `product` or `product_revision`
to ensure consistency of results
([keep a consistent MVCC snapshot](https://mariadb.com/docs/server/ha-and-performance/standard-replication/enhancements-for-start-transaction-with-consistent-snapshot#the-innodb-read-view),
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
  and follow in time (`last_modified_t` is always increasing)
  (if a revision is problematic for some reason,
  keep it but remove problematic data).
* latest revision is in `product`

A way to check that is to have:
`product.rev = 1 + COALESCE(MAX(product_revision.rev), 0) … WHERE product_revision.code = product.code`

The best way to do that, though, is to use a transaction,
immediately lock the row for update,
and add one to current revision.

### Time consistency

All timestamps must be UTC (it's up to the API to decide if it only accept UTC),
the schema definition of the database and query design
might help handle this smoothly.

`last_modified_t`, `last_updated_t` and `rev_archived_t` must come from the database clock
(eg `NOW(4)` in the insert statement)

### Tables are not redundant

`product` can't be rebuilt from `product_revision`,
and changes are stored separately in `change`,
so back-ups must include all three tables.


### Migrating schemas

This is an open question.

Should we migrate old revisions schema to newer schema versions ?

This seems a good idea at least for products in `product` table,
as data exports will lean on that,
and also it would avoid version transformation on Product Opener side
and enable tools to read directly the catalog
(eg. robotoff / search-a-licious / openfoodfacts-query ),
making them more efficient (and saving Product Opener queries).

This is also too be considered for `product_revision` and `change` tables.
This would ease re-users working with old version to have a consistent schema,
and there again avoid Product Opener conversions.
This might happen at a lower pace than for `product` table.
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

The distribution over multiple storage could be of use for `product_revision`,
but it really depends on usage and options.

* cons:
  * at the moment we expect a db size < 2T,
    which is bearable for a reasonable SSD size
  * the `product` table can already be assigned to a different storage
    that `product_revision`
  * the partitioning key is not evident:
    * if we partition on: `last_modified_t` which seems a natural choice
      * if people needs historical view of current products,
        `last_modified_t < SOME_POINT_IN_TIME` can leverage partitioning
        but we still have to check
        all partitions with `last_modified_t` before this point in time,
        because some products might have
        their last version created a long time ago.
        So for recent dates, optimization is really marginal.
      * if we want old partition to be frozen,
        we need to duplicate current product revision in `product_revision`
        (which might be reasonable though, see above)
    * `product_type` could also be a discriminant, but currently it would be marginal,
      as 90% is product_type food. Also `product_type` can change for a product.
    * hashing on `code` could also be a way to partition,
      * same products revisions would
      * the drawback is that partitions all grows over time (less predictable behaviour)
  * Partitioning impose several limitations in MariaDB see [Partitioning Limitations](https://mariadb.com/docs/server/server-usage/partitioning-tables/partitioning-limitations)
    The most important to consider here:

    * > All columns used in the partitioning expression for a partitioned table must be part of every unique key that the table may have

      If we don't plan to use `code` or `rev` to partition,
      this adds a new column to the table key.
      It does not prevent from having a `UNIQUE` constraint on `code` and `rev`

    * > A partitioned table cannot contain, or be referenced by, foreign keys.

      As we have no foreign key at this point,
      it does not seems to have a big impact

* pros:
  * if at some point we want to partition `product_revision`
    (because it's too big to fit a single disk) it is a very expensive operation,
    (maybe several days of read only operations)
    because it's equivalent to a full table rewrite.
    So doing it in anticipation seems a fair choice.
  * we have a `rev_archived_t` field
    (the date at which the revision was no more the last)
    we can partition on that.
    * In this case, we don't need to duplicate current `product` revision
      in `product_revision`
    * if people needs historical view of current products,
      they need to get lines for `product` where `last_modified_t < SOME_POINT_IN_TIME`
      and lines for `product_revision`
      where `achived_at > SOME_POINT_IN_TIME`
      (and `last_modified_t ⩽  SOME_POINT_IN_TIME`)
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
that only replicate the `product` table.

But beware that there are some drawbacks:
* consistency is not ensured
  because statements with multiples tables
  are not necessarily replicated, so there can be a drift, it's brittle.
  In particular in our case, most updates
  will use `product`, `product_revision` and `change` at the same time
* the target instance still receive the whole replication stream,
  and filtering happens at this end of the network,
  so it does not save network bandwidth.

