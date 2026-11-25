---
name: dokploy-api
description: Inspect and operate Docker Compose stacks managed by a Dokploy instance through its HTTP API. Covers listing projects and composes, reading a compose with secrets masked, restart, deploy, pinning images, editing a compose with yq, and backup and restore. Use when asked to restart, redeploy, upgrade, pin or inspect a self-hosted service that Dokploy manages. Do NOT use for plain `docker compose` projects that Dokploy does not manage.
license: Apache-2.0
metadata:
  version: "1.0.0"
---

# Dokploy API

Dokploy exposes tRPC-style endpoints under `<base>/api/<procedure>`.

- Base URL: Dokploy's UI/API port (`http://127.0.0.1:3000` on the host by default). Set `DOKPLOY_URL` and `DOKPLOY_API_KEY` in the environment; send the key as `x-api-key`.
- Queries are `GET` with a query string (`compose.one?composeId=ID`). Mutations are `POST` with a JSON body.
- Never print `DOKPLOY_API_KEY`, and never dump a raw `composeFile` or `env`: both can hold secrets.

```bash
dp()      { curl -sf -H "x-api-key: $DOKPLOY_API_KEY" "$DOKPLOY_URL/api/$1" "${@:2}"; }
dp_post() { curl -sf -H "x-api-key: $DOKPLOY_API_KEY" -H 'Content-Type: application/json' -X POST "$DOKPLOY_URL/api/$1" -d "$2"; }
```

## Procedures

| Need | Call |
|---|---|
| Inventory (projects, composes, apps, status) | `dp project.all` (compose entries carry `composeId`, `name`, `appName`, `composeStatus`) |
| One compose (file, env, source type) | `dp "compose.one?composeId=$ID"` |
| Save a new compose file (not applied yet) | `dp_post compose.update '{"composeId":"ID","composeFile":"..."}'` |
| Apply the saved compose (recreates containers) | `dp_post compose.deploy '{"composeId":"ID"}'` |
| Restart | `dp_post compose.stop '{"composeId":"ID"}'` then `dp_post compose.start '{"composeId":"ID"}'` |
| Move between projects | `dp_post compose.move '{"composeId":"ID","targetEnvironmentId":"ENV"}'` (containers untouched) |

Containers of a stack carry the label `com.docker.compose.project=<appName>`; on disk the compose lives at `/etc/dokploy/compose/<appName>/code/docker-compose.yml`.

## Gotchas

- **`stop` + `start` uses the on-disk compose and honours `pull_policy`.** With `pull_policy: always` and a `:latest` tag, a "restart" silently upgrades the image. Pin images (a digest is best) and tell the user before any deploy of a `latest` stack.
- `compose.update` only writes Dokploy's database. Nothing changes until `compose.deploy`.
- `sourceType: raw` composes are edited through the API. `github` composes come from the repository (often with `autoDeploy`): change them in git, not with `compose.update`.
- `composeStatus: error` means the last deployment failed; read the deployment log in the Dokploy UI.
- A redeploy reuses anonymous volumes. Images that write config into one at startup keep stale entries; remove the container and its volumes (`docker rm -f -v <container>`) and deploy again.
- Do not trust a masking regex to catch every secret format. Check any new format by eye, and prefer moving secrets into the compose `env` and referencing `${VAR}`.

## Changing an image

1. Read the compose, replace the `image:` line, show the user a diff.
2. Take a backup of the current compose file (below).
3. `compose.update` with the new text, then `compose.deploy`.
4. Poll `compose.one` until `composeStatus` leaves `running`, then confirm the containers are healthy and the service answers.

To pin by digest: `docker pull <image>`, then `docker image inspect <image> --format '{{index .RepoDigests 0}}'`, and use `image@sha256:...`.

## Editing a compose with yq

Use mikefarah yq v4 (check `yq --version`; a different `yq` of the same name is common). Always dry-run first.

```bash
f=$(dp "compose.one?composeId=$ID" | jq -r .composeFile)
nf=$(printf '%s\n' "$f" | yq '.services.web.healthcheck.interval = "60s"')
printf '%s\n' "$nf" | yq -e '.services' >/dev/null    # refuse a result with no services
diff <(printf '%s\n' "$f") <(printf '%s\n' "$nf")     # show masked in practice; may contain secrets
```

Then back up, `compose.update` with `jq -n --arg id "$ID" --arg f "$nf" '{composeId:$id,composeFile:$f}'`, and `compose.deploy` once the user agrees.

Do not put secrets in a yq expression: it lands in shell history and transcripts.

## Backup and restore

- Backup: write `compose.one`'s `composeFile` to a file created with `umask 077`, outside any git repository (it holds secrets unmasked).
- Restore: `compose.update` with the backup's contents, then `compose.deploy`. Back up the current version before restoring.
- Log every change with the compose id, the backup path and how to revert.
