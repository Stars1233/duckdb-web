---
github_repository: https://github.com/duckdb/duckdb-quack
layout: docu
redirect_from:
- /docs/quack
title: Quack Remote Protocol
---

The Quack extension turns a DuckDB instance into a server that other DuckDB instances (clients) can connect to over HTTP. Clients can either `ATTACH` to the server as a regular catalog or run one-off queries with the `quack_query` table function.

This page covers the protocol at a glance and walks through basic usage on both sides of the wire. For the full list of functions, settings, and logging knobs, see the [Reference]({% link docs/current/quack/reference.md %}). For deployment posture, TLS, and authentication / authorization, see [Security]({% link docs/current/quack/security.md %}).

> Warning Quack is under active development and the protocol, function names, settings, and defaults are still subject to change. This page documents the v1.5 release shipped via the `core_nightly` repository.

## Protocol

A few facts that explain how Quack behaves:

* **HTTP-based.** Quack messages travel over plain HTTP, or HTTPS via a reverse proxy (see [Security]({% link docs/current/quack/security.md %})). This means standard load balancers, firewalls, and reverse proxies handle Quack traffic the same way they handle any other HTTP service. There is no custom wire transport to operate.
* **Client-driven request / response.** Every interaction is initiated by the client. There is no server push.
* **`application/duckdb` serialization.** Requests and responses are encoded with DuckDB's internal serialization primitives (the same code path used by the Write-Ahead Log). This avoids round-tripping data through an interchange format and keeps complex types (nested, decimals, intervals, ...) lossless across the wire.
* **Single round-trip per query.** After the initial connection handshake, a query needs only one request / response pair. Large results stream back in chunks via follow-up `FETCH` requests, optionally on multiple threads in parallel.
* **Default port `1294`.** All URIs use the `quack:` scheme, e.g. `quack:hostname:port`.

## Server-Side Usage

### Starting a Server

A server is started from an existing DuckDB session. Everything that session can see (in-memory tables, attached files, schemas) becomes reachable over RPC.

```sql
-- Start listening on localhost (the only host allowed by default).
CALL quack_serve('quack:localhost');
```

`quack_serve` returns the listen URI, the HTTP URL, and, when the default authentication function is in use, an `auth_token` that clients need to connect. This token can also be set explicitly before starting (see [Security]({% link docs/current/quack/security.md %})).

By default the server refuses to bind anything other than a local hostname. To listen on an externally-reachable address, pass `allow_other_hostname => true`:

```sql
CALL quack_serve('quack:0.0.0.0:1294', allow_other_hostname => true);
```

When you do this you should front the server with a TLS-terminating reverse proxy. See [Securing Quack with a Reverse Proxy]({% link docs/current/quack/guides/reverse_proxy.md %}).

### URI Format

RPC endpoints use the `quack:` scheme. Some examples:

<div class="monospace_table"></div>

| URI                  | Host        | Port (default 1294) |
|----------------------|-------------|---------------------|
| `quack:localhost`    | `localhost` | `1294`              |
| `quack:myhost:9000`  | `myhost`    | `9000`              |
| `quack:127.0.0.1`    | `127.0.0.1` | `1294`              |
| `quack:[::1]:1234`   | `::1`       | `1234` (IPv6)       |
| `quack://localhost`  | `localhost` | `1294`              |

You can parse and validate a URI with the `quack_uri_parser(uri, ssl)` scalar function.

### Stopping a Server

```sql
CALL quack_stop('quack:localhost');
```

## Client-Side Usage

There are two ways to talk to an RPC server:

1. `quack_query(uri, query)`: one-shot, stateless query.
2. `ATTACH 'quack:host' AS name`: attach the remote as a full catalog.

The client picks plain HTTP automatically for local URIs (`localhost`, `127.0.0.1`, `::1`) and HTTPS otherwise. Either default can be overridden with `disable_ssl`.

### Stateless Queries with `quack_query`

Run any SQL against a remote server without mounting it:

```sql
-- Local: HTTP by default.
FROM quack_query('quack:localhost', 'SELECT 42');

-- Remote: HTTPS by default. Override if your proxy is plain HTTP.
FROM quack_query('quack:remote.com', 'SELECT 42', disable_ssl => true);
```

The query executes remotely and the result streams back. Errors from the server (parse errors, missing tables, etc.) surface as DuckDB errors on the client.

### Attaching a Remote Database

```sql
-- Local: HTTP by default.
ATTACH 'quack:localhost' AS quack;

-- Remote: HTTPS by default. Override on a plain-HTTP remote:
ATTACH 'quack:remote.com' AS quack (disable_ssl true);
```

Once attached, remote tables look local:

```sql
FROM quack.fuu;                          -- scan remote table
FROM quack.main.fuu WHERE col0 = 42;     -- filter evaluated client-side (see note below)
CREATE TABLE quack.t AS FROM range(10);  -- DDL on remote
INSERT INTO quack.t VALUES (42);         -- remote writes
BEGIN; ... COMMIT;                     -- transactions are forwarded
DETACH quack;
```

The attached catalog also exposes a `query` table macro for ad-hoc SQL scoped to that attachment:

```sql
FROM quack.query('SELECT 42');
```

> Warning Pushdown is currently not carried over to the server side: filters and projections evaluate on the client after the full result has streamed back.

### Authentication

Clients pass the auth token to the server in one of two ways: a `quack` secret scoped to the server URI, or an explicit `TOKEN` option on `ATTACH` / `quack_query`. See [Security]({% link docs/current/quack/security.md %}) for the full picture.

```sql
-- Recommended: scope a secret to the server URI.
CREATE SECRET (
    TYPE quack,
    TOKEN '⟨token-from-quack_serve⟩',
    SCOPE 'quack:localhost'
);

ATTACH 'quack:localhost' AS quack (TYPE quack);

-- Or pass the token directly (overrides any matching secret):
ATTACH 'quack:localhost' AS quack (TOKEN '⟨token-from-quack_serve⟩');
```

### Node Identity (`whoami`)

Each Quack node exposes a `whoami()` table macro that surfaces basic identity and runtime info, useful when proxying to a fleet of servers or when correlating logs:

```sql
FROM quack.query('FROM whoami()');
-- name | provider | hostname | region | uptime | ts_now | meta
```

Identity fields are populated either by setting `whoami_*` options directly or by calling the `quack_identify` helper:

```sql
CALL quack_identify(
    name => 'analytics-1',
    provider => 'ec2',
    region => 'eu-west-1',
    meta => '{"role": "worker"}'
);
```

`meta` is merged with auto-computed `duckdb_version` and `platform` keys; user-supplied keys win on conflict. `whoami_started_at` (an ISO-8601 timestamp) overrides the uptime anchor; otherwise uptime is measured from extension load.
