---
layout: post
title: "Building an Open Service Broker in Rust: A Weekend Engineering Journey"
date: 2026-05-21 07:00:00 +0700
categories: rust backend architecture
tags: [rust, axum, open-service-broker, kubernetes, cloud-foundry, postgres, docker]
description: "Building a production-shape Open Service Broker (OSB) API in Rust from scratch with axum, including JSON-Schema validation, async operations, Postgres-backed storage, Docker, and a CI pipeline that catches real bugs."
---

I spent a weekend building an [Open Service Broker (OSB) API](https://github.com/openservicebrokerapi/servicebroker) implementation in Rust. The result lives at [github.com/huzairuje/open-service-broker-rust](https://github.com/huzairuje/open-service-broker-rust). This post is the engineering narrative: the design choices, the trade-offs, the bugs my own conformance suite caught, and what I'd do differently next time.

## What is the Open Service Broker?

OSB is the standard that lets cloud platforms like **Kubernetes** (via Service Catalog or Crossplane), **Cloud Foundry**, and **OpenShift** provision external services on behalf of users. When a developer runs `cf create-service postgres free my-db` or applies a `ServiceInstance` in Kubernetes, the platform talks to a **broker** that exposes a fixed set of REST endpoints:

- `GET /v2/catalog` — what does this broker offer?
- `PUT /v2/service_instances/:id` — provision an instance (e.g., create a database)
- `PUT /v2/service_instances/:id/service_bindings/:bid` — bind, hand back credentials
- `DELETE` variants — unbind and deprovision

The wire format is plain HTTPS + JSON. There's no exotic protocol, just a precise contract about status codes, idempotency, and headers.

## Why Rust + axum

I picked Rust for three reasons: precise control over the JSON shape (the OSB spec is picky about field names), a fast async runtime, and the discipline the type system imposes for a project I want to keep clean.

For the HTTP framework I went with [axum](https://github.com/tokio-rs/axum). It's tower-compatible, async-native, and its extractor pattern reads naturally:

```rust
pub async fn provision(
    State(state): State<AppState>,
    Path(instance_id): Path<String>,
    Query(q): Query<ProvisionQuery>,
    Json(req): Json<ProvisionRequest>,
) -> BrokerResult<(StatusCode, Json<ProvisionResponse>)> {
    // ...
}
```

Compare that to wiring path params, query strings, and JSON bodies in plain `hyper` or even `actix-web`. axum keeps each handler a pure function over typed inputs.

## The Architecture

I aimed for a layout where each module has one job:

```
src/
├── main.rs              # entry point
├── lib.rs               # router wiring
├── config.rs            # env-driven config
├── error.rs             # BrokerError -> HTTP mapping
├── auth.rs              # basic-auth + version check middleware
├── broker.rs            # catalog + storage + ops glue
├── catalog_loader.rs    # JSON/YAML loader + built-in sample
├── operations.rs        # async operation tracker
├── validation.rs        # JSON-Schema validation
├── models/              # OSB request/response DTOs
├── handlers/            # one module per OSB resource
└── storage/             # Storage trait + memory + postgres
```

The `Storage` trait is the boundary that lets me swap backends:

```rust
#[async_trait]
pub trait Storage: Send + Sync {
    async fn put_instance(&self, instance: ServiceInstance) -> BrokerResult<()>;
    async fn get_instance(&self, id: &str) -> BrokerResult<Option<ServiceInstance>>;
    async fn delete_instance(&self, id: &str) -> BrokerResult<bool>;

    async fn put_binding(&self, binding: ServiceBinding) -> BrokerResult<()>;
    async fn get_binding(&self, id: &str) -> BrokerResult<Option<ServiceBinding>>;
    async fn delete_binding(&self, id: &str) -> BrokerResult<bool>;
}
```

Two implementations ship: a `DashMap`-backed in-memory store (default, perfect for tests) and a `sqlx`-backed Postgres store behind a `postgres` cargo feature.

## Idempotency Is the Spec

The OSB spec is strict about idempotency on `PUT`. The broker has to track three cases:

| Situation | Status |
|-----------|--------|
| Resource doesn't exist | `201 Created` |
| Resource exists with same params | `200 OK` |
| Resource exists with different params | `409 Conflict` |

My implementation looks like this:

```rust
if let Some(existing) = state.broker.storage().get_instance(&instance_id).await? {
    return if existing.service_id == req.service_id
        && existing.plan_id == req.plan_id
    {
        Ok((StatusCode::OK, Json(ProvisionResponse::default())))
    } else {
        Err(BrokerError::Conflict(format!(
            "instance {instance_id} already exists with different params"
        )))
    };
}
```

Same pattern on `DELETE`: first delete returns `200`, second returns `410 Gone`. These look like trivial checks, but skip them and you'll fail every conformance suite in the wild.

## JSON-Schema Validation Without Surprises

The OSB spec lets a plan declare JSON-Schema constraints on the `parameters` clients send during provision/update/bind. My catalog example:

```json
"schemas": {
  "service_instance": {
    "create": {
      "parameters": {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "properties": {
          "db_name": { "type": "string", "minLength": 1, "maxLength": 64 }
        }
      }
    }
  }
}
```

I added a `validation.rs` module that compiles the schema once per request (the [`jsonschema`](https://crates.io/crates/jsonschema) crate handles draft-07) and rejects bad input with `400 Bad Request`.

The first version had a subtle bug — I'll come back to it.

## Async Operations: The Real OSB Lifecycle

Real backends don't provision instantly. Spinning up a database can take minutes. The OSB spec accommodates this with a **202 Accepted** response containing an `operation` token, plus a `last_operation` polling endpoint.

I built an in-memory `OperationTracker`:

```rust
pub struct OperationTracker {
    ops: DashMap<String, Operation>,
}

impl OperationTracker {
    pub fn start(
        &self,
        resource_kind: ResourceKind,
        resource_id: &str,
        kind: OperationKind,
    ) -> String {
        let id = Uuid::new_v4().to_string();
        self.ops.insert(id.clone(), Operation {
            id: id.clone(),
            state: OperationState::InProgress,
            // ...
        });
        id
    }
}
```

When async mode is enabled (`BROKER_ASYNC_OP_MILLIS > 0` and the platform sends `accepts_incomplete=true`), the handler returns `202` immediately and spawns a `tokio` task that completes the work after the configured delay. A `last_operation` poll first checks the tracker, then falls back to storage existence — handy after a broker restart.

## Docker and Postgres

I added a multi-stage `Dockerfile` and a `docker-compose.yml` that brings up Postgres + the broker:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U broker -d broker"]
  broker:
    build: .
    depends_on:
      postgres: { condition: service_healthy }
    environment:
      BROKER_STORAGE: postgres
      DATABASE_URL: postgres://broker:broker@postgres:5432/broker
```

The Postgres backend uses `JSONB` columns for `parameters`, `context`, and `credentials`, with `ON CONFLICT DO UPDATE` for idempotent writes. Schema is applied on startup via a tiny migrate function.

This is where the project hit its first runtime bug.

## Bugs My Own Tooling Caught

I wrote a bash conformance script (`tests/conformance/run.sh`) that walks the OSB lifecycle and asserts every status code along the way. It's a lighter-weight alternative to the official Kotlin checker. Running it against the Docker stack uncovered three real bugs in sequence.

### Bug 1: Postgres rejects multi-statement prepared statements

My first migrate function did this:

```rust
sqlx::query(SCHEMA).execute(&pool).await?;
```

Where `SCHEMA` was two `CREATE TABLE` statements separated by `;`. The container exited with:

```
postgres migrate: error returned from database:
cannot insert multiple commands into a prepared statement
```

The reason: `sqlx::query` always uses prepared statements, and Postgres doesn't allow multiple commands per prepared statement. Fix: split into a slice of single-statement strings and loop.

```rust
const SCHEMA_STATEMENTS: &[&str] = &[
    "CREATE TABLE IF NOT EXISTS service_instances (...)",
    "CREATE TABLE IF NOT EXISTS service_bindings (...)",
];

for stmt in SCHEMA_STATEMENTS {
    sqlx::query(stmt).execute(&self.pool).await?;
}
```

### Bug 2: `OperationState` serialization

The OSB spec requires the `state` field in `last_operation` responses to be one of `"in progress"`, `"succeeded"`, or `"failed"`. Note that **space** in `"in progress"`. My initial enum:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum OperationState {
    InProgress,
    Succeeded,
    Failed,
}
```

`#[serde(rename_all = "lowercase")]` produced `"inprogress"`, not `"in progress"`. The async test caught this — I'd built it expecting `"in progress"` and it failed loudly. Fix: explicit `rename` per variant.

```rust
#[derive(Serialize, Deserialize)]
pub enum OperationState {
    #[serde(rename = "in progress")]
    InProgress,
    #[serde(rename = "succeeded")]
    Succeeded,
    #[serde(rename = "failed")]
    Failed,
}
```

This is a great example of **why specs matter character by character**. A space cost me an hour the first time I shipped a similar bug at work years ago. This time, the test caught it before the broker ever saw a real client.

### Bug 3: Validating absent parameters

The spec treats `parameters` as optional. My validation code did this:

```rust
let null = Value::Null;
let to_check = parameters.unwrap_or(&null);
compiled.validate(to_check)?;
```

That validates `null` against the schema when no parameters are sent. With `"type": "object"` in the schema, every clean provision call rejected:

```
400 Bad Request: parameters failed schema validation:
null is not of type "object"
```

The conformance script caught this on the very first provision step. Fix: skip validation entirely when parameters are absent.

```rust
let Some(to_check) = parameters else {
    return Ok(());
};
compiled.validate(to_check)?;
```

A good reminder that "validate everything" and "validate what was sent" are different commitments. The OSB spec contracts on the latter.

## The CI Pipeline

The repo has two GitHub Actions workflows:

**`ci.yml`** — runs on every push and PR:
- `cargo fmt --check`
- `cargo clippy` with both default and `postgres` features (with `-D warnings`)
- `cargo test --all-targets`
- A `conformance` job that brings up `docker compose`, waits for the broker, and runs `tests/conformance/run.sh`

**`release.yml`** — triggered on `v*.*.*` tag pushes:
- Linux x86_64 (with and without `postgres` feature)
- macOS Intel and Apple Silicon
- Windows x86_64
- FreeBSD x86_64 (built inside a `vmactions/freebsd-vm`)
- A `publish` job that downloads every artifact, attaches them all to a GitHub Release with auto-generated notes, and ships SHA-256 checksums

One gotcha worth flagging: GitHub deprecated the Intel macOS runner. Jobs targeting `macos-13` queue for 10-30+ minutes. The fix is to cross-compile the Intel binary from a `macos-latest` (Apple Silicon) runner with `--target x86_64-apple-darwin`. It's a one-line YAML change that cuts CI time by 3-5x.

## Numbers

Final scorecard for this weekend:

| Metric | Result |
|--------|--------|
| Files | 30 |
| Lines of Rust | ~2000 |
| `cargo test` | 9/9 pass |
| Conformance script (sync) | 17/17 |
| Conformance script (async) | 22/22 |
| `cargo clippy --all-targets -- -D warnings` | clean |
| `cargo clippy --features postgres -- -D warnings` | clean |
| Docker image size (release) | ~80 MB |
| Cold container start | <100 ms to "listening on..." |

## What I'd Do Differently

A few things I'd improve on the next pass:

- **Replace the in-memory `OperationTracker` with a Postgres table.** Right now operations are lost on restart. Persisting them would make the broker truly stateless.
- **Make provisioning actually do something.** This is a reference broker — it returns fake credentials. A real implementation would create a database, a role, a password, and hand back the connection URL.
- **Wire up tracing properly.** I'm using `tower_http::trace` plus `tracing_subscriber::fmt`, but a real production broker wants OpenTelemetry export, request IDs flowing through, and structured fields on every span.
- **Authentication beyond Basic Auth.** Basic Auth is the spec baseline, but a production deployment should layer mTLS or bearer tokens on top.
- **Migrations as files, not strings.** I inlined the schema for simplicity. `sqlx::migrate!` reading from `migrations/` is the right answer at scale.

## What This Project Taught Me

Three takeaways:

1. **Write your own conformance tests early.** I built `tests/conformance/run.sh` after the basic Rust tests. The bash script caught two production-grade bugs (the Postgres migration issue and the JSON-Schema null bug) that the Rust unit tests didn't exercise. End-to-end testing across the deployment boundary is non-negotiable.

2. **Specs are precise on purpose.** `"in progress"` vs `"inprogress"`. `200` vs `201` vs `202` vs `409` vs `410`. The OSB spec specifies these exactly, and the platforms that consume your broker depend on the exactness. Pay attention to every status code, every field name, every header.

3. **Rust pays off where you'd expect.** The `Storage` trait + feature-gated Postgres backend, the typed request/response DTOs that serialize 1:1 to the OSB spec, the `BrokerError` enum that maps each variant to a specific HTTP status — these are exactly the patterns where Rust's type system earns its keep. Compile-time safety on the boundary between HTTP and your domain.

## Try It

```bash
git clone https://github.com/huzairuje/open-service-broker-rust
cd open-service-broker-rust
docker compose up -d --build

curl -u admin:password \
  -H "X-Broker-API-Version: 2.17" \
  http://localhost:8080/v2/catalog
```

Or download a prebuilt binary from the [Releases page](https://github.com/huzairuje/open-service-broker-rust/releases).

The full source is on GitHub: [github.com/huzairuje/open-service-broker-rust](https://github.com/huzairuje/open-service-broker-rust). Issues and PRs welcome.
