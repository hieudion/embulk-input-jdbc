# MySQL input plugin for Embulk

MySQL input plugin for Embulk loads records from MySQL.

## Overview

* **Plugin type**: input
* **Resume supported**: yes

**[WARNING!]**

- The default embulk type for MySQL JSON type will be changed from `string` to `json`.

## Configuration

- **driver_path**: path to the jar file of the MySQL JDBC driver. If not set, the bundled JDBC driver (MySQL Connector/J 5.1.44) will be used (string). NOTE: embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 . And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different. Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
- **host**: database host name (string, required)
- **port**: database port number (integer, 3306)
- **user**: database login user name (string, required)
- **password**: database login password (string, default: "")
- **database**: destination database name (string, required)
- If you write SQL directly,
  - **query**: SQL to run (string)
  - **use_raw_query_with_incremental**: If true, you can write optimized query using prepared statement by yourself. See [Use incremental loading with raw query](#use-incremental-loading-with-raw-query) for more detail (boolean, default: false)
- If **query** is not set,
  - **table**: destination table name (string, required)
  - **select**: expression of select (e.g. `id, created_at`) (string, default: "*")
  - **where**: WHERE condition to filter the rows (string, default: no-condition)
  - **order_by**: expression of ORDER BY to sort rows (e.g. `created_at DESC, id ASC`) (string, default: not sorted)
- **fetch_rows**: number of rows to fetch one time (integer, default: 10000)
  - If this value is set to > 1:
    - It uses a server-side prepared statement and fetches rows by chunks.
    - Internally, `useCursorFetch=true` is enabled and `java.sql.Statement.setFetchSize` is set to the configured value.
  - If this value is set to 1:
    - It uses a client-side built statement and fetches rows one by one.
    - Internally, `useCursorFetch=false` is used and `java.sql.Statement.setFetchSize` is set to Integer.MIN_VALUE.
  - If this value is set to -1:
    - It uses a client-side built statement and fetches all rows at once. This may cause OutOfMemoryError.
    - Internally, `useCursorFetch=false` is used and `java.sql.Statement.setFetchSize` is not set.
- **connect_timeout**: timeout for socket connect. 0 means no timeout. (integer (seconds), default: 300)
- **socket_timeout**: timeout on network socket operations. 0 means no timeout. (integer (seconds), default: 1800)
- **options**: extra JDBC properties (hash, default: {})
- **incremental**: if true, enables incremental loading. See next section for details (boolean, default: false)
- **incremental_columns**: column names for incremental loading (array of strings, default: use primary keys). Columns of integer types, string types, `datetime` and `timestamp` are supported.
- **last_record**: values of the last record for incremental loading (array of objects, default: load all records)
- **default_timezone**: If the sql type of a column is `date`/`time`/`datetime` and the embulk type is `string`, column values are formatted int this default_timezone. You can overwrite timezone for each columns using column_options option. (string, default: `UTC`)
- **use_legacy_datetime_code**: recommended not to set the property (boolean, default: false). If true, embulk-output-mysql will get wrong datetime values when the server timezone and the client server timezone are different as older embulk-output-mysql did.
- **default_column_options**: advanced: column_options for each JDBC type as default. key-value pairs where key is a JDBC type (e.g. 'DATE', 'BIGINT') and value is same as column_options's value.
- **column_options**: advanced: key-value pairs where key is a column name and value is options for the column.
  - **value_type**: embulk get values from database as this value_type. Typically, the value_type determines `getXXX` method of `java.sql.PreparedStatement`.
  (string, default: depends on the sql type of the column. Available values options are: `long`, `double`, `float`, `decimal`, `boolean`, `string`, `json`, `date`, `time`, `timestamp`)
  - **type**: Column values are converted to this embulk type.
  Available values options are: `boolean`, `long`, `double`, `string`, `json`, `timestamp`).
  By default, the embulk type is determined according to the sql type of the column (or value_type if specified).
  - **timestamp_format**: If the sql type of the column is `date`/`time`/`datetime` and the embulk type is `string`, column values are formatted by this timestamp_format. And if the embulk type is `timestamp`, this timestamp_format may be used in the output plugin. For example, stdout plugin use the timestamp_format, but *csv formatter plugin doesn't use*. (string, default : `%Y-%m-%d` for `date`, `%H:%M:%S` for `time`, `%Y-%m-%d %H:%M:%S` for `timestamp`)
  - **timezone**: If the sql type of the column is `date`/`time`/`datetime` and the embulk type is `string`, column values are formatted in this timezone.
(string, value of default_timezone option is used by default)
- **before_setup**: if set, this SQL will be executed before setup. You can prepare table for input by this option.
- **before_select**: if set, this SQL will be executed before the SELECT query in the same transaction.
- **after_select**: if set, this SQL will be executed after the SELECT query in the same transaction.


