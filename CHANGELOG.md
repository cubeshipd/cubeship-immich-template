# Changelog

## [2.0.0](https://github.com/cubeshipd/cubeship-immich-template/compare/v1.0.0...v2.0.0) (2026-09-15)


### ⚠ BREAKING CHANGES

* an existing installation is not migrated. Its postgres app and the volume under it stay where they are; moving to the managed database means installing fresh and copying the data across with pg_dump and psql. Requires Cubeship 0.9.0.

### Features

* run Immich on a managed Postgres with extensions ([43c675f](https://github.com/cubeshipd/cubeship-immich-template/commit/43c675f3eb75672aa62288a490c8779a59352966))
