---
name: microsoft-graph-api
description: Calls Microsoft Graph, the REST API for Microsoft 365 and Microsoft Entra ID (formerly Azure AD), for users, groups and directory objects, mail, calendars and meetings, OneDrive and SharePoint files, Teams chats and channels, Planner, To Do and OneNote, and security and device reports. Covers app registration, delegated versus application permissions, the client credentials flow, MSAL and the Graph SDKs, OData query parameters, paging, batching, delta queries and throttling. Use when writing code or scripts that call graph.microsoft.com, registering an app to read Microsoft 365 data, working out which Graph permission an operation needs, or debugging a 401, 403 or 429 from Graph. Do NOT use for Azure resource management through ARM or the Azure CLI, for Azure DevOps (use azure-devops), for Exchange or SharePoint PowerShell modules, or for graph data structures and charts that have nothing to do with Microsoft.
license: Apache-2.0
metadata:
  version: "3.0.0"
---

# Microsoft Graph API

Graph is one REST endpoint, `https://graph.microsoft.com/{version}/{resource}`, in front of
Microsoft 365 and Microsoft Entra ID. Every call needs an OAuth 2.0 access token from the Microsoft
identity platform, and most problems are permissions problems rather than API problems.

## v1.0 and beta

Build on `v1.0`. `beta` holds previews: Microsoft can change or remove them without notice, and
they are not supported in production. Reach for beta only when the feature does not exist in v1.0
yet, keep those calls in one place, and check whether they have moved to v1.0 when you next touch
the code. "Try it in beta first" is not a testing strategy, because beta can behave differently
from the v1.0 you will ship on.

## Permissions decide everything

There are two kinds, and choosing the wrong one is the most common mistake:

- **Delegated** permissions act as the signed-in user, and can reach only what that user can. Use
  them for an app a person signs in to. `/me` works.
- **Application** permissions act as the app itself, across the whole tenant, with no user. Use
  them for a daemon or background job, through the client credentials flow. `/me` does not work;
  address users as `/users/{id}`. They need an administrator's consent, and they are broad:
  `Mail.Read` as an application permission reads every mailbox in the tenant.

Ask for the least privilege the operation needs, and check the Permissions section on the
operation's own reference page rather than guessing. Where Microsoft offers a narrower option, such
as `Sites.Selected` or an Exchange application access policy, use it.

Register the app in the Microsoft Entra admin centre (entra.microsoft.com) under App registrations.
Prefer a certificate or a managed identity to a client secret, and never put a secret in code.

## Calling it

Query parameters:

```text
$select=displayName,mail        only the properties you need
$filter=startsWith(displayName,'A')
$orderby=displayName
$top=50                         page size
$expand=members                 include related objects
$count=true                     total count (with ConsistencyLevel: eventual on directory objects)
$search="keyword"
```

- **Paging.** Follow `@odata.nextLink` until it is absent. Code that reads one page is the usual
  cause of "some users are missing".
- **Batching.** `POST /$batch` takes up to 20 requests in one call.
- **Change tracking.** Delta queries (`/users/delta`) return what changed since the last
  `@odata.deltaLink`, which is far cheaper than re-reading everything.
- **Throttling.** A `429` comes with `Retry-After`. Wait that long; retrying straight away
  lengthens the throttle.

Errors come back as `{"error": {"code": "...", "message": "..."}}`. `401` is a missing, expired or
wrong-audience token. `403` is a permission the token does not carry, or one granted without admin
consent: decode the token and read its `scp` (delegated) or `roles` (application) claim. `404` is
often a wrong ID, or a delegated call for something the user cannot see.

## SDKs

- .NET: `Microsoft.Graph` on NuGet, with `Azure.Identity` for credentials.
- Python: `msgraph-sdk` on PyPI (async by default, with `azure-identity`). The beta API has its own
  package, `msgraph-beta-sdk`.
- JavaScript and TypeScript: `@microsoft/microsoft-graph-client` on npm.

Graph Explorer (developer.microsoft.com/graph/graph-explorer) is the quickest way to try a call and
see which permissions it asks for.

To check a registration works, run `python3 scripts/test-connection.py` from this skill's folder.
It needs `requests` and `msal`, and reads `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and
`AZURE_CLIENT_SECRET` from the environment.

## References

| File | Covers |
|---|---|
| `resources/authentication-apps.md` | App registrations, service principals, permissions, auth flows, credentials |
| `resources/identity-access.md` | Users, groups, the organisation and directory objects |
| `resources/mail-operations.md` | Mail, folders, attachments, rules |
| `resources/calendar-scheduling.md` | Events, recurrence, meetings, free and busy |
| `resources/files-storage.md` | OneDrive, SharePoint, uploads, sharing |
| `resources/teams-communications.md` | Teams, channels, chats, presence, online meetings |
| `resources/planning-notes.md` | Planner, To Do, OneNote |
| `resources/security-governance.md` | Security alerts, compliance, devices, reports |
| `templates/auth-setup.md` | Step-by-step app registration and token acquisition |

External: the [API reference](https://learn.microsoft.com/graph/api/overview), the
[permissions reference](https://learn.microsoft.com/graph/permissions-reference), the
[versioning and support policy](https://learn.microsoft.com/graph/versioning-and-support) and the
[changelog](https://learn.microsoft.com/graph/changelog).
