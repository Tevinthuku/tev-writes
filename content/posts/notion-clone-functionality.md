+++
date = '2026-04-14T22:04:49+03:00'
draft = true
title = 'Coding challenges: Notion clone — Part 2: Notion Functionality'
+++

### Tiny update from the first infra post.

I had mentioned [here](./notion-clone-infra-experiement/index.md) that test runs were not cached. I've finally solved the issue by relying on garden's [container test action instead of exec](https://docs.garden.io/reference/action-types/test/container)
Since the image gets cached, when nothing changes, the test action runs almost instantaneously.

```bash
ℹ test.workspace-tests      → Getting status for Test workspace-tests (type container) at version v-8f31e82bdd...
ℹ test.workspace-tests      → Cached result found in Local Cache (saved  2 minutes 58 seconds)
✔ test.workspace-tests      → Already run
ℹ test.workspace-tests      → Test type=container name=workspace-tests status is ready.

──────────────────────────────────────────────────────────────────────────────────────────────────── Results after completion (took 1 second) ────────────────────────────────────────────────────────────────────────────────────────────────────
  1 actions requested [test.workspace-tests]
  3 succeeded (thereof 2 dependencies) [test.workspace-tests, resolve-action.test.workspace-tests, resolve-action.build.workspace-test-image]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


Done! ✔️
```
