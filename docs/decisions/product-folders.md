# Product Folders for Local Development

## Context and Problem Statement

Product Opener currently stores product details as JSON documents in a folder structure. The Product Catalog service needs to be able to support this structure and maintain the data for a period of time while services are migrated to use the new database.

However, for local development the product folders are currently stored in a docker volume that is part of the Product Opener project. This makes the folders difficult to access during local development.

## Decision Drivers

- simple approach that minimises inter-project dependencies
- works for local development outside of docker

## Considered Options

- keep existing Product Opener volume
- change Product Opener volume to be a bind mount
- move shared file system folders to Shared Services
- move product folder to Product Catalog

## Decision Outcome

Chosen option: "move product folder to Product Catalog", because it encapsulates data ownership with the service.

### Consequences

Migration will require merges in a specific order:

1. Product Catalog: Create the shared volume (as a bind mount)
2. Product Opener: 
   * Add Product Catalog as dependency
   * Migrate data from the old volume to the new one
   * Change containers to access the new volume
3. Product Catalog: Continued development
4. Product Opener: [after a suitable sunset period] Remove migration scripts

## Pros and Cons of the Options

### Keep existing Product Opener volume

The Product Catalog service could mount this volume in docker but for local development some kind of container would be needed to expose the volume as a Samba share, for example.

- Good: Least impact on Product Opener
- Bad: Circular dependency between Product Opener and Product Catalog
- Bad: Does not address transfer of ownership of product data
- Bad: Complex setup for local development

### Change Product Opener volume to be a bind mount

- Good: Easy for local and docker deployments
- Bad: Requires data migration
- Bad: Circular dependency between Product Opener and Product Catalog
- Bad: Does not address transfer of ownership of product data

### Move shared file system folders to Shared Services

- Good: Easy for local and docker deployments
- Bad: Requires data migration
- Bad: Does not encapsulate ownership of product data

### Move product folder to Product Catalog

- Good: Easy for local and docker deployments
- Good: Encapsulates ownership of product data
- Bad: Requires data migration

