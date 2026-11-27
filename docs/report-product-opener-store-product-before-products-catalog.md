# Report: storage mechanisms for products in Product Opener (before Products Catalog)

Products Catalog is a new service
that will replace the current product storage mechanisms in Product Opener.

[Product Opener (aka openfoodfacts-server)](https://github.com/openfoodfacts/openfoodfacts-server)
is a monolithic Perl application
that stores product data in a combination of disk files.

This is the situation before Products Catalog is implemented (at commit [a4b52231](https://github.com/openfoodfacts/openfoodfacts-server/tree/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/).
This helps understand the perimeter Products Catalog will need to handle.

## Product storage

### `store_product`

Normally product storage should go through `store_product` sub.

Here is how it currently work and should be adapted.

#### How `store_product` currently works

Normally every product write goes through
[`ProductOpener::Products.pm::store_product`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Products.pm#L1300).

The function signature is:
`store_product ($user_id, $product_ref, $comment, $client_id = undef)`

Write path, in order:

1. **Schema**: sets `schema_version`.
2. **Structural-move detection**:
   `old_code`, `old_product_type` or `server` set
   → never skip the save (`has_structural_move`, Products.pm:1312).
3. **Server/product_type resolution**: `server` → `change_product_type`,
   to detect if product type or server has changed.
   Then `get_server_for_product` / `get_products_collection` for the *previous*
   and *new* Mongo collections (`products` vs `products_obsolete`, incl. cross-flavour servers).
4. **Barcode change** (`old_code`):
   moves `PRODUCTS` and `PRODUCTS_IMAGES` dirs,
   deletes the old Mongo doc from the previous collection,
   recomputes `_id` (with owner prefix on pro),
   appends "barcode changed" to the comment.
5. **Revision**:
   `rev` is set to `scalar @changes`
   (that is, recomputed from the list of revision in `changes`.
   This provides a self-heals for desynced revs, but can also skip data),
   then incremented.
6. **Metadata**:
   `last_modified_by`, `last_modified_by_client`,
   `last_modified_t`, `last_updated_t`,
   `creator` (defaults to `openfoodfacts-contributors` when `user_id` empty),
   `owners_tags`.
7. **History entry**:
   `change_ref` = userid,
   clientid, ip, `user_agent`/`app_name`/`app_version`/`app_uuid`
   (read from form params), pushed onto `changes`.
8. **Computations**:
   `add_user_teams`, `compute_codes`, `compute_languages`, `compute_product_history_and_completeness`,
   `compute_data_sources`, `compute_main_countries`, `compute_sort_keys`, `compute_keywords`;
   `_id`/`code` string coercion, numeric coercion,
   `make_sure_numbers_are_stored_as_numbers`.
9. **No-op skip**: for `rev > 1` with no structural move,
   compares against the stored `product`;
   if equivalent, restores state and returns 1 without writing.
10. **Persist**:
    1. `store_object(".../$path/$rev")` (disk),
    2. Mongo `replace_one(upsert)` or `delete_one` if `deleted`,
       and `delete_one` from the previous collection
       if obsolete/server/type changed,
    4. update the `product` link (disk),
    5. `store_object(".../changes")` (disk)
    6. `log_change` (recent_changes collection in MongoDB),
    7. `push_product_update_to_redis($action)`.
11. Always returns `1`.

#### What changes for `store_product` with Products Catalog

The points that should be handled by Products Catalog:
* **No-op skip (9)** - this should be Products Catalog to decide if we have a no-op
  (because we must ensure it is done on the right version, within the transaction)
  This could mean that we store a meaningfull hash
  (see `_products_are_equivalent_for_revision` function)
  given by Product Opener in the database,
  or Products Catalog knows how to compare revisions
  (mostly skipped keys but also space trim).
* **Persist (10)** - of course this is where we replace all the logic
  by a call to Products Catalog write API.

The points still partially handled by Product Opener in a different way
* **Structural-move detection (2)** now it has to be done using Products Catalog API to retrieve old product data that is needed
* **Server/product_type resolution (3)** Product Opener still decides
  on product and type change
  but it may be a bit different.
* **Barcode change (4)**
  it's Product Opener that handles the images directory change.
  MogoDB deletion must be provided by Products Catalog
* **Revision (5)** is set by Product Opener, but:
  * Products Catalog must ensure revision consistency.
  * it should be computed from the old revision + 1 (now safe thanks to Products Catalog)

The points that don't change in Product Opener:
* **Schema (1)**
* **Metadata (6)**
* **History entry (7)**
* **Computations (8)** - note that Products Catalog might (in the future)
  check that the JSON is fully valid and correspond to its schema.

#### Transaction capabilities

Currently `store_product` is not transactional.

The future version, calling Products Catalog API,
must make provision for retry (in case of concurrent edits)
and account for possible errors (network problem, 400, 503 etc.)
in a better way than today.

Product Catalog, on its side, will be transactional.

### Noticeable `store_product` call sites

There are cases where `store_product` is called in non-standard ways:

- [`APIProductRevert::revert_product_api (at line 161)`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/APIProductRevert.pm#L161)
  passes `$revision_ref`
  (a file revision loaded via `retrieve_product($id, 0, $rev)`,
  already schema-converted/normalized and with `{server}` injected by
  `retrieve_product`) as the new current product.
  It is written as a fresh revision;
  relies on `store_product` to rebuild `_id`/derived fields.

  - Product Catalog should provide a revert API, and this method needs to be rewritten

- Return value of `store_product` (always `1`)
  is effectively unused:
  only [`cgi/product_jqm_multilingual.pl:451`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/cgi/product_jqm_multilingual.pl#L451)
  checks it,
  but `store_product` always returns `1`,
  so its `else` ("not modified") branch is dead code.

  - this needs to be handled as a bug in Product Opener ([issue #15559](https://github.com/openfoodfacts/openfoodfacts-server/issues/15559)

- [`cgi/product_multilingual.pl:285`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/cgi/product_multilingual.pl#L285)
  stores the product ref, and then keeps editing the same ref in the request.

  - this should not be a problem, although this could be considered a bad pattern.

- [`Images.pm:1447`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Images.pm#L1447)
  creates a product implicitly from an image move
  ("Creating product (image move)").

  - there again this is not a problem

- Scripts/imports pass an empty `user_id`
  → creator becomes `openfoodfacts-contributors`
  (intended behaviour, not a bug), the owner is set correctly.

  - This is not a bug, but still I think that an identified off-import user would be better.

### Direct writes that bypass `store_product`

#### In `lib/`

- [`Import.pm:3215`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Import.pm#L3215) (+3219)
  export-status update: `store_object(.../product)`
  + Mongo `replace_one`, no new rev, no history/Redis.

  - this is hackish as it is.
    Products Catalog will provide an API to update a product without creating a new revision (there might be some conditions).
    Here we update the export status of the object (why not a new revision ?)

- [`Products::replace_user_id_in_product:2297-2313`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Products.pm#L2297-L2313)
  rewrites *every* revision file,
  updates the newest Mongo doc
  and pushes Redis manually;
  no rev bump, no `changes` entries.
  This method is used when we remove a user from the database.

  - this is a real problem, because otherwise old product revisions are immutable.

    We must consider if it's worth having a mapping from anonymous hash to user_id
    directly in Product Catalog.
    See [issue #3](https://github.com/openfoodfacts/products-catalog/issues/3)

- [`Images.pm:1232`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Images.pm#L1232)
  writes the auxiliary `.../images` object (not the product itself).

  - this is a real problem to deal with, see [issue #4](https://github.com/openfoodfacts/products-catalog/issues/4)

- `scans.json` also live in the products files hierarchy,
  see [issue #5](https://github.com/openfoodfacts/products-catalog/issues/5)

#### In `scripts/`

- [`scanbot.pl:527`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/scanbot.pl#L527) — scan-count update: `store_object` + `replace_one` + manual Redis.

  - It should use the products-catalog API.

- [`update_all_products.pl`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/update_all_products.pl) — main "silently updated" path (:1634, :1643), manual Redis (:1667); `_id` owner-prefix fix (:570-579); `changes` rewrites (:898, :1475); pre-autorotate save (:1212); [`check_products_in_sto_files.pl:120`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/check_products_in_sto_files.pl#L120) does use `store_product`.

  - This is a major scripts to look at, but it should fit within the proposed Product Catalog API.
    see [issue #6](https://github.com/openfoodfacts/products-catalog/issues/6)

- [`revert_changes_from_user.pl:204`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/revert_changes_from_user.pl#L204) (`changes` rewrite) and `:227` — `replace_one({"id" => _id})` (suspicious key) with disk/Mongo rewritten separately.

  - Another script to look at, because it's a bit specific. see [issue #7](https://github.com/openfoodfacts/products-catalog/issues/7)

- scripts with direct-path `store_object` + `replace_one`/`delete_one`.
  - [`fix_non_normalized_codes.pl:91, 173-179`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/fix_non_normalized_codes.pl#L91) — direct-path `store_object` + `replace_one`/`delete_one`.
  - [`fix_mongodb_collections.pl:87,126`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/fix_mongodb_collections.pl#L87) — `store_object` + `insert_one`/`delete_one`.

  - those scripts should be easy to change to use Products Catalog

- whole directories move:

  - [`cgi/remove_products.pl:80`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/cgi/remove_products.pl#L80),

  - [`scripts/remove_all_private_products_for_owner.pl:59`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/remove_all_private_products_for_owner.pl#L59).

  This is specific to producers platform.

  Products Catalog might offer a direct API for those use cases. [issue #8]

- Tests write raw fixtures:

  - [`tests/integration/fix_non_normalized_codes.t:47`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/tests/integration/fix_non_normalized_codes.t#L47)

    - this should be possible to use Products Catalog API

  - [`tests/unit/store.t`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/tests/unit/store.t) (generic store tests).

    - this one would be obsolete or can be changed to use the new API

**`scripts/obsolete/*`** also write `product` directly
but are out of scope.


## Product retrieval / read

### `retrieve_product`

Normally all product read should use this method

#### How `retrieve_product` currently works

[`Products::retrieve_product ($product_id, $include_deleted = 0, $rev = undef)`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Products.pm#L1059)

Read path (filesystem only — never MongoDB):

1. **Compute path**: `product_path_from_id($product_id)`
   ([`Products.pm:846`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Products.pm#L846)) — on the pro platform splits `owner/code` and splits the code into sub-dirs (`split_code`), else just `split_code($product_id)`.
2. **Chose object**: `$rev` given → `PRODUCTS/$path/$rev`; else `PRODUCTS/$path/product` (the symlink to the current revision). `$rev` must be `^\d+$`, otherwise returns `undef` silently.
3. **Load**: `retrieve_object($full_product_path)` ([`Store.pm:467`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Store.pm#L467)) — prefers `.sto` if present (legacy master), else `.json`; follows symlinks; `carp`s and returns undef on a malformed JSON read.
4. **Deleted gate**: if `deleted` and not `$include_deleted` → returns `undef` (logs debug). Note: check is truthy (`"on"`/any value), not `eq 'on'`.
5. **Server tagging**: if `server_for_product_type($product_ref->{product_type})` differs from the current server, injects `$product_ref->{server}` (so a later `store_product` writes to the right flavour / Mongo DB); otherwise deletes a stale `server` field.
6. **Schema conversion**: `convert_product_schema($product_ref, $current_schema_version)` ([`ProductSchemaChanges.pm:101`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/ProductSchemaChanges.pm#L101), currently 1004) — incremental upgrade (or downgrade) of in-memory data, including barcode normalization etc. Uses the global `$Org_id` (acknowledged design wart, see comment at ProductSchemaChanges.pm:77-82).
7. **Normalization**: `normalize_product_data` ([`Products.pm:1819`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Products.pm#L1819)) — coerces `lc`/`lang` to one main language (default `en`).
8. **Returns the product**: return the (possibly mutated) product ref — same hash shape for current and historical revisions; there is no read cache.

Important: `convert_product_schema`/`normalize_product_data`
run on *every* read, including old revisions,
and it always returns a schema-converted object.

#### What changes for `retrieve_product` with Products Catalog

1. **Compute path**: not needed, the API only needs the code or id
2. **Chose object**: not needed, the API only needs the code or id
3. **Load**: this should be the API call to Product Catalog
   * errors handling must be included
4. **Deleted gate**: the result returned by Product Catalog
   should clearly state if a product was removed
5. **Server tagging**: This logic will not change
6. **Schema conversion**: There again this does not change,
   schema conversion is the responsibility of Product Opener.
7. **Normalization**: does not change
8. **Returns the product**: does not change

Overall this is a simplification and there is no big issue in the transformation

### Noticeable call sites for retrieve_product

There are cases where `store_product` is called in non-standard ways:

- **`scripts/get_product_from_mongodb.pl:10-22`**
  comment says "Retrieve the product from MongoDB" but it calls `retrieve_product`,
  i.e. reads the file.

  * Misleading name but can use the Products Catalog.

- **[`Import.pm::import_products_categories_from_public_database`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Import.pm#L3301)**
  reads the public product by building the path and calling `retrieve_object` directly,
  bypassing `retrieve_product` *because it is reading another server's export directory*.
  This skips schema conversion and `server` tagging — intentional but fragile.

  * We have to change that so the pro platform should read the public Products Catalog

- **`Products.pm:2485`** — `retrieve_product($product_id, 1, $rev)` inside `compute_product_history_and_completeness`: re-reads every revision file during a save, so the file layout is hit heavily on writes.

  * Either as a first step it means store will read the whole product history
    from Products Catalog.

    But maybe it would be a huge improvement to just run it
    when we import products in the database
    and then rely on the atomicity of the database to avoid such a computation.

- **`scripts/fix_non_normalized_codes.pl:65, 88, 91`** — deliberately uses `retrieve_object` on the raw path instead of `retrieve_product`, because the path is invalid for `retrieve_product`.

  * This should not be a problem, there won't be any path any more.
    It should be replaced by the API call to products catalog.

### Read of changes

Some part of the codes read the `changes.json` file:

* [`Display::display_product_api`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Display.pm:9823)
* [`Display::display_rev_info`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Display.pm:9844)
* [`Display::get_product_history_revisions`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Display.pm:9915)

* [`APIProductRead::read_product_api`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/APIProductRead.pm:207)

* [`cgi/product_image.pl`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/cgi/product_image.pl)

Scripts:
* `scripts/count_product_contributions_by_year.pl`
* `scripts/extract_historical_product_data.pl`
* `scripts/update_recent_changes.pl`

### Reads that bypass `retrieve_product`

#### Product data read directly from files

This means (no schema conversion, no deleted check, no `server` tagging)

- [`Products.pm::assign_new_code`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Products.pm#L427)
  uses a global `internal_code.sto` file

  - This should be provided by the API of Products Catalog

- [`Products.pm::replace_user_id_in_product`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Products.pm#L441)
  reads revision directly,

  - we mentioned that already, see [issue #3](https://github.com/openfoodfacts/products-catalog/issues/3)

- `scripts/check_products_in_sto_files.pl`, `scripts/fix_mongodb_collections.pl`
  iterates all the files by path

- `scripts/update_all_products_from_dir_in_mongodb.pl`

#### Reads from MongoDB instead of the product file

For search and queries this is expected (I don't list them).

But there are a number of places where we iterate the database
by using MongoDB.
This should instead be provided by product Catalog

- [`Data.pm:175, 235`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Data.pm#L175)
  provides iteration helpers that should use this new API instead

- `PackagerCodes`/[`PackagingStats.pm:626`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/PackagingStats.pm#L626)
- [`Export.pm:295-302`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Export.pm#L295)
- [`Missions.pm:233`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Missions.pm#L233).
- `scripts/update_all_products_from_dir_in_mongodb.pl`


### Other usages of store / retrieve (sto files)

Those are eventual blockers for full horizontal scalability of Product Opener.
A service for "generic" storage could solve this issue (but is not the current focus).

As already mentions, we have the `scans.json` and `images.json` files in product folders.

* Usage as a database
  * `Users.pm` stores users preferences: `users/<id>.sto`
  * `scripts/gen_users_emails.pl` re-creates `users_emails.sto`
    (this one should be normally deprecated since we have keycloak)
  * `Orgs.pm` stores org data `orgs/<id>.sto` and a global `orgs_glns.sto`
  * `Producers.pm` stores columns mapping in `orgs/<id>/all_columns_fields.sto`
    (this could be an attribute of org)

    and history of exports as `orgs/<id>/exports.sto`
  * scripts around import asynchronous jobs (`import_file*.pl`)
    * store imports in `import_files.sto` per org,
    * they also store some data about the imports in `sto` files

* Usage to store global data
  * `build_lang.pl` and `Lang.pm` stores language compilation results
  * `Missions.pm` stores a global `missions.sto`
  * `scripts/update_packager_codes*.pl` and `PackagerCodes.pm` stores a global `packager-codes/packager_codes.sto`
  * `Display.pm` and `scripts/gen_top_tags_per_country.pl`
    store global private data like users / ambassadors / countries points
  * `Stats.pm` stores categories stats per country in `categories_stats` folder
  * `scripts/scanbot.pl` stores a global `all_products_scans.sto` file

* `Redis.pm` us a sto file to persist the stream id.

  This could be simply stored in Redis.

* Usage for cache
  * `CRM.pm` stores a temporary cache file `crm_data.sto`
  * `APITest::fake_http_server` reads and stores temporary queries and responses

  We could use Redis for those use cases.

## Some global conclusion

* We need to have org_id in the database schema for producer platform to be able to use Products Catalog (for public platforms it will always be `null`, and it should never be `null` in producers platform. This could be a parameter for the Products Catalog database instance).

* We need to be able to read the changes of a product

* We need to be able to iterate all the products
  (it may come afterwards, in-between MongoDB is still usable).

  A cursor parameter might be a good idea to avoid having page boundaries problems.

* We might need to add an "id" field with "org_id/code" or "code"

* We need a read all version of a product at once function

* We should be able to assign a new code to a product without a barcode
  avoiding conflict (a counter seems a good idea)