---
name: freeagent-api
description: Calls the FreeAgent accounting API for invoices, estimates and credit notes, contacts, projects, timeslips, expenses, bank accounts and transactions, categories and company details, with OAuth 2.0 set-up, token refresh, paging, rate limits and a Python client template. Reads by default. Every create, update or delete is previewed and waits for explicit confirmation, and runs against the sandbox unless production is named. Use when asked to list or analyse FreeAgent data, create or update an invoice, log a timeslip or expense, reconcile bank transactions through the API, script against api.freeagent.com, or debug a FreeAgent 401, 422 or 429. Do NOT use for bookkeeping or tax advice with no API call, for other accounting systems such as Xero or QuickBooks, or for the FreeAgent web interface.
license: Apache-2.0
allowed-tools: Bash, Read, Write, Edit, WebFetch
metadata:
  version: "3.0.0"
---

# FreeAgent API

FreeAgent is online accounting for freelancers and small businesses. Its API is REST and JSON at
`https://api.freeagent.com/v2/`, with a separate sandbox at `https://api.sandbox.freeagent.com/v2/`
that has its own accounts and its own OAuth apps.

## Live money: the rules for changing anything

This skill touches real accounts, so a mistake is an invoice sent to a client or a bank record
that is wrong at year end. The rules follow from that.

- **Read unless asked to write.** Treat a request as analysis unless it plainly asks to create,
  update or delete. If it is unclear, ask. Never infer that someone wants a change.
- **Preview, then wait.** Before any POST, PUT or DELETE, show the method, the endpoint and the
  JSON body, and say in plain words what will change. Go ahead only on a clear yes. Anything
  ambiguous is a no.
- **Sandbox unless production is named.** Writes go to the sandbox URL unless the person says
  production in so many words.
- **Ask twice for the irreversible ones.** Deleting any financial record, creating or changing
  bank transactions or explanations, any bulk change, and changing an invoice's status (marking it
  sent, cancelling it, applying a credit note) get the preview, a yes, and then a second question.
  These are the changes an accountant would have to unpick.

Keep `FREEAGENT_ACCESS_TOKEN` and `FREEAGENT_REFRESH_TOKEN` in the environment. Never print them
or put them in code. On a `401`, say the token has expired and walk through re-authenticating,
rather than refreshing quietly in the background.

## Getting a token

1. Create an OAuth app at https://dev.freeagent.com/ (and a separate one for the sandbox).
2. Send the person to `https://api.freeagent.com/v2/approve_app?client_id=CLIENT_ID` to approve it
   and get an authorisation code.
3. Exchange the code at `https://api.freeagent.com/v2/token_endpoint`.
4. Store both tokens in environment variables, and send
   `Authorization: Bearer $FREEAGENT_ACCESS_TOKEN` on every request.

`resources/authentication-setup.md` has the full flow, refresh, and the usual failures.

## How the API behaves

- Responses wrap the resource in its name: `{"contact": {...}}` for one, `{"contacts": [...]}` for
  a list. Refer to other records by their full URL, such as
  `https://api.freeagent.com/v2/contacts/123`, not by a bare ID.
- Lists page with `page` and `per_page`: 25 by default, 100 at most. Follow the `Link` header's
  `next` rather than building page URLs, and read `X-Total-Count` for the total.
- Most lists filter with `view` (`?view=open_or_overdue` on invoices, `?view=active` on contacts)
  and `updated_since` (an ISO 8601 timestamp).
- Dates are `YYYY-MM-DD` and timestamps `YYYY-MM-DDTHH:MM:SSZ`.
- The limits are 120 requests a minute and 3,600 an hour per user, and 15 token refreshes a minute.
  On a `429`, back off (honouring `Retry-After` when it is sent) before trying again.
- A `422` names the fields that failed validation in its `errors` body. Read it rather than
  retrying with the same request.

| Task | Method and path |
|---|---|
| List active contacts | `GET /v2/contacts?view=active&per_page=100` |
| One invoice | `GET /v2/invoices/456` |
| Create a contact | `POST /v2/contacts` with a JSON body |
| Update an invoice | `PUT /v2/invoices/456` with the changed fields |
| Delete a timeslip | `DELETE /v2/timeslips/789` |

## Templates

- `templates/api-request-template.sh`, run as `bash templates/api-request-template.sh`: a curl
  wrapper for any method and endpoint.
- `templates/python-client.py`, run with `python3` or imported: a `FreeAgentClient` class with
  paging, rate-limit handling and helpers for the common resources.

## References

| File | Covers |
|---|---|
| `resources/authentication-setup.md` | OAuth set-up, refresh, token storage, 401 troubleshooting |
| `resources/contacts-organizations.md` | Contacts, the company record, users |
| `resources/accounting-objects.md` | Invoices, estimates, credit notes, projects, expenses, timeslips |
| `resources/banking-financial.md` | Bank accounts, transactions and explanations, categories |
| `resources/advanced-patterns.md` | Errors, rate limits, retries, caching, validation |
| `resources/examples.md` | Worked examples with the Python client, including bulk import and export |
| `resources/endpoints.md` | Every endpoint, for lookup |

External: the [FreeAgent API docs](https://dev.freeagent.com/docs).
