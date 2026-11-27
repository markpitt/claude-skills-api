---
name: adguard-home-api
description: Queries and manages an AdGuard Home DNS server through its REST API. Covers stats, the query log, finding out why a domain is blocked, allowlisting, persistent clients and tags, blocklists, rate limiting, DHCP leases and the access list. Use when a site or app breaks or shows CAPTCHAs behind AdGuard Home, for DNS audits, or when asked to change AdGuard settings. Do NOT use for Pi-hole or other DNS filters.
license: Apache-2.0
metadata:
  version: "1.0.1"
---

# AdGuard Home API

- Base: `http://<host>:<web-port>/control/` (the web UI port, not 53). HTTP basic auth: `ADGUARD_HOME_USER` and `ADGUARD_HOME_PASSWORD` from the environment. Never print the password.
- Read with `GET`; write with `POST` and a JSON body.

```bash
ag()      { curl -sf -u "$ADGUARD_HOME_USER:$ADGUARD_HOME_PASSWORD" "$ADGUARD_URL/control/$1"; }
ag_post() { curl -s  -u "$ADGUARD_HOME_USER:$ADGUARD_HOME_PASSWORD" -H 'Content-Type: application/json' -X POST "$ADGUARD_URL/control/$1" -d "$2" -w '\nHTTP %{http_code}\n'; }
```

## Read calls

| Need | Call |
|---|---|
| Version, running, protection | `ag status` |
| Upstreams, rate limit, DNSSEC, blocking mode | `ag dns_info` |
| Blocklists and user rules | `ag filtering/status` |
| Totals, top clients/blocked/queried | `ag stats` |
| Why is X blocked? | `ag "filtering/check_host?name=X"` (`reason`, matching `rules`; filter id 0 = a user rule) |
| What did a client just get blocked? | `ag "querylog?search=<ip-or-domain>&response_status=blocked&limit=300"` |
| Persistent clients | `ag clients` |
| DHCP state, leases, static leases | `ag dhcp/status` |
| Access list (disallowed clients) | `ag access/list` |

Also available: `querylog/config`, `stats/config`, `tls/status`.

## Write calls

- **`filtering/set_rules {"rules":[...]}` replaces all user rules.** Read `filtering/status`, change `user_rules`, send the whole list back.
- Allow a domain: add `@@||example.com^$important` to the user rules. Remove the same line to undo.
- Add or remove a blocklist: `filtering/add_url {"name","url","whitelist":false}`, `filtering/remove_url {"url","whitelist":false}`.
- DNS settings: `dns_config` with only the fields to change, for example `{"ratelimit":0}`.
- Clients: `clients/add`, `clients/update {"name","data":{full client object}}`, `clients/delete {"name"}`. Update needs the full object, so read the client first and change one field.
- Block one client's DNS entirely: add its IP to `disallowed_clients` through `access/list` then `access/set` (read-modify-write).
- DHCP: `dhcp/set_config`, `dhcp/add_static_lease {mac,ip,hostname}`, `dhcp/remove_static_lease`.

## Gotchas

- After `set_rules`, `check_host` can report the old result for a second or two while filters reload. Re-check before concluding.
- Client ids hold IPs and MACs. MAC ids only match while AdGuard is the DHCP server; otherwise the IP id does the work. In rules, target `$client='<client name>'` instead of an IP, and tags with `$ctag=<tag>`.
- **An invalid `dns_config` combination (for example an empty `local_ptr_upstreams` with `use_private_ptr_resolvers: true`) returns HTTP 500 and can leave DNS stopped for the whole network.** After every `dns_config` write, check `ag status` shows `running: true` and that a real lookup (`dig @<host> example.com`) answers.
- Never enable AdGuard's DHCP server while another DHCP server (usually the router) is active.
- `dhcp/find_active_dhcp` is unreliable for detecting other servers.

## Triage: a site breaks or shows CAPTCHAs

1. `check_host` on the domain, and on the anti-bot and fraud-scoring domains the page loads.
2. Look at the client's query log for `FilteredBlackList` entries around the failure.
3. Allow the specific domain with an `$important` rule; do not remove a whole list for one site.
4. Re-test, and note the change and how to undo it.

Over-aggressive lists show as a block rate above roughly 30%; 10 to 20% is typical for a household.
