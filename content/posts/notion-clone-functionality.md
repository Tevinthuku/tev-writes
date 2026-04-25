+++
date = '2026-04-14T22:04:49+03:00'
draft = false
title = 'Coding challenges: Notion clone — Part 2: Notion Backend Functionality'
+++

### Tiny update from the first infra post.

1. I mentioned [here](./notion-clone-infra-experiement/index.md) that test runs were not being cached. I finally fixed this by using Garden’s [container test action instead of exec](https://docs.garden.io/reference/action-types/test/container). Because the image is cached, test runs are now almost instantaneous when backend code has not changed. I also fixed the test Dockerfile. I had been running `cargo-chef` with the `--release` flag, but tests do not run in release mode. That caused the workspace crates and dependencies to be recompiled every time. After removing `--release`, only my crates are recompiled, which significantly reduced test runtime.

```bash
✔ deploy.db                 → Done (took 28 sec)
ℹ deploy.db                 → Deploy type=helm name=db is ready.
ℹ test.workspace-tests      → Testing workspace-tests (type container) at version v-a27d8bf16e...
ℹ test.workspace-tests      → Starting Pod test-workspace-tests-3c7de6 with command 'cargo test'
ℹ test.workspace-tests      → [cargo]    Compiling db-migrations v0.1.0 (/usr/src/app/db-migrations)
ℹ test.workspace-tests      → [cargo]    Compiling db-schema v0.1.0 (/usr/src/app/db-schema)
ℹ test.workspace-tests      → [cargo]    Compiling http-server v0.1.0 (/usr/src/app/http-server)
ℹ test.workspace-tests      → [cargo]     Finished `test` profile [unoptimized + debuginfo] target(s) in 5.09s
ℹ test.workspace-tests      → [cargo]      Running unittests src/lib.rs (target/debug/deps/db_migrations-71fcf7c571791bee)

...


✔ test.workspace-tests      → Done (took 9.3 sec)
ℹ test.workspace-tests      → Test type=container name=workspace-tests is ready.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── Results after completion (took 39 seconds) ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  1 actions requested [test.workspace-tests]
  6 succeeded (thereof 5 dependencies) [test.workspace-tests, deploy.db, build.workspace-test-image, resolve-action.test.workspace-tests, ...]
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


Done! ✔️
```

-- 2 weeks later --

2. I simplified my database migration flow from a Kubernetes Job to a plain Garden `run` command.

Garden still spins up a pod from a container image and executes the command I provide; in this case, it just runs the `db-migration` binary.
A lot has changed since the “Tiny update from the first infra project” section above. I shut down my Azure Kubernetes cluster to reduce costs, as my bill was approaching $100/month (likely due to some misconfiguration). As a result, I dropped preview environment generation and stopped using a remote image registry.
Using a remote registry could still save significant CI time during image builds. Right now, `garden test --env=ci` on GitHub takes around 8 minutes because built images are not cached, so they are rebuilt on every run. I do not see this locally because images are cached on my machine.
For my next project, I plan to optimize both my Kubernetes setup and Rust build performance further. Despite these trade-offs, it has been great to see my Rust tests run in a Kubernetes environment.

I'll go over some of the things I think are note-worthy, regarding the backend implementation:

### Page sorting:

Since users may need pages to be sorted regardless of creation date, I used [lexorank](https://medium.com/whisperarts/lexorank-what-are-they-and-how-to-use-them-for-efficient-list-sorting-a48fc4e7849f) as an alternative to numeric sorting. It makes it easy to find a "rank" between 2 already existing pages. My actual implementation [can be found here](https://github.com/Tevinthuku/notion/blob/main/backend/http-server/src/pages/sort_order.rs)

### Page trees:

Pages are hierarchical: a page can have children, and those children can have their own children. So we need an efficient way to query both direct children and full descendant trees.
A simple approach is to store a `parent_id` on each row in `pages`:

```sql
CREATE TABLE pages (
  id uuid primary key default uuidv7(),
  ...
  parent_id uuid references pages(id) ON DELETE CASCADE
)
```

At scale, though, this becomes inefficient for some operations, especially moving a page and all of its descendants from one ancestor to another.
I used the [closure-table pattern](https://www.percona.com/blog/moving-subtrees-in-closure-table/) instead. With this approach, the full ancestor-descendant relationships are materialized in a separate closure table.

Pros:

- Read queries are simple and fast.
- No recursive CTEs are needed to fetch ancestry/descendants.
  Cons:

- It is more complex than maintaining only a parent_id.
- Changing parents requires rebuilding parts of the closure tree.
  That said, the complexity is manageable in practice. My implementation is [here](https://github.com/Tevinthuku/notion/blob/main/backend/http-server/src/pages/page_closures.rs), and the related tests are [here](https://github.com/Tevinthuku/notion/blob/main/backend/http-server/src/pages/mod.rs#L378).

The main idea is to delete the old ancestor references on the page and all its descendants and then to create a cartesian product of the new ancestors with all existing descendants. One can do it with a cross-join in sql, however, I opted to do it in the application code using itertool's cartesian_product [method](https://docs.rs/itertools/latest/itertools/trait.Itertools.html#method.cartesian_product).

### Modular errors:

One option is to maintain a single application-wide error enum with variants for every operation. In practice, that tightly couples the entire app to one error type. Instead, each service method and helper (for example, the function that builds the ancestor/descendant page tree) has its own local error type.
This modular approach makes refactoring much easier. Because operation-level errors are not global, they can evolve independently, and each error definition stays close to the code that produces it. Sabrina Jewson has a great write-up on this style of error design [here](https://sabrinajewson.org/blog/errors).
I did not implement observability/logging yet, but once that is in place, rich backtraces should make debugging straightforward: context from nested operations is propagated and attached to the final error.
Our `http-app-error` derive macro converts these operation-specific errors into `AppError`, which is based on small shared primitives rather than a single “god enum”.

```rs
#[derive(Debug)]
pub struct AppError {
    pub status_code: u16,
    pub message: String,
    pub cause: Box<dyn std::error::Error>,
}
```

Example: in the error enum below, the #[status(...)] attribute maps to AppError.status_code, and the #[error(...)] message becomes AppError.message via the error’s Display implementation.

```rs
#[derive(Debug, thiserror::Error, IntoAppError)]
pub(crate) enum BuildAncestorAndDescendantsTree {
    #[status(500)]
    #[error("internal error when building the ancestor-descendant-tree for page")]
    InternalError(diesel::result::Error),
    #[status(400)]
    #[error("descendant {parent_id} cannot be a parent of page {page_id}")]
    DescendantPickedAsParent { parent_id: Uuid, page_id: Uuid },
}
```

That’s mostly it for the backend; I’ll tackle the frontend in a later update.
