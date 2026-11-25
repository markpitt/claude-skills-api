<h1>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/assets/banner-dark.svg">
    <img src="docs/assets/banner-light.svg" alt="API skills" width="100%">
  </picture>
</h1>

Working with REST APIs: GitHub, Microsoft Graph, Home Assistant, FreeAgent, AdGuard Home and Dokploy.

Skills for driving an API from Claude. The first four (GitHub, Microsoft Graph, Home Assistant, FreeAgent) are orchestration hubs that route to focused reference files. `adguard-home-api` and `dokploy-api` are single-file operator guides for self-hosted services, so do not expect hub structure from every skill here.

This is one of seven skill groups. [claude-skills](https://github.com/markpitt/claude-skills) lists them all and
holds the shared tooling: validation, packaging, the evaluation harness and the TUI.

## Skills

<!-- BEGIN GENERATED SKILL INDEX -->
| Skill | Bundled files | What it does |
|---|---|---|
| [adguard-home-api](skills/adguard-home-api/SKILL.md) | none | Query and manage an AdGuard Home DNS server through its REST API. Covers stats, the query log, finding out… |
| [dokploy-api](skills/dokploy-api/SKILL.md) | none | Inspect and operate Docker Compose stacks managed by a Dokploy instance through its HTTP API. Covers listing… |
| [freeagent-api](skills/freeagent-api/SKILL.md) | 7 files | Interacts with the FreeAgent accounting API to manage invoices, contacts, projects, expenses, timeslips, and… |
| [github-api](skills/github-api/SKILL.md) | 7 files | Orchestrates GitHub API access across all services. |
| [home-assistant-api](skills/home-assistant-api/SKILL.md) | 7 files | Orchestrates access to the Home Assistant REST API for programmatic control of smart home devices. |
| [microsoft-graph-api](skills/microsoft-graph-api/SKILL.md) | 8 files | Orchestration hub for Microsoft Graph API across Microsoft 365 services. |

6 skills.
<!-- END GENERATED SKILL INDEX -->

## Install

Copy a skill directory into either skills location, then start a new session:

```bash
cp -r skills/<skill-name> ~/.claude/skills/    # personal, all projects
cp -r skills/<skill-name> .claude/skills/      # this project only
```

For Claude Desktop, zip the skill directory's contents and upload the zip under Settings,
Capabilities, Skills:

```bash
cd skills/<skill-name> && zip -r ../../<skill-name>.zip .
```

## Licence

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Each skill directory carries its
own copy of the licence, so a skill copied out on its own stays covered.
