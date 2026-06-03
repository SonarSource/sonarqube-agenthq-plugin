# SonarQube Plugin for GitHub agent apps

Bring SonarQube Cloud code quality and security analysis into **Agent Apps for GitHub**: a custom agent (`sonarqube`) backed by the SonarQube MCP Server and a set of skills covering the most common quality, security, coverage, duplication, and SCA workflows.

This plugin is **GitHub agent apps-only**. There is no manual authentication to perform. Credentials are injected at runtime by GitHub agent app via OIDC.

## What you get

- **`sonarqube` agent** (`agents/main.agent.md`) — a specialized assistant that operates in the context of pull requests, branches, and files, routes user intent to the right skill, and calls the SonarQube MCP Server directly for tools that have no skill wrapper.
- **Skills** (`skills/`) — eight slash-invocable workflows:
  - `sonar-quality-gate` — quality gate pass/fail with per-condition detail
  - `sonar-list-issues` — search/filter bugs, vulnerabilities, and code smells
  - `sonar-fix-issue` — apply a fix for a specific rule violation
  - `sonar-analyze` — run server-side analysis on a single file
  - `sonar-coverage` — find files with low coverage and inspect uncovered lines
  - `sonar-duplication` — list duplicated files and inspect duplication blocks
  - `sonar-dependency-risks` — SCA (Advanced Security) dependency risks
  - `sonar-list-projects` — discover project keys accessible to the agent
- **SonarQube MCP Server** — wired in `agents/main.agent.md`, started in a Docker container with credentials supplied as environment variables.

## How authentication works

The agent definition (`agents/main.agent.md`) configures the SonarQube MCP Server with three environment variables, all sourced from GitHub agent app:

| Variable                | Source                                            | Purpose                                           |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------- |
| `SONARQUBE_TOKEN`       | `$GITHUB_COPILOT_OIDC_MCP_TOKEN` (OIDC exchange)  | Bearer token for SonarQube Cloud API calls       |
| `SONARQUBE_ORG`         | `${{ vars.COPILOT_MCP_SONARQUBE_ORG }}`           | SonarQube Cloud organization key                  |
| `SONARQUBE_PROJECT_KEY` | `${{ vars.COPILOT_MCP_SONARQUBE_PROJECT_KEY }}`   | Default project key for MCP tools                 |

The OIDC token is minted by GitHub on every session against the audience `https://sonarcloud.io` and exchanged with SonarQube Cloud — no static tokens, no user prompts, no system keychain.

## Repository setup

To use the plugin in your repository, follow the [guide in our docs](https://docs.sonarsource.com/agent-centric-development-cycle/developer-tools/agent-plugins/github-agent-apps).

## What the agent does beyond skills

When a user request doesn't map cleanly to one of the eight skills, the agent calls SonarQube MCP tools directly — for example:

- Explaining a rule (`show_rule`)
- Fetching arbitrary metric values (`get_component_measures`, `search_metrics`)
- Listing quality gates configured in the organization (`list_quality_gates`)
- Working with Security Hotspots (`search_security_hotspots`, `show_security_hotspot`, `change_security_hotspot_status`)
- Accepting / marking false-positive / reopening an issue (`change_sonar_issue_status`)

See `agents/main.agent.md` for the full routing rules and operating principles.

## License

Copyright (C) 2025-2026 SonarSource Sàrl. Licensed under [SSAL-1.0](LICENSE).

## Support

- Issues: https://community.sonarsource.com/
