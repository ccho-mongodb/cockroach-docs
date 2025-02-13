---
title: cockroach debug tsdump
summary: Learn the commands for collecting timeseries debug information from all nodes in your cluster.
toc: true
key: debug-tsdump.html
docs_area: reference.cli
---

The `cockroach debug tsdump` [command]({% link {{ page.version.version }}/cockroach-commands.md %}) connects to your cluster and collects timeseries diagnostic data from each active node (inactive nodes are not included). This includes both current and historical runtime metrics for your cluster, including those exposed in the [DB Console Metrics]({% link {{ page.version.version }}/ui-overview-dashboard.md %}) pages as well as internal metrics.

`cockroach debug tsdump` is mostly used in tandem with the [`cockroach debug zip`]({% link {{ page.version.version }}/cockroach-debug-zip.md %}) command to gather diagnostic data during escalations to Cockroach Labs support. Follow the steps in this procedure to gather and prepare the timeseries diagnostic data and prepare it for transit to Cockroach Labs.

## Subcommands

{% include {{ page.version.version }}/misc/debug-subcommands.md %}

## Synopsis

~~~ shell
$ cockroach debug tsdump {flags} > {dump file destination} 
~~~

{{site.data.alerts.callout_info}}
The following [flags](#flags) must apply to an active CockroachDB node. If no nodes are live, you must [start at least one node]({% link {{ page.version.version }}/cockroach-start.md %}).
{{site.data.alerts.end}}

## Flags

The `debug tsdump` subcommand supports the following [general-use](#general), [client connection](#client-connection), and [logging](#logging) flags.

### General

Flag | Description
-----|-----------
`--format` | The output format to write the collected diagnostic data. Valid options are `text`, `csv`, `tsv`, `raw`.<br><br>**Default:** `text`
`--from` | The oldest timestamp to include (inclusive), in the format `YYYY-MM-DD [HH:MM[:SS]]`.<br><br>**Default:** `0001-01-01 00:00:00`
`--to` | The newest timestamp to include (inclusive), in the format `YYYY-MM-DD [HH:MM[:SS]]`.<br><br>**Default:** Current timestamp plus 29 hours

#### Available resolutions

A `debug tsdump` includes all the available resolutions (10s and 30m) of the raw time series values in the time range set using the `--from` and `--to` flags:

- Time series values will be dumped with a 10 seconds resolution if the supplied time range is within the 10s TTL set by the  [`timeseries.storage.resolution_10s.ttl`]({% link {{ page.version.version }}/cluster-settings.md %}#setting-timeseries-storage-resolution-10s-ttl) cluster setting (default: 10 days).
- Time series values will be dumped with a 30 minutes resolution if the supplied time range is older than the 10s TTL back to the 30m TTL set by the [`timeseries.storage.resolution_30m.ttl`]({% link {{ page.version.version }}/cluster-settings.md %}#setting-timeseries-storage-resolution-30m-ttl) cluster setting (default: 90 days).

For example, if today was `2024-11-15` and `--to` was set to the current timestamp (`2024-11-15`), and `--from` was set to 15 days before the current timestamp (`2024-11-01`), the `debug tsdump` would include:

- Time series values with a 10 seconds resolution for `2024-11-05` to `2024-11-15` (within the 10s TTL of 10 days).
- Time series values with a 30 minutes resolution for `2024-11-01` to `2024-11-04` (outside the 10s TTL of 10 days, but within the 30s TTL of 90 days). 

### Client connection

Flag | Description
-----|------------
`--cert-principal-map` | A comma-separated list of `<cert-principal>:<db-principal>` mappings. This allows mapping the principal in a cert to a DB principal such as `node` or `root` or any SQL user. This is intended for use in situations where the certificate management system places restrictions on the `Subject.CommonName` or `SubjectAlternateName` fields in the certificate (e.g., disallowing a `CommonName` like `node` or `root`). If multiple mappings are provided for the same `<cert-principal>`, the last one specified in the list takes precedence. A principal not specified in the map is passed through as-is via the identity function. A cert is allowed to authenticate a DB principal if the DB principal name is contained in the mapped `CommonName` or DNS-type `SubjectAlternateName` fields.
`--certs-dir` | The path to the [certificate directory]({% link {{ page.version.version }}/cockroach-cert.md %}) containing the CA and client certificates and client key.<br><br>**Env Variable:** `COCKROACH_CERTS_DIR`<br>**Default:** `${HOME}/.cockroach-certs/`
`--cluster-name` | The cluster name to use to verify the cluster's identity. If the cluster has a cluster name, you must include this flag. For more information, see [`cockroach start`]({% link {{ page.version.version }}/cockroach-start.md %}#general).
`--disable-cluster-name-verification` | Disables the cluster name check for this command. This flag must be paired with `--cluster-name`. For more information, see [`cockroach start`]({% link {{ page.version.version }}/cockroach-start.md %}#general).
`--host` | The server host and port number to connect to. This can be the address of any node in the cluster.<br><br>**Env Variable:** `COCKROACH_HOST`<br>**Default:** `localhost:26257`
`--insecure` | Use an insecure connection.<br><br>**Env Variable:** `COCKROACH_INSECURE`<br>**Default:** `false`
<a name="sql-flag-url"></a> `--url` | A [connection URL]({% link {{ page.version.version }}/connection-parameters.md %}#connect-using-a-url) to use instead of the other arguments. To convert a connection URL to the syntax that works with your client driver, run [`cockroach convert-url`]({% link {{ page.version.version }}/connection-parameters.md %}#convert-a-url-for-different-drivers).<br><br>**Env Variable:** `COCKROACH_URL`<br>**Default:** no URL

### Logging

By default, this command logs messages to `stdout`. If you need to troubleshoot this command's behavior, you can [customize its logging behavior]({% link {{ page.version.version }}/configure-logs.md %}).

## Examples

### Generate a tsdump `gob` file

Generate the tsdump `gob` file for an insecure CockroachDB cluster:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach debug tsdump --format=raw --insecure > tsdump.gob
~~~

Generate the tsdump `gob` file for a secure CockroachDB cluster:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach debug tsdump --format=raw --certs-dir=${HOME}/.cockroach-certs/ > tsdump.gob
~~~

{{site.data.alerts.callout_info}}
Secure examples assume you have the appropriate certificates in the default certificate directory, `${HOME}/.cockroach-certs/`. See the [`cockroach cert`]({% link {{ page.version.version }}/cockroach-cert.md %}) documentation for more information.
{{site.data.alerts.end}}

### Generate a tsdump `gob` file and compress using `gzip`

Generate a tsdump `gob` file for an insecure CockroachDB cluster, and compress using `gzip` in preparation to send to Cockroach Labs for troubleshooting. Your server must have `gzip` installed:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach debug tsdump --format=raw --insecure > tsdump.gob
gzip tsdump.gob
~~~

Generate a tsdump `gob` file for a secure CockroachDB cluster, and compress using `gzip` in preparation to send to Cockroach Labs for troubleshooting. Your server must have `gzip` installed:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach debug tsdump --format=raw --certs-dir=${HOME}/.cockroach-certs/ > tsdump.gob
gzip tsdump.gob
~~~

{{site.data.alerts.callout_info}}
Secure examples assume you have the appropriate certificates in the default certificate directory, `${HOME}/.cockroach-certs/`. See the [`cockroach cert`]({% link {{ page.version.version }}/cockroach-cert.md %}) documentation for more information.
{{site.data.alerts.end}}

### Generate a tsdump `gob` file with a custom timestamp range

Generate a tsdump `gob` file specifying a custom timestamp range to limit the data collection to a specific interval. This is useful for reducing the size of the resulting `gob` file if the data needed to troubleshoot falls within a known timestamp range:

~~~ shell
$ cockroach debug tsdump --format=raw --from='2023-01-10 01:00:00' --to='2023-01-20 23:59:59' > tsdump.gob
~~~

## See also

- [File an Issue]({% link {{ page.version.version }}/file-an-issue.md %})
- [`cockroach` Commands Overview]({% link {{ page.version.version }}/cockroach-commands.md %})
- [Troubleshooting Overview]({% link {{ page.version.version }}/troubleshooting-overview.md %})
