+++
date = '2026-03-21T07:55:33+03:00'
draft = false
title = 'Api Versioning: Part 2 - Request versioning'
+++

This post is a continuation of [Part 1 on response versioning](../api-versioning-part-1-response-versioning/).

The key insight is that request transformers run in reverse.

Clients can send an older request model, and the library upgrades it to the model the route handler expects.

As a small example, assume we have a user-creation endpoint: `/users`.

In version `0.5`, the endpoint accepts a `name` field:

```rust
#[derive(Debug, Serialize, Deserialize, VersionChange)]
#[description = "Clients before v1.0.0 send `name` instead of `full_name`"]
pub struct LegacyCreateUserRequestV0_5 {
    pub name: String,
}
```

In version `1.0`, it accepts a `full_name` field instead of `name`:

```rust
#[derive(Debug, Serialize, Deserialize, VersionChange)]
#[description = "Clients before v2.0.0 send `full_name` instead of split name fields"]
pub struct LegacyCreateUserRequestV1 {
    pub full_name: String,
}
```

For the latest model, the API expects a proper split and takes two fields: `first_name` and `last_name`.

```rust
// the latest model
#[derive(Debug, Serialize, Deserialize, VersionChange)]
#[description = "The latest request model expects first and last names separately"]
pub struct CreateUserRequest {
    pub first_name: String,
    pub last_name: String,
}
```

The request handler stays simple: it always expects the latest model.

```rust
#[post("/users")]
async fn create_user(
    user: VersionedJsonRequest<CreateUserRequest>,
) -> Result<VersionedJsonResponder<CreateUserResponse>> {
    let user = user.into_inner();
    Ok(VersionedJsonResponder(CreateUserResponse {
        first_name: user.first_name,
        last_name: user.last_name,
        status: "created".to_string(),
    }))
}
```

`VersionedJsonRequest` works similarly to `VersionedJsonResponder` from Part 1. While writing this, I realized both request and response wrappers could likely be unified into one type (similar to [web::Json](https://docs.rs/actix-web/latest/actix_web/web/struct.Json.html)). I plan to refactor that later, but the core concept stays the same.

`VersionedJsonRequest` extracts the provided version ID and upgrades the request body to the latest model before it reaches the handler.

One more thing: there is now a demo service that handles this flow end-to-end.

[Service source code](https://github.com/Tevinthuku/version-api/tree/main/version-actix-roundtrip-example)

[https://version-api-11g3.onrender.com](https://version-api-11g3.onrender.com)

To test this, keep in mind the service runs on a free plan and may need time to boot on the first request.

Version `0.5.0`: apart from the request-change, the response includes a `success` boolean field.

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  -H 'X-API-Version: 0.5.0' \
  -d '{"name":"Alice Doe"}' \
  https://version-api-11g3.onrender.com/users
{"name":"Alice Doe","success":true}%
```

Version `1.0.0`: apart from the request changes, the response now uses a `status` field instead of `success`.

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  -H 'X-API-Version: 1.0.0' \
  -d '{"full_name":"Alice Doe"}' \
  https://version-api-11g3.onrender.com/users
{"full_name":"Alice Doe","status":"created"}%
```

Version `2.0.0`

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  -H 'X-API-Version: 2.0.0' \
  -d '{"first_name":"Alice", "last_name": "Doe"}' \
  https://version-api-11g3.onrender.com/users
{"first_name":"Alice", "last_name": "Doe", "status":"created"}%
```

This service is a full roundtrip example where both request and response versions are respected, so existing clients do not break.

The next area I will work on is error handling. I currently have several `TODO` comments across the repository to improve this part. Specifically, I want to distinguish between internal errors (for example, transformer failures) and external errors (for example, a client sending an invalid DTO for an older version).

I will cover that in the next post.
