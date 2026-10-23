# Products catalog (for Open Food Facts)

This service is the service in charge of storage and archival of products data for Product Opener (aka [openfoodfacts-server](https://github.com/openfoodfacts/openfoodfacts-server)).

It's a wrapper around all product reads and writes.

This service use [MariaDB](https://mariadb.com/) as a backend.
[RepMan](https://docs.signal18.io/) from [Signal18](https://signal18.io/)
is used to monitor and replicate the MariaDB instances,
bringing effective scalability and a lot of other benefits.

This project is made possible by a grant from Hyper Open X, funded by the France 2030 program (french government) and Next Generation EU (France relance)


See [Explain project scope](./docs/explain-scope.md)