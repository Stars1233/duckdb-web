---
layout: docu
title: Reference
---

This page lists every function, setting, and log type exposed by the Quack extension. For a tour of the protocol, start with the [Overview]({% link docs/current/quack/overview.md %}).

## Function Reference

### Server Management

<div class="monospace_table"></div>

| Function | Description |
|----------|-------------|
| `quack_serve(uri, allow_other_hostname := false, disable_ssl := false)` | Start a server on `uri`. Localhost-only by default. Returns listen URI, URL, and an auth token. |
| `rpc_stop(uri)`                                           | Stop the server listening on `uri`. |
| `quack_identify(name, provider, hostname, region, meta)`  | Set this node's `whoami` identity fields. Any subset can be supplied. |
| `whoami()`                                                | Table macro returning identity + runtime info for the current node. |

### Client Queries

<div class="monospace_table"></div>

| Function | Description |
|----------|-------------|
| `quack_query(uri, query, disable_ssl := false)`  | Run `query` on remote `uri`, stream result back. |
| `quack_query_by_name(catalog, query)`            | Run `query` against an already-attached Quack catalog (used by `⟨catalog⟩.query()`). |

### Utility

<div class="monospace_table"></div>

| Function | Description |
|----------|-------------|
| `quack_uri_parser(uri, ssl)`             | Parse a Quack URI into `{host, port, ipv6, ssl, url}`. |
| `quack_check_token(sid, token)`          | Default authentication callback; compares against `rpc_default_token`. |
| `quack_nop_authorization(sid, query)`    | Default authorization callback; always allows. |

### `ATTACH` Options

<div class="monospace_table"></div>

| Option        | Type    | Default                          | Description |
|---------------|---------|----------------------------------|-------------|
| `disable_ssl` | BOOLEAN | `true` for local, else `false`   | Force the client transport. Local URIs default to plain HTTP. |
| `type`        | VARCHAR | inferred                         | Pin the secret type used for token resolution (e.g. `quack`). |

## Settings

All settings are regular DuckDB session / global options. Set with `SET ⟨name⟩ = ⟨value⟩` or `SET GLOBAL`.

### Authentication / Authorization

<div class="monospace_table"></div>

| Setting                       | Type    | Default                   | Description |
|-------------------------------|---------|---------------------------|-------------|
| `quack_authentication_function` | VARCHAR | `quack_check_token`        | Name of a 2-arg scalar function `(sid, token) -> BOOLEAN` used by the server to authenticate clients. |
| `quack_authorization_function`  | VARCHAR | `quack_nop_authorization`  | Name of a 2-arg scalar function `(sid, query) -> BOOLEAN` used by the server to authorize each query. |
| `rpc_default_token`             | VARCHAR | *(unset)*                  | Shared secret. The server generates one on `quack_serve`; clients use it as fallback if no quack secret matches. |

You can plug in your own auth by creating any scalar function with the expected signature and pointing the setting at it. See [Security]({% link docs/current/quack/security.md %}) for examples.

### FETCH Batching (Server-Side)

The server batches multiple `DataChunk`s into each `FETCH` response to reduce per-chunk overhead.

<div class="monospace_table"></div>

| Setting                    | Type    | Default | Description |
|----------------------------|---------|---------|-------------|
| `quack_fetch_batch_chunks` | UBIGINT | `12`    | Max `DataChunk`s shipped per FETCH response. |

### Node Identity

These settings back the `whoami()` macro. `quack_identify(...)` is sugar that updates them.

<div class="monospace_table"></div>

| Setting              | Type    | Default                              | Description |
|----------------------|---------|--------------------------------------|-------------|
| `whoami_name`        | VARCHAR | *(empty)*                            | Human-readable node name. |
| `whoami_provider`    | VARCHAR | *(empty)*                            | Deployment provider (`ec2`, `docker`, `local`, ...). |
| `whoami_hostname`    | VARCHAR | *(empty)*                            | Network hostname / public address. |
| `whoami_region`      | VARCHAR | *(empty)*                            | Deployment region. |
| `whoami_started_at`  | VARCHAR | *(empty)*                            | Node start time (ISO-8601 timestamp). Anchors `uptime`. |
| `whoami_meta`        | VARCHAR | `{}`                                 | Provider-specific metadata as JSON. |
| `quack_loaded_at_us` | BIGINT  | epoch microseconds at extension load | Fallback uptime anchor when `whoami_started_at` is empty. |

## Logging

Two log types are registered by the extension. Enable them to debug connectivity or measure request timing.

### `Quack` Log

Structured log of every Quack message (both client- and server-side):

```sql
CALL enable_logging('Quack');

FROM quack_query('quack:localhost', 'SELECT 42');

SELECT * FROM duckdb_logs_parsed('Quack');
```

Fields on each entry:

<div class="monospace_table"></div>

| Field               | Description |
|---------------------|-------------|
| `message_type`      | Request type: `PREPARE_REQUEST`, `FETCH_REQUEST`, etc. |
| `quack_connection_id` | Server-issued connection id (stable across requests in one ATTACH). |
| `client_query_id`   | Monotonic id assigned by the client; correlates client / server logs. |
| `query`             | SQL payload for `PREPARE_REQUEST`s. |
| `server`            | HTTP URL on client-side logs; NULL on server-side logs. |
| `duration_ms`       | Round-trip time (client) or handling time (server). |
| `response_type`     | Response type, or `ERROR`. |
| `error`             | Error message if the request failed. |

To correlate a client request with its server-side handling, join on `(quack_connection_id, client_query_id)`.

### `HTTP` Log

The underlying HTTP transport can be logged separately:

```sql
CALL enable_logging('HTTP');
FROM quack_query('quack:localhost', 'SELECT 1');
SELECT request.type, request.url, response.status
FROM duckdb_logs_parsed('HTTP');
```

Requests are `POST`s to a `/quack` endpoint.

### Persisting Logs for Querying

`duckdb_logs_parsed` reads from DuckDB's in-memory log buffer. For non-trivial sessions you'll want to persist logs:

```sql
CALL enable_logging(
    'Quack',
    storage => 'file',
    storage_config => {'path': '/tmp/duckdb-rpc-logs'}
);
```

Use `CALL truncate_duckdb_logs();` to clear between runs and `CALL disable_logging();` to turn logging off.