## Incremental loading

Incremental loading uses monotonically increasing unique columns (such as AUTO_INCREMENT column) to load records inserted (or updated) after last execution.

First, if `incremental: true` is set, this plugin loads all records with additional ORDER BY. For example, if `incremental_columns: [updated_at, id]` option is set, query will be as following:

```
SELECT * FROM (
  ...original query is here...
)
ORDER BY updated_at, id
```

When bulk data loading finishes successfully, it outputs `last_record: ` paramater as config-diff so that next execution uses it.

At the next execution, when `last_record: ` is also set, this plugin generates additional WHERE conditions to load records larger than the last record. For example, if `last_record: ["2017-01-01T00:32:12.487659", 5291]` is set,

```
SELECT * FROM (
  ...original query is here...
)
WHERE updated_at > '2017-01-01 00:32:12.487659' OR (updated_at = '2017-01-01 00:32:12.487659' AND id > 5291)
ORDER BY updated_at, id
```

Then, it updates `last_record: ` so that next execution uses the updated last_record.

**IMPORTANT**: If you set `incremental_columns: ` option, make sure that there is an index on the columns to avoid full table scan. For this example, following index should be created:

```
CREATE INDEX embulk_incremental_loading_index ON table (updated_at, id);
```

Recommended usage is to leave `incremental_columns` unset and let this plugin automatically finds an AUTO_INCREMENT primary key. Currently, only strings, integers, DATETIME and TIMESTAMP are supported as incremental_columns.


## Trouble shooting

- If you get an exception 'The server time zone value XXX is unrecognized ...', please set proper time zone to the MySQL server or set `true` to the `use_legacy_datetime_code` property.

### Use incremental loading with raw query

**IMPORTANT**: This is an advanced feature and assume you have an enough knowledge about incremental loading using Embulk and this plugin

Normally, you can't write your own query for incremental loading.
`use_raw_query_with_incremental` option allow you to write raw query for incremental loading. It might be well optimized and faster than SQL statement which is automatically generated by plugin.

Prepared statement starts with `:` is available instead of fixed value.
`last_record` value is necessary when you use this option.
Please use prepared statement that is well distinguishable in SQL statement. Using too simple prepared statement like `:a` might cause SQL parse failure.

In the following example, prepared statement `:foo_id` will be replaced with value "1" which is specified in `last_record`.

```yaml
in:
  type: mysql
  query:
    SELECT
      foo.id as foo_id, bar.name
    FROM
      foo LEFT JOIN bar ON foo.id = bar.id
    WHERE
      foo.hoge IS NOT NULL
      AND foo.id > :foo_id
    ORDER BY
      foo.id ASC
  use_raw_query_with_incremental: true
  incremental_columns:
    - foo_id
  incremental: true
  last_record: [1]
```

## Example

```yaml
in:
  type: mysql
  host: localhost
  user: myuser
  password: ""
  database: my_database
  table: my_table
  select: "col1, col2, col3"
  where: "col4 != 'a'"
  order_by: "col1 DESC"
```

This configuration will generate following SQL:

```
SELECT col1, col2, col3
FROM `my_table`
WHERE col4 != 'a'
ORDER BY col1 DESC
```

If you need a complex SQL,

```yaml
in:
  type: mysql
  host: localhost
  user: myuser
  password: ""
  database: my_database
  query: |
    SELECT t1.id, t1.name, t2.id AS t2_id, t2.name AS t2_name
    FROM table1 AS t1
    LEFT JOIN table2 AS t2
      ON t1.id = t2.t1_id
```

Advanced configuration:

```yaml
in:
  type: mysql
  host: localhost
  user: myuser
  password: ""
  database: my_database
  table: "my_table"
  select: "col1, col2, col3"
  where: "col4 != 'a'"
  default_column_options:
    TIMESTAMP: { type: string, timestamp_format: "%Y/%m/%d %H:%M:%S", timezone: "+0900"}
    BIGINT: { type: string }
  column_options:
    col1: {type: long}
    col3: {type: string, timestamp_format: "%Y/%m/%d", timezone: "+0900"}
  after_select: "update my_table set col5 = '1' where col4 != 'a'"

```

## Build

```
$ ./gradlew gem
```

Running tests:

```
$ cp ci/travis_mysql.yml ci/mysql.yml  # edit this file if necessary
$ EMBULK_INPUT_MYSQL_TEST_CONFIG=`pwd`/ci/mysql.yml ./gradlew :embulk-input-mysql:check --info
```

This test data are expected by using 'UTC' as MySQL server's timezone. On the other hand, unit tests use 'Europe/Helsinki' as jdbc driver's session timezone.
