# Explain project scope

Note: this document is essentially a rewrite of https://wiki.openfoodfacts.org/Project:Product_Service written by John G.

## Where it starts from

Initially, product opener was a monolithic application.

The storage was simply serializing the hashmap representing a product into a STO file.
- One directory per product code,
- one file for each revision,
- a `changes.sto` file with a hasmap tracking revisions,
- and a `current.sto` file that was a symlink to the latest revision.

Later on we migrated from the `sto` format
to the more cross-language `json` format.

While it has the advantage of simplicity,
this mechanism suffers from several drawbacks.

The main drawbacks are:
* more difficult to scale horizontally. While there exists solutions to share files on different servers, they are either a bit complex to setup (Ceph), can have bigger latency (NFS) and lack features other databases offers
* no real concurrent edits handling (a lock was used but only at time of writing the sto file, so concurrent edit could happen and overwrite the last sto, or simply don't take into account last revisions modifications: the sto file exists but the information of second to last might have been skipped in the last one)
* no indexation at storage level, so some queries are very slow
* it's a less common architecture so a lot of tools are not available

## Main goal of the project

### Product Storage, Retrieval and Versioning

Is the core feature for the service.

### Products types

Open Food Facts merged the database of it's different products types
(food, beauty, pet food, products) a while ago.
This decisions enable less friction when product type changes
(which is frequent because of users using the wrong type)
and also helps finding the product matching a barcode
whatever its type.
The products-catalog also contains all products in one place.

In the future we might partition the underlying storage by product type,
(if there are enough advantages in doing it)
but the service would still serve all products types.

### Concurrent edits

Basic clash detection is provided for updates,
where the caller can provide the expected current revision number when providing a product update.

### Queries and iterations

 The Product Service supports fetching one or more products by code
 (and revision for historical versions)
 and iterating over all products, by product type.

### Projection

Retrieval of product data support projection,
i.e. allowing the client to specify which fields they want.

This makes the payload more efficient.

Field groups might be introduced for convenience, such as "product for list", "product for read", "product for update".

Frequency of use of different projections could inform future optimisations to the storage structure, such as keeping different parts of the product object in different fields / tables.

### Security

The Product Service should require authentication for update operations
but be open for read operations.
Initially, only certain clients, i.e. Product Opener,
would have permission to write data on a user's behalf.

In the future, the security model could be enhanced
so that certain OIDC clients have permissions to update only a subset of the data model.
For example, any contributor could update core data derived from packaging,
whereas only the "Environmental Analysis" service (currently Product Opener)
could update data relating to the Green Score.

### Images data

Products data contains information about their corresponding images
that will initially be part of this service
(but in an agnostic way).

The images themselves are stored in a different location / service.

In the long term, images management could be part of a specific service.

### Validation

We intend to eventually apply basic JSON schema validation.

Initially some branches of the schema would support "any" value
until the data is cleaned up.

The JSON validation might also not be mandatory to start with.

### Changes

The information previously contained in `changes.json` (for each product)
is handled by this service.

### Events stream

This service is responsible for publishing
product changes events on the Redis stream.

This stream is used by other services to watch for products changes.
For example openfoodfacts-query and search-a-licious use it to update their indexes,
while robotoff use it to reprocess products to find new insights.

Product Opener will still be responsible to publish events unrelated to products.

### Quality of service

Concerning read and iterations in particular,
the service should be able to handle a quality of service mechanism.

## Currently out of scope (but may change in the future)

### Taxonomies

The Product Service would not be the primary source of truth for Taxonomies,
In the future, it could store them to allow taxonomy lookups to be performed on-the-fly during queries,
e.g. to return the category name of a product in the user's specified language.
This is not in the initial scope.

### Knowledge Panels

Augmenting raw data with Knowledge Panels
is outside of the scope of the raw Product Service

In the future it could change,
as there may be some benefit in considering pre-rendering of Knowledge Panel data
and storing this alongside the Product
for quick retrieval with minimal processing. (eg. only translations)

### Schema Migrations

Product Opener is responsible for schema migrations
(downgrading / upgrading)
as it currently involves a lot of business logic.

### Removing MongoDB from Product Opener

In production Product Opener is using MongoDB
where it stores the last version
of every products of the instance main product type.

MongoDB is used by Product Opener for several tasks:
1. product iteration
2. quick product retrieval (or sequence of products)
3. searches
4. facets (on the projects where openfoodfacts-query is not deployed)
5. maybe other topics
6. It is also used by Robotoff for quick objects retrieval

This service should enable getting rid of MongoDB for
(1) products iteration and (2) and (6) quick product retrieval
(or sequence of products).

The point (3) should be removed by search-a-licious
(and providing an API layer
compatible with Product Opener search cgi as well as v0/1/2 API)

The point (4) would be solved by deploying
openfoodfacts-query for more products types.

Point (5) should be able to be delegated to either data exports (for scripts)
or this service or openfoodfacts-query and search-a-licious.

## Out of scope for the project

### Users preferences and Organisations

In Product Opener, historically
they use the same "sto" storage pattern as Products.

Users identity have been moved to Keycloak ([openfoodfacts-auth repository](https://github.com/openfoodfacts/openfoodfacts-auth))
but organisations (in producers platform) and users preferences
are still in "sto" files.

Although there is benefit in moving these to a database,
the nature of the data is quite different from Products
in that it is not Open Data.

Users preferences and Organisations and other non-public data
should be handled by a different service.

### Advanced search queries

Advanced queries and pagination would be handled by
[off-query](https://github.com/openfoodfacts/openfoodfacts-query),
or [search-a-licious](https://github.com/openfoodfacts/search-a-licious)
and are out of scope of this service.

### Facets

As for search, there already is a service handling them:
[off-query](https://github.com/openfoodfacts/openfoodfacts-query)

### Data Export

[openfoodfacts-exports](https://github.com/openfoodfacts/openfoodfacts-exports)
is the service responsible for uploads.

It should switch it's reliance on MongoDB exports to using this service instead
(thanks to iteration interface).