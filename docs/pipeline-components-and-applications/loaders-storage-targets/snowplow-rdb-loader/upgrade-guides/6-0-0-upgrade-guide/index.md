---
title: "6.0.0 upgrade guide"
sidebar_position: -20
---

## License acceptance

Since version 6.0.0, RDB Loader has been migrated to use the [Snowplow Limited Use License](https://docs.snowplow.io/limited-use-license-1.0/) ([FAQ](/docs/contributing/limited-use-license-faq/index.md)).

You have to explicitly accept the [Snowplow Limited Use License](https://docs.snowplow.io/limited-use-license-1.0/) ([FAQ](/docs/contributing/limited-use-license-faq/index.md)). To do so, either set the `ACCEPT_LIMITED_USE_LICENSE=yes` environment variable, or update the following section in the configuration:

```hcl
{
  license {
    accept = true
  }
  ...
}
```

## Identification of the schemas that need patching
Starting with this version, breaking schema keys will begin to produce recovery tables.

Previously, transformer would just take the latest version of the schema from the iglu-server, without any considerations for the lineage. That was the source of many loading errors due to incompatible schemas.

Often, to recover from such errors, users would patch the database schema and perform a data migration for the affected
column.  Typical workflow:
- `1-0-0` - `{ "type": "integer", "maximum": 100 }` - translates to `SMALLINT`. All is going good.
- `1-0-1` - `{ "type": "integer", "maximum": 1000000 }` - breaking change as it translates to `INT` and loader can't
  migrate. As a result some or maybe all of your data doesn't load anymore. As long as the batch contain values over `32767` (smallint limit) it won't load.
  To work around this issue users often chose to perform migration in warehouse manually. Following typical steps:
- pause loading
- adding new `INT` column
- moving all the data to new column
- drop the old `SMALLINT`  column
- rename new column to old name
- alter the table comment
- resume loading
- _(leave the old schema as is)_

So the new database schema would match the latest published schema version in the iglu-schema. New recovery logic would not allow for this workaround. As it takes the entire lineage of schemas as opposed to just the latest version. It would detect the incompatible schema and put it into the recovery table. And after the rdb-loader's upgrade, users might find out that some old schema lineages land in the recovery table.

To avoid this, older iglu schemas must be patched to align with the latest. In the example above it would mean that `1-0-0` must be patched to change maximum to at least `32768` (first value of `INT`) or to `1000000` for simplicity, so type inference won't break.

To help with the identification of the schemas that need patching, we have rewritten the rdbms table-check command.
`igluctl` will now cross-check the types between the iglu server and RedShift or Postgres. Depending on what types
offended the schema errors might vary.

Procedure of identifying offending warehouse patches is as follows:

1) Run `igluctl static generate` command. If recovery table to be created it will show up as a warning message. Example:
```bash
mkdir <schemas_folder> <sql_folder>
igluctl static pull <schemas_folder> <iglu_url> <iglu_key>
igluctl static generate <schemas_folder> <sql_folder> 
# ...
# iglu:com.snowplowanalytics.iglu/resolver-config/jsonschema/1-0-3 has a breaking change Incompatible encoding in column cache_size old type RedshiftBigInt/ZstdEncoding new type RedshiftDouble/RawEncoding
# ...
```

2) Run `table-check` to crosscheck if warehouse was patched. This is done in case if  schema lineage had a broken schema
   that were supreseed and no longer used, for example:
```
100 - {"type" : "integer"}
101 - {"type" : "number"}  // deprecated
102 - {"type" : "integer"} // new corrected version
```
Situation above is not an issue, but would produce a warning in step 1).
```bash
igluctl --server <iglu_url> \
        --apikey <iglu_key> \
        --host <redshift_host> \
        --port <redshift_port> \
        --username <username> \
        --password <password> \
        --dbname  <database> \
        --dbschema <schema>
# ...
# * Comment problem - SchemaKey found in table comment [iglu:com.test/test/jsonschema/1-0-0] does not match expected [iglu:com.test/test/jsonschema/1-0-1]
# * Column doesn't match, expected: 'wrong_type BIGINT', actual: 'wrong_type VARCHAR(4096)'
# * Column existing in the storage but is not defined in the schema: 'only_in_storage VARCHAR(4096)'
# ...   
```
This might produce false positives for the JSONpaths managed tables.


After offending schemas were identified earlier version of them should be patched to reflect the database changes.
Process above created a divergence in schema lineage. In such situations original schema without this patch schema `1-0-1` would land in a recovery table after loader upgrade.

Schema casting rules could be found [here](/docs/storing-querying/schemas-in-warehouse/?warehouse=redshift#types).

#### `$.featureFlags.disableRecovery` configuration

This is useful if you have older schemas with breaking changes and don’t want the loader to apply the new logic to them.

If you have older schemas with breaking changes and don’t want the loader to apply the new logic to them, you can use `$.featureFlags.disableRecovery` configuration. For the provided schema criterions only, RDB Loader will neither migrate the corresponding shredded table nor create recovery tables for breaking schema versions. Loader will attempt to load to the corresponding shredded table without migrating.
