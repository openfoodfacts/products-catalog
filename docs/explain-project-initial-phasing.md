# Explanation on project initial phasing

The products catalog service is a service
that will take the products storage
out of [Product Opener](https://github.com/openfoodfacts/openfoodfacts-server)

As Open Food Facts has a very high workload and many people depending on it
the transition must be as smooth and transparent as possible.

Also there are risks on moving so much data
(unexpected pitfalls, performances problems, etc.),
so the phasing is very important.

This document explains how the migration from the old system,
to this new service will take place.

It is based upon [initial reflections written down by John on the wiki](https://wiki.openfoodfacts.org/Project:Product_Service#Phasing)

## General strategy

The idea is that Products Catalog will be plugged to Product Opener
as soon as possible and will be the backend for all products storage / retrieval.

Most Product storage / retrieval happens at one place in the code
(dedicated functions),
so it's transparent to the rest of Product Opener,
but for some specific scripts (that must be adapted in time).

But the data storage itself will not change immediately,
it will still be a file based data storage, but served by Products Catalog.

Products Catalog will then handle the progressive migration of products data
to MariaDB,
keeping the same API seen from Product Opener.

## Phase A - keeping file storage but switching to the new service

### Phase A.1 - Use JSON files instead of STO files

STO files are readable from Perl only.
Storing products revision files in JSON instead of STO has the advantage
that those files are readable by a very large number of languages,
in particular Python,
which is the language Products Catalog is developed in
(see [project architecture](./explain-architecture.md)).

After verification of impact on performance
(how much slower is JSON serialization compared to STO serialization)

Product Opener is changed to store data in JSON instead of STO files.

This transformation is phased:
* phase A.1: write JSON + STO for new information
  but retrieve from JSON when it exists.
  This validate everything goes fine with a possible rollback to STO only.
* phase A.2: write only JSON for new information,
  retrieve from JSON when it exists,
  STO otherwise
* phase A.3: migrate the old STO files to JSON (in parallel to production),
  removing STO files along the way

### Phase A.2 - Use Products Catalog to write JSON

At this phase, we will replace current JSON/STO file access in Product Opener,
with calls to the Products Catalog.

Products Catalog will already offer
the base of the target API for Products Data update products
but the implementation will work with the existing JSON files.

The advantage of this approach is that we can identify
the additional performance impact of adding a web service
without introducing other changes,
like storing the data in a database.

It also allows us to revert back to the current approach
without the need for any re-synchronising of data.

* Create the APIs that product opener will use to update products.
* These API will:
  * write back to the JSON files (product data and changes)
  * generate the product_update events
  * update MongoDB products and recent changes collections

Implementing write before read should be quicker to implement
as writes are less common
and will allow us to get early feedback on
the impact of the changes on mass updates

### Phase A.3 - Use Products Catalog to read and write JSON

In the continuity of the logic of Phase A.2,
now products and changes reads
inside Product Opener are delegated to Products Catalog.

* Add read APIs for Product Opener
  to use instead of reading from the files directly.
* Also handles the Changes read

### Managing phase A

Phase 2 to 3 are guarded by a configuration switch (feature flag)
in Product Opener
(that is the same code can run A.1, A.2 or A.3
only based upon a configuration change).
implementation levels correspond to the phases:

* A.1 Existing read and write directly to the JSON files
* A.2 Write using the API but still read direct from JSON files
* A.3 Read and write using the API

This enables to easily go back and forth.
Also this enables first switching on the instances with less traffic and reuse
(and less risk)
like Open Pet Food Facts, Open Products Facts and Open Beauty Facts.

## Phase B - migration to the Database

The next step will introduce the MariaDB backend for storing product data.

### Phase B.1 - migrating the data to MariaDB

During this phase, the Products Catalog service, still works with JSON file,
but new revisions are also written in the database.

At the same time, scripts are run in parallel to port old products revisions to the database.

It may require to alleviate some checks during this period
(eg: adding an exception on revision number check,
if previous revisions are not yet written to the database)

At the end of Phase B.1, the database is a mirror of JSON files,
constantly updating.

The end of phase B.1 is also a time to verify writes performance in MariaDB
are at the expected level.

### Phase B.2 - adding replication

While keeping the same behaviour for Products Catalog as in phase B.1,
as the database is now complete,
it's the right time to deploy replications of the MariaDB database.

It enables verifying everything works well,
measuring the latency before update
and verifying how well it can support parallel requests (using tests scripts)

### Phase B.3 - using MariaDB but keeping storage synced

In phase B.3, the MariaDB writes become primary candidates,
and, more important, products read happens on the database.

we keep JSON files in sync, but we do it in an async mode,
possibly using a side process to Products Catalog.
This offers a possible rollback if things don't go as expected
(be it for a temporary interval until some issues are resolved).
JSON file sync is guarded by a feature flag so that we can drop it later on.

We also keep updating MongoDB.

At this point Products Catalog is almost working as it should.

## Phase C - Products Catalog as primary storage service

Note: phase C is out of Hyper Open X project scope
(which fund initial work on the project).

But it is an important part to avoid technical debt
and leverage Products Catalog full benefits.

### Phase C.1 - more clients for Products Catalog

In this phase,
we can then identify other processes
that are reading data from the file system and / or MongoDB
and update these to call the Products Catalog directly.

Already identified targets include:

* database exports
* partial exports for development
* openfoodfacts-query and search-a-licious calls
  to fetch product information when a product_update event is received
* robotoff calls to process products to find insights

At this point, it might be also ok to have MongoDB sync
done in asynchronous mode.
This could enhance Products Catalog performance.

### Phase C.2 - replacing MongoDB

In this phase (which can happen concurrently to C.1),
we identify parts of the code (mainly in Product Opener)
that still use MongoDB, and try to fill the gap
* either by using Products Catalog (mostly for batch products iterations)
* or by using openfoodfacts-query
  (requests for products based on simple criteria)
* or by using search-a-licious (in particular the v0/v2 search API)

This step will require
* to update openfoodfacts-query to support all product types (currently only supports food)
* deploy search-a-licious for all product types (currently only food is deployed)

This might require some enhancements of Products Catalog API.

### Phase C.3 - dropping JSON files

Phase C.4 is the point when
we feel confident enough in the new storage
and have migrated every script that might rely on JSON files.
At this point we can drop JSON file sync.

With it we can drop all the JSON files.

### Phase C.4 - dropping MongoDB

It is the point where no useful piece of code is using MongoDB any more.
We might just stop updating it, and remove it.


## Beyond - enhancing Products Catalog

This can happen at any time in parallel of phase B.3 and C,
and is not really part of migration plan.

It encompasses enhancements to Products Catalog
to be more useful and robust.
* Schema validation
* Schema migrations
* clients authentication and permissions
* Data Exports pinned to a schema version
* Allowing updates from clients other than Product Opener
* eventually extend the service to directly handle some read APIs:
  * Knowledge Panels
  * Taxonomy lookup,
  * etc.


