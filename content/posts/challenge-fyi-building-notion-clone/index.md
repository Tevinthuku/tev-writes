+++
date = '2026-04-12T07:55:33+03:00'
draft = false
title = 'Coding challenges: Notion clone — Part 1: Infrastructure experiment'
+++

I picked up [this Notion clone challenge](https://codingchallenges.fyi/challenges/challenge-notion) not because building a Notion-like product is hard for me - but because I wanted a real excuse to set up and orchestrate a Kubernetes cluster from scratch.

I've touched Kubernetes at work, but always at arm's length. This felt like the right project to go deeper: deploy a frontend, a backend, and a Postgres database, wired together properly, running on a real remote cluster.

---

#### The Kubernetes objects I've pieced together

- **[Ingress](https://github.com/Tevinthuku/notion/blob/main/ingress/ingress.yaml)** — One ingress per namespace. All external traffic into the environment routes through a single place, fanning out to the right services.
- **[HTTP server service](https://github.com/Tevinthuku/notion/blob/main/backend/http-server/manifests/service.yaml)** — Exposes the backend HTTP server pods to the rest of the cluster.
- **[DB migration job](https://github.com/Tevinthuku/notion/blob/main/backend/db-migrations/manifests/job.yaml)** — A one-off Job that runs database migrations before the HTTP server deploys. If migrations fail, the new backend doesn't roll out.
- **[Web service](https://github.com/Tevinthuku/notion/blob/main/frontend/manifests/service.yaml)** — Exposes the frontend.

#### Tools I'm using

- **[Garden](https://docs.garden.io)** — Spins up production-like environments for local dev, testing, and CI. The main reason I can have a live preview environment on every PR.
- **[Azure AKS](https://azure.microsoft.com/en-us/products/kubernetes-service)** — Managed Kubernetes on Azure. Handles the control plane so I'm not babysitting cluster nodes.

#### What I wanted before writing a single line of product code

My goal from day one was a proper preview environment on every PR — a live, running version of the stack where I can verify changes before merging, with tests running in a dedicated CI namespace. No cutting corners and testing locally while telling myself "it'll work in prod."

I wanted the infrastructure to feel like a real team setup, just scaled down for cost reasons.

#### Challenges so far

**1. Rust builds in CI are painfully slow.**

Garden recommends `cluster-buildkit` for in-cluster builds, but I hit a [BuildKit issue](https://github.com/moby/buildkit/issues/6505) that seemed specific to how BuildKit interacts with the AKS nodes. I didn't dig deep into it — switched to `kaniko` instead, but that was even slower, almost certainly because my cluster is deliberately underpowered (cost). I've fallen back to `local-docker` for now, but its still slow.

**2. No dependency caching between test runs.**

Right now the test job recompiles everything from scratch on each push. I'd expect these dependencies to be cached somewhere, but clearly something in my setup isn't wiring that up correctly. I haven't had time to investigate properly — it's on the list.

Garden's [Remote Container Builder](https://docs.garden.io/features/remote-container-builder) might address both of these, but I'd rather find a cheaper solution that doesn't add another paid service to a hobby project.

---

#### Next steps

Now that the infrastructure skeleton is in place, I'll actually start building the Notion spec: backend API, frontend, database schema. The interesting part will be watching how each push interacts with the live cluster — that feedback loop is exactly what I wanted to get comfortable with.

Once the project wraps up, I'll tear down the AKS cluster so I'm not paying for idle nodes.
