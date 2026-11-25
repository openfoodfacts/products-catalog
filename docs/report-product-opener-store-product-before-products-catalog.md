# Report: storage mechanisms for products in Product Opener (before Products Catalog)

Products Catalog is a new service
that will replace the current product storage mechanisms in Product Opener.

[Product Opener (aka openfoodfacts-server)](https://github.com/openfoodfacts/openfoodfacts-server)
is a monolithic Perl application
that stores product data in a combination of disk files.

This is the situation before Products Catalog is implemented (at commit [a4b52231](https://github.com/openfoodfacts/openfoodfacts-server/tree/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/)).
This helps understand the perimeter Products Catalog will need to handle.


## `store_product`

### How `store_product` currently works

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

### Which step should be under Products Catalog control

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

### Transaction capabilities

Currently `store_product` is not transactional.

The future version, calling Products Catalog API,
must make provision for retry (in case of concurrent edits)
and account for possible errors (network problem, 400, 503 etc.)
in a better way than today.

Product Catalog, on its side, will be transactional.

## Odd `store_product` call sites

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

  - this needs to be handled as a bug in Product Opener ([issue #15559](https://github.com/openfoodfacts/openfoodfacts-server/issues/15559))

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

## Direct writes that bypass `store_product`

### In `lib/`

- [`Import.pm:3215`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/lib/ProductOpener/Import.pm#L3215) (+3219)
  export-status update: `store_object(.../product)`
  + Mongo `replace_one`, no new rev, no history/Redis.

  - this is hackish as it is.
    Products Catalog will provide an API to update a product without creating a new revision (there might be some conditions)

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

- scans.json also live in the products files hierarchy,
  see [issue #5](https://github.com/openfoodfacts/products-catalog/issues/5)

### In `scripts/`

- [`scanbot.pl:527`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/scanbot.pl#L527) — scan-count update: `store_object` + `replace_one` + manual Redis.

  - It should use the products-catalog API.

- [`update_all_products.pl`](https://github.com/openfoodfacts/openfoodfacts-server/blob/a4b52231eb6c221c7c5fa279e3232b5c233fd7f2/scripts/update_all_products.pl) — main "silently updated" path (:1634, :1643), manual Redis (:1667); `_id` owner-prefix fix (:570-579); `changes` rewrites (:898, :1475); pre-autorotate save (:1212); `check_products_in_sto_files.pl:120` does use `store_product`.

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
