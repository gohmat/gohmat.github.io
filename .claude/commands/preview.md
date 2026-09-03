---
description: Build and start the Jekyll site locally in Docker for live preview
---

Start a local live preview of this Jekyll site using Docker, so changes can be checked at http://localhost:8080 without waiting for the GitHub Actions deploy.

Steps:

1. Confirm Docker is running: `docker info`. If it's not, tell the user to open Docker Desktop and stop — don't try to start it yourself.
2. Check whether the `jekyll` service image already exists and is up to date with `Dockerfile`/`docker-compose.yml`/`Gemfile` (e.g. via `docker compose images` and file mtimes, or just ask "has Gemfile changed since last build?" if unsure). Only run a full `docker compose build` when needed — otherwise skip straight to step 3, since rebuilding is slow and usually unnecessary.
3. Run `docker compose up -d`.
4. Wait a few seconds, then verify it's actually serving: `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/` should return 200. If not, check `docker compose logs --tail=50` and diagnose.
5. Confirm with `git status --porcelain` that nothing in the repo (especially `Gemfile.lock`) got modified by the container — the compose setup should not touch it, but verify.
6. Tell the user the site is live at http://localhost:8080, that editing any file auto-rebuilds with LiveReload (no manual restart needed), and that `docker compose down` stops it.

Do not modify `Dockerfile`, `docker-compose.yml`, or `bin/entry_point.sh` as part of running this command — they're already configured correctly (Ruby pinned to 3.2.2 to match the GitHub Actions deploy workflow, and the compose `command` overridden to avoid the image's default entrypoint deleting the bind-mounted `Gemfile.lock`). If something is broken, report it rather than silently changing the setup.
