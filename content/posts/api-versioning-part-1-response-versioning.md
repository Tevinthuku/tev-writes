+++
date = '2026-03-01T18:07:20+03:00'
draft = true
title = 'Api Versioning: Part 1 - Response versioning'
+++

## Introduction

> "APIs are Forever" — Werner Vogels

API consumers expect stability — once they've integrated your API, breaking changes are off the table. At the same time, your product naturally needs to evolve, and new features can require changes that would break existing consumers.

There are other ways to maintain backward compatibility — for example, versioning via the request path (`/v1/users` → `/v2/users`). However, maintaining multiple endpoints for each version can introduce significant maintenance overhead for your team.

I was inspired by [Stripe's approach to API versioning](https://stripe.com/blog/api-versioning), but their post didn't go into detail on how the versioning module actually worked under the hood.

That's what led me to build [version-api](https://github.com/Tevinthuku/version-api) (terrible name, I know) — an attempt to figure out how such a versioning module can actually work.

## How it works

Your endpoint always returns the **latest** response shape. When a consumer sends a request pinned to an older API version, the library automatically downgrades the response through a chain of transformers until it matches what that version expects.

Let's walk through a concrete example.

### 1. Define your API versions

First, declare the versions your API supports. The newest version comes first:

```rust
use version_actix::{BaseActixVersionIdExtractor, VersionedJsonResponder};
use version_core::{
    ApiVersionId, ChangeHistory, VersionChange, registry::ApiResponseResourceRegistry,
};

#[derive(ApiVersionId)]
enum ApiVersion {
    #[version("2.0.0")]
    V2_0_0,
    #[version("1.0.0")]
    V1_0_0,
}
```

### 2. Write your endpoint against the latest shape

Your handler always returns the current / latest DTO

```rust

#[derive(Serialize, Deserialize)]
struct CurrentUser {
    first_name: String,
    last_name: String,
}

#[get("/users/{id}")]
async fn get_user(id: web::Path<String>) -> Result<VersionedJsonResponder<CurrentUser>> {
    let user = CurrentUser {
        first_name: "Jane".into(),
        last_name: "Doe".into(),
    };
    Ok(VersionedJsonResponder(user))
}
```

### 3. Describe the change history

As shown above, In v2.0.0 we split the name field into first_name and last_name. Consumers on older versions still expect the single name field, so we describe that with a ChangeHistory:

```rust
#[derive(ChangeHistory)]
#[head(CurrentUser)]
#[changes(
    below(ApiVersion::V2_0_0) => UserWithSingleNameField,
)]
struct CurrentUserResponseHistory;
```

This reads as: "The head (latest) shape is CurrentUser. For any version below 2.0.0, downgrade it to UserWithSingleNameField."

### 4. Implement the downgrade

Define the older shape and a From conversion that maps the current structure back to it:

```rust
#[derive(VersionChange, Serialize, Deserialize)]
#[description = "Consumers below v2.0.0 expect a single name field"]
struct UserWithSingleNameField {
    name: String,
}

impl From<CurrentUser> for UserWithSingleNameField {
    fn from(user: CurrentUser) -> Self {
        Self {
            name: format!("{} {}", user.first_name, user.last_name),
        }
    }
}
```

### 5. Wire it up

Register the change history and tell the framework how to extract the version from incoming requests (here, from an X-API-Version header):

```rust
let mut registry = ApiResponseResourceRegistry::new();
CurrentUserResponseHistory::register(&mut registry).unwrap();

let version_extractor = BaseActixVersionIdExtractor::header_extractor(
    "X-API-Version".to_string(),
    ApiVersion::validator(),
);

let registry = web::Data::new(registry);
let version_id_extractor = web::Data::new(version_id_extractor);
HttpServer::new(move || {
        App::new()
            .service(get_user)
            .app_data(registry.clone())
            .app_data(version_id_extractor.clone())
    })
    .bind(("127.0.0.1", 8080))?
    .run()
.await
```

Now when a consumer sends X-API-Version: 1.0.0, they get { "name": "Jane Doe" }. A consumer on 2.0.0 (or with no header) gets { "first_name": "Jane", "last_name": "Doe" }. Your handler code stays the same either way.
