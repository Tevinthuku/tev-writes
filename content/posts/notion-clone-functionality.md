+++
date = '2026-04-14T22:04:49+03:00'
draft = true
title = 'Coding challenges: Notion clone — Part 2: Notion Functionality'
+++

### Tiny update from the first infra post.

1. I had mentioned [here](./notion-clone-infra-experiement/index.md) that test runs were not cached. I've finally solved the issue by relying on garden's [container test action instead of exec](https://docs.garden.io/reference/action-types/test/container)
   Since the image gets cached, when the backend code doesn't change, the test action runs almost instantaneously.
   One other thing I did was I fixed the test Dockerfile, I was running cargo-chef with the release flag, since tests don't run on release mode, it would always re-compile the workspace-crates and the dependencies, after removing this flag, only my crates recompile, reducing the time it takes to run tests drastically.

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

2. I've simplified my db migration from a Job to a simple Garden run command, which internally spins up a pod based off a container image and executes the command I specify. In my case, its simply running the db-migration binary.
