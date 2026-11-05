# Changelog

## v1.0.0 (unreleased)

- Reusable workflows: `node-ci`, `backend-ci`, `docker-publish`, `deploy`.
- `deploy` pins the SSH host key, serializes deploys per service, waits for health and rolls back automatically.
- Server config: Docker Compose with Caddy, env examples and a setup guide.
- Consumer examples for Frontend, Landing Page, Backend and Mailserver.
- `test-templates` workflow (actionlint, sample project runs, compose validation).
