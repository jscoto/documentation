---
title: "6.0.0 upgrade guide"
sidebar_position: -20
---

```mdx-code-block
import License from "@site/docs/pipeline-components-and-applications/loaders-storage-targets/snowplow-rdb-loader/reusable/license/_index.mdx"

<License/>
```

## Redshift invalid schema evolution recovery

### What is schema evolution?

One of Snowplow’s key features is the ability to [define custom schemas and validate events against them](/docs/understanding-your-pipeline/schemas/). Over time, users often evolve the schemas, e.g. by adding new fields or changing existing fields. To accommodate these changes, RDB loader automatically adjusts the database tables in the warehouse accordingly.

There are two main types of schema changes:

**Breaking**: The schema version has to be changed in a major way (`1-2-3` → `2-0-0`). In Redshift, each major schema version has its own table (`..._1`, `..._2`, etc, for example: `com_snowplowanalytics_snowplow_ad_click_1`).

**Non-breaking**: The schema version can be changed in a minor way (`1-2-3` → `1-3-0` or `1-2-3` → `1-2-4`). Data is stored in the same database table.

### How it used to work

In the past, the transformer would format the data according to the latest version of the schema it saw (for a given major version, e.g. `1-*-*`). For example, if a batch contained events with schema versions `1-0-0`, `1-0-1` and `1-0-2`, the transformer would structure the shredded data based on version `1-0-2`. Then the loader would adjust the database table and load the file.

This logic relied on two assumptions:

1. **Old events compatible with new schemas.** Events with older schema versions, e.g. `1-0-0` and `1-0-1`, had to be valid against the newer ones, e.g. `1-0-2`. Those that were not valid would result in failed events.

2. **Old columns compatible with new schemas.** The corresponding Redshift table had to be migrated correctly from one version to another. Changes, such as altering the type of a field from `integer` to `string`, would fail. Loading would break with SQL errors and the whole batch would be stuck and hard to recover.

These assumptions were not always clear to the users, making the transformer and loader error-prone.

### What happens now?

Transformer and loader are now more robust, and the data is easy to recover if the schema was not evolved correctly.


First, we support schema evolution that’s not strictly backwards compatible (although we still recommend against it since it can confuse downstream consumers of the data). This is done by _merging_ multiple schemas so that both old and new events can coexist. For example, suppose we have these two schemas:

```json
{
   // 1-0-0
   "properties": {
      "a": {"type": "integer"}
   }
}
```

```json
{
   // 1-0-1
   "properties": {
      "b": {"type": "integer"}
   }
}
```

These would be merged into the following:
```json
{
   // merged
   "properties": {
      "a": {"type": "integer"},
      "b": {"type": "integer"}
   }
}
```


Second, the loader does not fail when it can’t modify the database table to store both old and new events. (As a reminder, an example would be changing the type of a field from `integer` to `string`.) Instead, it creates a _temporary_ table for the new data as an exception. The users can then run SQL statements to resolve this situation as they see fit. For instance, consider these two schemas:
```json
{
   // 1-0-0
   "properties": {
      "a": {"type": "integer"}
   }
}
```

```json
{
   // 1-0-1
   "properties": {
      "a": {"type": "string"}
   }
}
```

Because `1-0-1` events cannot be loaded into the same table with `1-0-0`, the data would be put in a separate table, e.g. `com_snowplowanalytics_ad_click_1_0_1_recovered_9999999`, where:
  - `1_0_1` is the version of the offending schema;
  - `9999999` is a hash code unique to the schema (i.e. it will change if the schema is overwritten with a different one).

If you create a new schema `1-0-2` that reverts the offending changes and is again compatible with `1-0-0`, the data for events with that schema will be written to the original table as expected.

### Identification of the schemas that need patching

After the rdb-loader's upgrade, users might find out that some old schema lineages land in the recovery table.

To avoid this, older iglu schemas must be patched to align with the latest.

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

After offending schemas were identified earlier version of them should be patched to reflect the database changes.
Process above created a divergence in schema lineage. In such situations original schema without this patch schema `1-0-1` would land in a recovery table after loader upgrade.

Schema casting rules could be found [here](/docs/storing-querying/schemas-in-warehouse/?warehouse=redshift#types).

#### `$.featureFlags.disableRecovery` configuration

This is useful if you have older schemas with breaking changes and don’t want the loader to apply the new logic to them.

If you have older schemas with breaking changes and don’t want the loader to apply the new logic to them, you can use `$.featureFlags.disableRecovery` configuration. For the provided schema criterions only, RDB Loader will neither migrate the corresponding shredded table nor create recovery tables for breaking schema versions. Loader will attempt to load to the corresponding shredded table without migrating.
