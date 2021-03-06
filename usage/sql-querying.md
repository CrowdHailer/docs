---
description: Powerful SQL querying against your structured log data
---

# SQL Querying

Timber offers [full ANSI compliant SQL querying](https://prestodb.github.io/docs/current/sql/select.html) against your structured log data, providing you with powerful unrestricted access to your log data. Query your log data just as you would any SQL compliant database.

{% hint style="info" %}
This feature is designed for complex long-term querying. Data is made available on 15 minute intervals. For real-time access please see the [Live Tailing & Searching feature](live-tailing.md).
{% endhint %}

## Getting Started

{% hint style="info" %}
SQL Querying is limited to the [Timber CLI](../clients/cli/) since it is a low-level data feature. Integration into the Timber web app is planned.
{% endhint %}

1. [Install the `timber` CLI.](../clients/cli/#installation)
2. List your available sources:  


   ```bash
   timber sources
   ```

3. Execute the `sql-queries` command using your chosen source ID as the [table name](sql-querying.md#tables):  


   ```bash
   timber sql-queries execute "
   SELECT dt, message
   FROM source_{source_id}
   LIMIT 10
   "
   ```

![Timber SQL Querying Demo](../.gitbook/assets/sql_queries-executing.gif)

## Usage

### Executing Queries

#### SQL Query Syntax

Timber supports the [Presto `SELECT` syntax](https://prestodb.github.io/docs/current/sql/select.html):

```text
[ WITH with_query [, ...] ]
SELECT [ ALL | DISTINCT ] select_expr [, ...]
[ FROM from_item [, ...] ]
[ WHERE condition ]
[ GROUP BY [ ALL | DISTINCT ] grouping_element [, ...] ]
[ HAVING condition]
[ { UNION | INTERSECT | EXCEPT } [ ALL | DISTINCT ] select ]
[ ORDER BY expression [ ASC | DESC ] [, ...] ]
[ LIMIT [ count | ALL ] ]
```

Please see the [Presto `SELECT` docs](https://prestodb.github.io/docs/current/sql/select.html) for a comprehensive syntax overview.

#### Tables

Your table name is formatted as `source_{id}`. Where `{id}` is replaced by your actual source ID.

For example: `SELECT * FROM source_1234 LIMIT 100` would select data from source ID `1234`.

{% hint style="info" %}
Your can obtain your source ID via the `timber sources` command.
{% endhint %}

#### Columns

Any column you send as part of your log data is automatically made available for querying. If you haven't already, please read out [Dynamic Schema Maintenance document](../under-the-hood/schema-maintenance.md).

Nested columns are delimited by a `.`. For example `context.user.id` 

{% hint style="warning" %}
When specifying columns with a `.` be sure to quote the name with \` characters!
{% endhint %}

For example \(notice the column quoting\):

```sql
SELECT `context.user.id`
FROM source_{id}
LIMIT 10
```

#### Special Columns

| Name | Type | Description |
| :--- | :--- | :--- |
| `application_id` | `int` | The ID of the source, application is an unfortunate legacy term. |
| `dt` | `float` | Log date in fractional [Unix timestamp format](https://en.wikipedia.org/wiki/Unix_time) \(float\). The timestamp represents seconds and the fractions represent fractions of a second. Use this to efficiently narrow your queries to a specific date range. |
| `normalized_message` | `string` | Downcased and [ANSI formatting](https://en.wikipedia.org/wiki/ANSI_escape_code) stripped. Convenient for sub-string searching. |
| `raw_message` | `string` | The raw, unaltered message that you sent to Timber. |
| `severity` | `int` | Numerical representation of the `level` field. The value follows the [Syslog 5424 severities](https://en.wikipedia.org/wiki/Syslog#Severity_level). |

#### Functions

Please see the [Presto `SELECT` docs](https://prestodb.github.io/docs/current/sql/select.html) for a comprehensive overview of all functions available.

#### Example 1: Retrieve the last 50 logs logs:

```sql
SELECT dt, message
FROM source_{id}
ORDER BY `dt.desc`
LIMIT 50
```

#### Example 2: Count logs over the past 24 hours:

```sql
SELECT COUNT(*) AS count
FROM source_{id}
WHERE dt >= (now() - interval '24' hour)
```

#### Example 3: Count errors by user over the last week

{% hint style="info" %}
This query assumes you have a `user.id` field.
{% endhint %}

```sql
SELECT
    `context.user.id` AS user_id,
    COUNT(*) AS count
FROM source_{id}
WHERE
    level = 'error' AND
    dt >= (now() - interval '1' week)
GROUP BY `context.user.id`
```

#### Example 4: Average HTTP server response times over the last 24 hours \(1 minute intervals\):

{% hint style="info" %}
This query assumes you have a `http_response_sent.duration_ms` field.
{% endhint %}

```sql
SELECT
    floor(dt - mod(dt, 60)) AS interval,
    AVG(`http_response_sent.duration_ms`) AS response_avg
FROM source_{id}
WHERE dt >= (now() - interval '24' hours)
GROUP BY floor(dt - mod(dt, 60))
```

#### Example 5: Search logs by a phrase

```sql
SELECT dt, message
FROM source_{id}
WHERE normalized_message LIKE '%sent 500%'
ORDER BY `dt.desc`
LIMIT 50
```

`normalized_message` is a special field that Timber provides. It is a downcased and ANSI formatted stripped version of the `message` field, providing for case-insensitive searching.

### Listing Queries

```text
timber sql-queries
```

### Getting A Query's Info

1. [Execute a query](sql-querying.md#executing-queries) or [list your queries](sql-querying.md#listing-queries).
2. Run the `info` sub-command to get a query's status:  


   ```bash
   timber sql-queries info [sql_query_id]
   ```

Please see the [Query Statuses section](sql-querying.md#query-statuses) for status explanations.

### Displaying Results

1. [Execute a query](sql-querying.md#executing-queries) or [list your queries](sql-querying.md#listing-queries).
2. Run the `results` sub-command to displaying the results, replacing `[sql_query_id]` with the ID of your query:  


   ```bash
   timber sql-queries results [sql_query_id]
   ```

### Downloading Results

1. [Execute a query](sql-querying.md#executing-queries) or [list your queries](sql-querying.md#listing-queries).
2. Run the `download` sub-command to download the result, replacing `[sql_query_id]` with the ID of your query:  


   ```bash
   timber sql-queries download [sql_query_id] | open
   ```

   We pipe with the `open` command to

### Cancelling Queries

{% hint style="info" %}
You cannot cancel queries that are not RUNNING.
{% endhint %}

You can cancel running queries by issuing the `cancel` sub-command:

```text
timber sql-queries cancel [sql_query_id]
```

## Query Statuses

SQL queries can have any of the following statuses:

| Status | Description |
| :--- | :--- |
| `QUEUED` | The query is queued for running. Typically queries run immediately or within minutes if they are queued. Queueing should not last longer than 5 minutes. |
| `RUNNING` | The query is currently running. |
| `SUCCEEDED` | The query successfully completed. |
| `FAILED` | The query failed due to an error. Get the status to view error details. |
| `CANCELLED` | The query was [manually cancelled](sql-querying.md#cancelling-queries). |

## Usage Calculation

Each SQL query scans data in order to execute and return its result. The amount of data scanned is entirely dependent on the query and the data within your account. The data scanned will be displayed in your client after the query has finished executing.

It is very easy to write a query that will scan all of your data and exhaust your limit. Additionally, it is just as easy to write efficient queries that only scan the smallest amount of data necessary. Please see our [querying best practices](sql-querying.md#best-practices) on how to do this.

## Best Practices

1. Supply a date range on the `dt` column to limit the amount of data scanned, otherwise all data within your account will be scanned.
2. Supply a `LIMIT` to return early and reduce the amount of data returned.
3. Specify individual columns within the `SELECT` clause to reduce the amount of data scanned and returned. Avoid `*`.
4. Avoid `JOIN`s if possible.

## How It Works

{% hint style="info" %}
If you haven't already, please read our [Log Ingestion document](../under-the-hood/ingestion-pipeline.md) for a deeper dive into our log ingestion pipeline.
{% endhint %}

### Architecture

Timber persists your data in an efficient [columnar format](https://en.wikipedia.org/wiki/Column-oriented_DBMS) \([ORC](https://orc.apache.org/)\) on [S3](https://aws.amazon.com/s3/) and uses [Athena](https://aws.amazon.com/athena/) \([Presto](https://prestodb.github.io/)\) to query that data. Athena utilizes thousands of CPU cores to query your data, resulting in [incredibly fast query speeds](https://tech.marksblogg.com/billion-nyc-taxi-rides-aws-athena.html).

### Performance

Timber's S3 / SQL querying pipeline is built to extract every possible performance benefit. This is the result of hard won experience building and maintaining big data S3 pipelines. With Timber you get all of this as part of your account:

1. Timber [dynamically maintains a schema](../under-the-hood/schema-maintenance.md) for each of your sources.
2. Because we maintains a consistent schema we're able to write your data in compressed ORC columnar format for efficient data retrieval.
3. Hourly partitions are used for high-level granular access, helping to reduce the amount of data scanned for each query.

### Asynchronous Processing

Because SQL queries can vary in complexity, they are executed asynchronously, and the status of the query is polled. This is due to the fact that SQL queries can sometimes take a while to complete. This is entirely dependent on the complexity of your query and the amount of data scanned. Performance is _largely_ correlated with the amount of data scanned. You can reduce the execution time by following our [best practices](sql-querying.md#best-practices).

### Data Availability

Timber's S3 / SQL Querying pipeline flushes data on 15 minute intervals, meaning data can be delayed up to 15 minutes before being made available for SQL querying. If you need real-time access please see the [Live Tailing & Search feature](live-tailing.md).

## Limitations

1. SQL queries are read-only, only `SELECT` queries are allowed.
2. Execution time cannot exceed 10 minutes. Beyond this the query will be canceled.
3. Only 10 X your [billing plan's volume](account-management/billing.md#volume) can be scanned within a given billing period. For example, if you have a 10gb [volume billing plan](account-management/billing.md#volume), you can only execute up to 100GB in cumulative data scanned within a [single billing period](account-management/billing.md#billing-period).Note that because of [the way Timber stores your log data](sql-querying.md#performance), SQL queries scan significantly less data than the total amount of data contained in your log lines. See the [Usage Calculation section](sql-querying.md#usage-calculation) for more information.

Please contact support if you want to inquire about a limit increase.

