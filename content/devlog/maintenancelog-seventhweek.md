---
title: "Maintenance Log - Seventh Week: Deployment, CI/CD & Running It For Real"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Backend", "Deployment", "DevOps", "Docker", "CI/CD", "GitHubActions"]
series: ["Maintenance Log"]
date: 2026-03-29
draft: false
---

# Devlog Week 7: Deployment, CI/CD & Running It For Real

This week was all about getting the project out of my IDE and onto an actual server.
The goal wasn’t to add new domain features — it was to make the API build, ship, and update automatically.

**Deployed API:** https://maintenancelog.heltsort.dk/

Right now the root endpoint just returns a small JSON welcome message:
`{"message":"Welcome to the Maintenance Log!"}`

If you hit `/routes` you can see an overview of the available endpoints and what roles are required.

## What Changed This Week

- **CI pipeline**: GitHub Actions builds the project with Maven and only continues if the tests pass
- **Docker image**: the build produces a Docker image and pushes it to Docker Hub
- **Server setup**: DigitalOcean droplet runs the API via `docker-compose`
- **Auto-updates**: Watchtower pulls new images and restarts the container automatically
- **HTTPS + domain**: Caddy reverse proxies the API behind a real domain + TLS

---

## The Big Picture: From Push → Running Server

My deployment pipeline ended up being roughly:

1. I push to `main`
2. GitHub Actions runs `mvn package` (tests included)
3. If that succeeds, it builds a Docker image from my `Dockerfile`
4. The image is pushed to Docker Hub (`jespertaxicon/maintenancelog:latest`)
5. My droplet pulls/runs the image using `docker-compose`
6. Watchtower monitors the image and updates the running container when `:latest` changes
7. Caddy sits in front and serves the API over HTTPS on my domain

This matches the “full pipeline” setup we used in the Deployment & DevOps week (Docker + GitHub Actions + Docker Hub + DigitalOcean + Watchtower + Caddy).

---

## GitHub Actions: Build, Test, Then Push

I used a workflow based on the course setup. The important part (for me) was that **deployment is gated by tests** — if my tests fail, nothing gets pushed.

My workflow does:
- checkout
- setup JDK 17
- `mvn --batch-mode --update-snapshots package`
- login to Docker Hub
- build + push the Docker image

I also had to provide environment variables at build time. The fix was adding the JWT-related values as **GitHub Secrets** and injecting them into the Maven build step (so the build/test phase in CI has the same inputs as my local setup):

- `ISSUER`
- `SECRET_KEY`
- `TOKEN_EXPIRE_TIME`

---

## Docker Compose: API + Caddy + Watchtower

On the droplet I used a `docker-compose.yml` similar to the course example, but with my own image.

I’m also running Postgres on the droplet, but it’s firewall-blocked so it’s not exposed publicly — it’s only meant to be reachable from the server/internal network.

My API container ended up roughly like this (redacted):

```yml
maintenancelog:
  image: "myuser"/maintenancelog:latest
  container_name: maintenancelog
  environment:
    - DEPLOYED=true
    - DB_NAME=...
    - DB_USERNAME=...
    - DB_PASSWORD=...
    - CONNECTION_STR=...
    - SECRET_KEY=...
    - ISSUER=...
    - TOKEN_EXPIRE_TIME=...
```

I’m intentionally not listing port mappings here — the important part is that the API runs inside the Docker network and is reached through a reverse proxy.

My `Caddyfile` is basically:

```caddyfile
maintenancelog.heltsort.dk {
  reverse_proxy maintenancelog:7070
}
```

So **Caddy** acts as the front door (reverse proxy + TLS), and **Watchtower** handles automatic redeploys when the Docker Hub image updates.

---

## The Annoying Part: Tests Blocking Deployment

The biggest pain point this week was not Docker — it was the fact that the pipeline is set up correctly: **tests must pass before deployment happens**.

I hit two issues:

1. **Tests timing out**
   After adding JWT auth/authorization, my integration tests got heavier. Some runs started timing out because the test setup now includes token creation and authenticated requests.

2. **GitHub Actions env vars**
   Locally everything worked, but in GitHub Actions the build/test step didn’t have the environment variables it needed. That caused tests to fail *only* in CI, and I couldn’t reproduce it locally at first.

The takeaway for me: when your build depends on env vars, you need to treat CI like a separate environment and be explicit about what it gets (secrets/env vars), otherwise you end up chasing “works on my machine” failures.

---

## What I Learned

- A “real” deployment isn’t just shipping code — it’s making the system **repeatable**: build, test, package, run.
- Gating Docker pushes on tests is painful when tests are flaky… but it’s the right kind of painful.
- Caddy + Watchtower + Docker Compose is a nice combo for a small API: simple mental model, and updates are basically automatic once it’s set up.
