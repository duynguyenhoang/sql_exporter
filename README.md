# Prometheus SQL Exporter [![Go](https://github.com/burningalchemist/sql_exporter/workflows/Go/badge.svg)](https://github.com/burningalchemist/sql_exporter/actions?query=workflow%3AGo) [![Go Report Card](https://goreportcard.com/badge/github.com/burningalchemist/sql_exporter)](https://goreportcard.com/report/github.com/burningalchemist/sql_exporter) [![Docker Pulls](https://img.shields.io/docker/pulls/burningalchemist/sql_exporter)](https://hub.docker.com/r/burningalchemist/sql_exporter) ![Downloads](https://img.shields.io/github/downloads/burningalchemist/sql_exporter/total)

This is a permanent fork of Database agnostic SQL exporter for [Prometheus](https://prometheus.io) created by
[@free](https://github.com/free/sql_exporter).

## Overview

SQL Exporter is a configuration driven exporter that exposes metrics gathered from DBMSs, for use by the Prometheus
monitoring system. Out of the box, it provides support for the following databases and compatible interfaces:

- MySQL
- PostgreSQL
- Microsoft SQL Server
- Clickhouse
- Snowflake
- Vertica

In fact, any DBMS for which a Go driver is available may be monitored after rebuilding the binary with the DBMS driver
included.

The collected metrics and the queries that produce them are entirely configuration defined. SQL queries are grouped into
collectors -- logical groups of queries, e.g. *query stats* or *I/O stats*, mapped to the metrics they populate.
Collectors may be DBMS-specific (e.g. *MySQL InnoDB stats*) or custom, deployment specific (e.g. *pricing data
freshness*). This means you can quickly and easily set up custom collectors to measure data quality, whatever that might
mean in your specific case.

Per the Prometheus philosophy, scrapes are synchronous (metrics are collected on every `/metrics` poll) but, in order to
keep load at reasonable levels, minimum collection intervals may optionally be set per collector, producing cached
metrics when queried more frequently than the configured interval.

## Usage

Get Prometheus SQL Exporter, either as a [packaged release](https://github.com/burningalchemist/sql_exporter/releases/latest),
as a [Docker image](https://hub.docker.com/r/burningalchemist/sql_exporter).

Use the `-help` flag to get help information.

```shell
$ ./sql_exporter -help
Usage of ./sql_exporter:
  -config.file string
      SQL Exporter configuration file name. (default "sql_exporter.yml")
  -web.listen-address string
      Address to listen on for web interface and telemetry. (default ":9399")
  -web.metrics-path string
      Path under which to expose metrics. (default "/metrics")
  [...]
```

## Build

Prerequisites:

- Go Compiler
- GNU Make

By default we produce a binary with all the supported drivers with the following command:

```shell
make build
```

It's also possible to reduce the size of the binary by only including specific set of drivers like Postgres, MySQL and
MSSQL. In this case we need to update `drivers.go`. To avoid manual manipulation there is a helper code generator
available, so we can run the following commands:

```shell
make drivers-minimal
make build
```

The first command will regenerate `drivers.go` file with a minimal set of imported drivers using `drivers_gen.go`.

Running `make drivers-all` will regenerate driver set back to the current defaults.

Feel free to revisit and add more drivers as required. There's also the `custom` list that allows managing a separate
list of drivers for special needs.

## Run as a Windows service

If you run SQL Exporter from Windows, it might come in handy to register it as a service to avoid interactive sessions.
It is **important** to define `--config.file` parameter to load the configuration file. The other settings can be added
as well. The registration itself is performed with Powershell or CMD (make sure you run it as Administrator):

Powershell:

```powershell
New-Service -name "SqlExporterSvc" `
-BinaryPathName "%SQL_EXPORTER_PATH%\sql_exporter.exe --config.file %SQL_EXPORTER_PATH%\sql_exporter.yml" `
-StartupType Automatic `
-DisplayName "Prometheus SQL Exporter"
```

CMD:

```shell
sc.exe create SqlExporterSvc binPath= "%SQL_EXPORTER_PATH%\sql_exporter.exe --config.file %SQL_EXPORTER_PATH%\sql_exporter.yml" start= auto
```

`%SQL_EXPORTER_PATH%` is a path to the SQL Exporter binary executable. This document assumes that configuration files
are in the same location.

## Configuration

SQL Exporter is deployed alongside the DB server it collects metrics from. If both the exporter and the DB
server are on the same host, they will share the same failure domain: they will usually be either both up and running
or both down. When the database is unreachable, `/metrics` responds with HTTP code 500 Internal Server Error, causing
Prometheus to record `up=0` for that scrape. Only metrics defined by collectors are exported on the `/metrics` endpoint.
SQL Exporter process metrics are exported at `/sql_exporter_metrics`.

The configuration examples listed here only cover the core elements. For a comprehensive and comprehensively documented
configuration file check out
[`documentation/sql_exporter.yml`](https://github.com/burningalchemist/sql_exporter/tree/master/documentation/sql_exporter.yml).
You will find ready to use "standard" DBMS-specific collector definitions in the
[`examples`](https://github.com/burningalchemist/sql_exporter/tree/master/examples) directory. You may contribute your
own collector definitions and metric additions if you think they could be more widely useful, even if they are merely
different takes on already covered DBMSs.

**`./sql_exporter.yml`**

```yaml
# Global settings and defaults.
global:
  # Subtracted from Prometheus' scrape_timeout to give us some headroom and prevent Prometheus from
  # timing out first.
  scrape_timeout_offset: 500ms
  # Minimum interval between collector runs: by default (0s) collectors are executed on every scrape.
  min_interval: 0s
  # Maximum number of open connections to any one target. Metric queries will run concurrently on
  # multiple connections.
  max_connections: 3
  # Maximum number of idle connections to any one target.
  max_idle_connections: 3
  # Maximum amount of time a connection may be reused to any one target. Infinite by default.
  max_connection_lifetime: 10m

# The target to monitor and the list of collectors to execute on it.
target:
  # Target name (optional). Setting this field enables extra metrics e.g. `up` and `scrape_duration` with
  # the `target` label that are always returned on a scrape.
  name: "prices_db"
  # Data source name always has a URI schema that matches the driver name. In some cases (e.g. MySQL)
  # the schema gets dropped or replaced to match the driver expected DSN format.
  data_source_name: 'sqlserver://prom_user:prom_password@dbserver1.example.com:1433'

  # Collectors (referenced by name) to execute on the target.
  # Glob patterns are supported (see <https://pkg.go.dev/path/filepath#Match> for syntax).
  collectors: [pricing_data_freshness, pricing_*]

  # In case you need to connect to a backend that only responds to a limited set of commands (e.g. pgbouncer) or
  # a data warehouse you don't want to keep online all the time (due to the extra cost), you might want to disable `ping`
  # enable_ping: true

# Collector definition files.
# Glob patterns are supported (see <https://pkg.go.dev/path/filepath#Match> for syntax).
collector_files:
  - "*.collector.yml"
```

**NOTE:** The `collectors` and `collector_files` configurations support [Glob pattern matching](https://pkg.go.dev/path/filepath#Match).
To match names with literal pattern terms in them, e.g. `collector_*1*`, these must be escaped: `collector_\*1\*`.

### Collectors

Collectors may be defined inline, in the exporter configuration file, under `collectors`, or they may be defined in
separate files and referenced in the exporter configuration by name, making them easy to share and reuse.

The collector definition below generates gauge metrics of the form `pricing_update_time{market="US"}`.

**`./pricing_data_freshness.collector.yml`**

```yaml
# This collector will be referenced in the exporter configuration as `pricing_data_freshness`.
collector_name: pricing_data_freshness

# A Prometheus metric with (optional) additional labels, value and labels populated from one query.
metrics:
  - metric_name: pricing_update_time
    type: gauge
    help: 'Time when prices for a market were last updated.'
    key_labels:
      # Populated from the `market` column of each row.
      - Market
    static_labels:
      # Arbitrary key/value pair
      portfolio: income
    values: [LastUpdateTime]
    # Static metric value (optional). Useful in case we are interested in string data (key_labels) only. It's mutually
    # exclusive with `values` field.
    # static_value: 1
    # Timestamp value (optional). Should point at the existing column containing valid timestamps to return a metric
    # with an explicit timestamp.
    # timestamp_value: CreatedAt
    query: |
      SELECT Market, max(UpdateTime) AS LastUpdateTime
      FROM MarketPrices
      GROUP BY Market
```

### Data Source Names (DSN)

To keep things simple and yet allow fully configurable database connections, SQL Exporter uses DSNs (like
`sqlserver://prom_user:prom_password@dbserver1.example.com:1433`) to refer to database instances.

Since v0.9.0 `sql_exporter` relies on `github.com/xo/dburl` package for parsing Data Source Names (DSN).
This can potentially affect your connection to certain databases like MySQL, so you might want to adjust your connection
string accordingly:

```plaintext
mysql://user:pass@localhost/dbname - for TCP connection
mysql:/var/run/mysqld/mysqld.sock - for Unix socket connection
```

If your DSN contains special characters in any part of your connection string (including passwords), you might need to
apply [URL encoding](https://en.wikipedia.org/wiki/URL_encoding#Reserved_characters) (percent-encoding) to them.
For example, `p@$$w0rd#abc` then becomes `p%40%24%24w0rd%23abc`.

For additional details please refer to [xo/dburl](https://github.com/xo/dburl) documentation.

#### Using AWS Secrets Manager

If the database runs on AWS EC2 instance, this is a secure option to store the DSN without having it in
the configuration file. To use this option:

- Create a [secret](https://docs.aws.amazon.com/secretsmanager/latest/userguide/manage_create-basic-secret.html) in
  key/value pairs format, specify Key `data_source_name` and then for Value enter the DSN value.
  For the secret name, enter a name for your secret, and pass that name in the configuration file as a value for
  `aws_secret_name` item under `target`. Secret json example:

```json
{
  "data_source_name": "sqlserver://prom_user:prom_password@dbserver1.example.com:1433"
}
```

- Configuration file example:

```yaml
...
target:
  aws_secret_name: '<AWS_SECRET_NAME>'
...
```

- Allow read-only access from EC2 IAM role to the secret by attaching a [resource-based
policy](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_resource-based-policies.html) to
the secret. Policy example:

```json
{
  "Version" : "2012-10-17",
  "Statement" : [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/EC2RoleToAccessSecrets"},
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*",
    }
  ]
}
```

Currently, AWS Secret Manager integration is only available for a single target configuration.

### Multiple database connections

It is possible to run a single exporter instance against multiple database connections. In this case we need to
configure `jobs` list instead of the `target` section as in the following example:

```yaml
jobs:
  - job_name: db_targets
    collectors: [pricing_data_freshness, pricing_*]
    enable_ping: true # Optional, true by default. Set to `false` in case you connect to pgbouncer or a data warehouse
    static_configs:
        - targets:
            pg1: 'pg://db1@127.0.0.1:25432/postgres?sslmode=disable'
            pg2: 'pg://db2@127.0.0.1:25432/testdb?sslmode=disable'
```

, where DSN strings are assigned to the arbitrary instance names (i.e. pg1 and pg2).

We can also define multiple jobs to run different collectors against different target sets.

### TLS and Basic Authentication

SQL Exporter supports TLS and Basic Authentication. This enables better control of the various HTTP endpoints.

To use TLS and/or Basic Authentication, you need to pass a configuration file using the `--web.config.file` parameter.
The format of the file is described in the
[exporter-toolkit](https://github.com/prometheus/exporter-toolkit/blob/master/docs/web-configuration.md) repository.

## Why It Exists

SQL Exporter started off as an exporter for Microsoft SQL Server, for which no reliable exporters exist. But what is
the point of a configuration driven SQL exporter, if you're going to use it along with 2 more exporters with wholly
different world views and configurations, because you also have MySQL and PostgreSQL instances to monitor?

A couple of alternative database agnostic exporters are available:

- [justwatchcom/sql_exporter](https://github.com/justwatchcom/sql_exporter);
- [chop-dbhi/prometheus-sql](https://github.com/chop-dbhi/prometheus-sql).

However, they both do the collection at fixed intervals, independent of Prometheus scrapes. This is partly a
philosophical issue, but practical issues are not all that difficult to imagine:

- jitter;
- duplicate data points;
- collected but not scraped data points.

The control they provide over which labels get applied is limited, and the base label set spammy. And finally,
configurations are not easily reused without copy-pasting and editing across jobs and instances.
