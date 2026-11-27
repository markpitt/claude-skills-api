---
name: home-assistant-api
description: Calls the Home Assistant REST and WebSocket APIs from code or scripts, for reading entity states and history, calling services to control lights, switches, climate, covers, locks, fans and media players, triggering automations and scripts, rendering templates on the server, firing events, and checking or reloading configuration. Covers long-lived access tokens, the Authorization header, entity IDs, service data, response handling and a small Python client. Use when writing code or scripts against a Home Assistant instance, asking which endpoint turns on a light or reads a sensor, counting or filtering entities, or debugging a 401 or 404 from the API. Do NOT use for installing, upgrading or troubleshooting a particular Home Assistant installation (containers, networking, tunnels), for writing automations in the YAML editor with no API involved, or for general REST API design.
license: Apache-2.0
metadata:
  version: "3.0.0"
---

# Home Assistant API

Home Assistant exposes a REST API under `http://HOST:8123/api/` and a WebSocket API at
`/api/websocket`. REST suits one-off reads and commands from a script. WebSocket suits anything
that should react to changes, because it pushes state changes instead of making you poll.

## Authentication

Create a long-lived access token in the user's profile (Security tab) and send it on every
request:

```bash
curl -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" \
  http://homeassistant.local:8123/api/
```

`{"message": "API running."}` means the URL and token are good. Keep the token in an environment
variable. It carries the full rights of the user who made it, so a leaked token can unlock doors,
not just read sensors. Make a dedicated user for an integration where that is practical.

## Reading state

- `GET /api/states/{entity_id}` for one entity, such as `sensor.kitchen_temperature`.
- `GET /api/states` for everything, then filter by the prefix of `entity_id` (`light.`,
  `sensor.`). On a large install that is a big response, so filter on the server instead when you
  can.
- `POST /api/template` with `{"template": "..."}` runs a Jinja template on the server and returns
  the result. It is the cheapest way to count or filter:
  `{{ states.light | selectattr('state', 'eq', 'on') | list | length }}`.
- `GET /api/history/period/{timestamp}?filter_entity_id=...` for history.

A state is a string, even for a number. `"unavailable"` and `"unknown"` are states too, so check
for them before converting.

## Controlling devices

Devices are controlled by calling a service, never by writing a state:

```bash
# One light, with brightness
curl -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" \
  -d '{"entity_id": "light.kitchen", "brightness": 200}' \
  http://homeassistant.local:8123/api/services/light/turn_on

# Several at once: a list, not several calls
-d '{"entity_id": ["light.kitchen", "light.hall"]}'
```

The path is always `/api/services/{domain}/{service}`: `climate/set_temperature`, `lock/unlock`,
`cover/set_cover_position`, `automation/trigger`, `script/turn_on`. `GET /api/services` lists every
service the instance has, with its fields, so use it rather than guessing a parameter name. For a
service that returns data, add `?return_response` to the URL.

`POST /api/states/{entity_id}` only changes Home Assistant's record of a state. It does not touch
the device, and the integration overwrites it at the next update. Use it for a virtual entity a
script owns, not to switch something on.

## Things that can hurt

Unlocking a lock, opening a garage door or cover, disarming an alarm, and restarting Home Assistant
(`homeassistant/restart`) change the physical world or take the house offline. Confirm each one
with the person before calling it, and never send them in a loop or a batch without saying so.

## Responses and errors

A service call returns the list of states it changed (possibly empty). `401` is a bad or missing
token. `404` is usually a misspelt `entity_id` or service; check `GET /api/states` or
`GET /api/services`. `400` is malformed JSON or service data. On a `500`, read `GET /api/error_log`.

## References

| File | Covers |
|---|---|
| `resources/core-concepts.md` | Tokens, HTTP basics, errors, timeouts, the WebSocket API |
| `resources/state-management.md` | Reading, filtering and watching states; history |
| `resources/service-reference.md` | Every common service with its fields, and returning service data |
| `resources/entity-types.md` | Attributes and services for each entity domain |
| `resources/templates.md` | Server-side template queries and aggregation |
| `resources/system-config.md` | Configuration, components, config checks, reloads, the WebSocket API |
| `resources/examples.md` | Clients in Python, Node.js, Bash and curl, retries and multi-entity patterns |

External: the [REST API docs](https://developers.home-assistant.io/docs/api/rest/) and the
[WebSocket API docs](https://developers.home-assistant.io/docs/api/websocket/).
