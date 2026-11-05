# ci-blueprints

Reusable GitHub Actions workflows and server config for our four repos: Backend, Frontend, Landing Page and Mailserver.
Each repo keeps a short workflow that calls these blueprints, so a new service never needs its own pipeline.

```
.github/workflows/
  node-ci.yml          lint, test, build for Node projects
  backend-ci.yml       toolchain setup + your commands (node, python, go, java)
  docker-publish.yml   build and push an image to GHCR
  deploy.yml           deploy one service to the server over SSH, with rollback
  test-templates.yml   CI for this repo (actionlint + sample projects)
servers/               compose file, Caddyfile, env examples, SETUP.md
examples/consumers/    ready-to-copy workflows for each of the four repos
examples/node-app, examples/docker-app    tiny projects used by test-templates.yml
```

## How it fits together

```
push to main in a project repo
  -> node-ci / backend-ci        (lint, test, build)
  -> docker-publish              (image to ghcr.io, tagged with the short commit SHA)
  -> deploy                      (SSH to the server, pull and restart only that service)
```

Pull requests run CI and a build-only image check (`push: false`), and never deploy.

## Add a new service in 3 steps

1. **Project repo:** copy the closest file from `examples/consumers/` to `.github/workflows/ci.yml` and change `image-name` and `service`.
2. **Server:** add the service to `/opt/app/docker-compose.yml` using `image: ghcr.io/your-org/<name>:${<NAME>_TAG:-latest}` plus a healthcheck, and add a block to the `Caddyfile`.
3. **GitHub:** make sure the repo can see the `production` environment secrets (see below), then push.

The tag variable name is the service name uppercased with `-` turned into `_`, plus `_TAG` (service `my-api` uses `MY_API_TAG`).

## Required GitHub settings

- **Let other repos use these workflows:** in this repo go to Settings, Actions, General, Access, and allow repositories in the organization.
- **Environments:** create `staging` / `production` in each project repo (or use organization-level secrets). Add required reviewers to `production`.
- **Environment secrets** used by `deploy.yml`:

  | Secret | Meaning |
  | --- | --- |
  | `DEPLOY_SSH_KEY` | private key of the server's `deploy` user |
  | `DEPLOY_HOST` | server hostname or IP |
  | `DEPLOY_USER` | SSH user, normally `deploy` |
  | `DEPLOY_KNOWN_HOSTS` | pinned host key line from `ssh-keyscan` |

  The deploy call uses `secrets: inherit` because environment-level secrets are only visible to the job that declares the environment. The workflow fails early with a clear message if one is missing.
- **Image access:** `docker-publish.yml` pushes with `GITHUB_TOKEN`. The server pulls with a read-only token (see `servers/SETUP.md`).
- The calling job of `docker-publish.yml` must grant `packages: write` (the consumer examples already do).

## Versioning

- Releases are semver tags (`v1.0.0`, `v1.1.0`, ...). Consumers call `@v1`.
- Move the `v1` tag forward only for non-breaking changes. Breaking changes get `v2`, and repos migrate one at a time.
- Never point consumers at `@main`: one bad commit would break every pipeline.
- For stricter supply-chain safety, pin to a commit SHA and let Dependabot or Renovate open update PRs.
- Record every change in `CHANGELOG.md`.

Releasing:

```bash
git tag v1.0.0 && git push origin v1.0.0
git tag -f v1 && git push -f origin v1     # moves the major tag
```

## Rollback

Redeploy an older image tag by re-running the deploy workflow for that commit (Actions, "Re-run jobs" on an older run), or on the server:

```bash
cd /opt/app
sed -i 's|^FRONTEND_TAG=.*|FRONTEND_TAG=<old-tag>|' .env
docker compose up -d --no-deps frontend
```

`deploy.yml` also rolls back automatically to the previous tag if the new container does not become healthy within `health-timeout` seconds.

## Notes

- `deploy.yml` serializes deploys per service and environment with `concurrency`, so two runs never overlap.
- Third-party actions are pinned to major versions. Dependabot keeps them current.
- Compose has no zero-downtime rollout: the service restarts briefly on each deploy. If that becomes a problem, move to k3s later. Only `deploy.yml` would change, not the consumer repos.
