# SonarQube Plugin for GitHub AgentHQ

Bring SonarQube Cloud code quality and security analysis into **GitHub AgentHQ**: a custom agent (`sonarqube`) backed by the SonarQube MCP Server and a set of skills covering the most common quality, security, coverage, duplication, and SCA workflows.

This plugin is **AgentHQ-only**. There is no CLI to install, no manual authentication to perform, and no IDE-side configuration. Credentials are injected at runtime by AgentHQ via OIDC.

## What you get

- **`sonarqube` agent** (`agents/main.agent.md`) — a specialized assistant that operates in the context of pull requests, branches, and files, routes user intent to the right skill, and calls the SonarQube MCP Server directly for tools that have no skill wrapper.
- **Skills** (`skills/`) — eight slash-invocable workflows:
  - `sonar-quality-gate` — quality gate pass/fail with per-condition detail
  - `sonar-list-issues` — search/filter bugs, vulnerabilities, and code smells
  - `sonar-fix-issue` — apply a fix for a specific rule violation
  - `sonar-analyze` — run server-side analysis on a single file with project context
  - `sonar-coverage` — find files with low coverage and inspect uncovered lines
  - `sonar-duplication` — list duplicated files and inspect duplication blocks
  - `sonar-dependency-risks` — SCA (Advanced Security) dependency risks
  - `sonar-list-projects` — discover project keys accessible to the agent
- **SonarQube MCP Server** — wired in `agents/main.agent.md`, started in a Docker container with credentials supplied as environment variables.

## How authentication works

The agent definition (`agents/main.agent.md`) configures the SonarQube MCP Server with three environment variables, all sourced from AgentHQ:

| Variable                | Source                                            | Purpose                                           |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------- |
| `SONARQUBE_TOKEN`       | `$GITHUB_COPILOT_OIDC_MCP_TOKEN` (OIDC exchange)  | Bearer token for SonarQube Cloud API calls       |
| `SONARQUBE_ORG`         | `${{ vars.COPILOT_MCP_SONARQUBE_ORG }}`           | SonarQube Cloud organization key                  |
| `SONARQUBE_PROJECT_KEY` | `${{ vars.COPILOT_MCP_SONARQUBE_PROJECT_KEY }}`   | Default project key for MCP tools                 |

The OIDC token is minted by AgentHQ on every session against the audience `https://sonarcloud.io` and exchanged with SonarQube Cloud — no static tokens, no user prompts, no system keychain.

## Repository setup

To use the plugin in your repository, configure two **Copilot variables** (Settings → Copilot → Variables) on the repository, organization, or enterprise level:

| Variable                            | Value                                                   |
| ----------------------------------- | ------------------------------------------------------- |
| `COPILOT_MCP_SONARQUBE_ORG`         | Your SonarQube Cloud organization key                   |
| `COPILOT_MCP_SONARQUBE_PROJECT_KEY` | The SonarQube Cloud project key analyzed by this repo   |

Once both variables are set and the agent is installed, sessions will reach SonarQube Cloud automatically.

### Optional: `sonar-project.properties`

If you want the agent to fall back to a key derived from the repo (rather than from `COPILOT_MCP_SONARQUBE_PROJECT_KEY`), drop a `sonar-project.properties` file at the repo root:

```properties
sonar.projectKey=my-project
sonar.projectName=My Project
sonar.projectVersion=1.0
sonar.sources=src
sonar.sourceEncoding=UTF-8
```

The agent resolves project keys in the following order: user-provided argument → `.sonarlint/connectedMode.json` → repo config files (`sonar-project.properties`, `pom.xml`, Gradle, `package.json`) → CI files (`.github/workflows/*.yml`, etc.) → `search_my_sonarqube_projects` when only a project name is given.

## Usage

Invoke skills from chat with their slash form. The agent will pick a skill automatically based on the request when phrasing is clear; you can also call them explicitly.

### Quality gate

```
/sonarqube:sonar-quality-gate
/sonarqube:sonar-quality-gate my-project --branch main
/sonarqube:sonar-quality-gate my-project --pr 42
```

### List issues

```
/sonarqube:sonar-list-issues
/sonarqube:sonar-list-issues my-project --severity HIGH,BLOCKER
/sonarqube:sonar-list-issues my-project --qualities SECURITY
/sonarqube:sonar-list-issues my-project --component src/auth/login.py
/sonarqube:sonar-list-issues my-project --pr 42
```

### Fix an issue

```
/sonarqube:sonar-fix-issue java:S1481 src/main/java/MyClass.java:42
/sonarqube:sonar-fix-issue python:S2077 src/auth/login.py:34
```

### Analyze a file

```
/sonarqube:sonar-analyze
/sonarqube:sonar-analyze src/auth/login.py
```

### Coverage

```
/sonarqube:sonar-coverage
/sonarqube:sonar-coverage --max 50
/sonarqube:sonar-coverage --file src/auth/login.py
/sonarqube:sonar-coverage --pr 42
```

### Duplication

```
/sonarqube:sonar-duplication
/sonarqube:sonar-duplication my-project --pr 42
/sonarqube:sonar-duplication my-project --file src/auth/login.py
```

### Dependency risks (Advanced Security)

```
/sonarqube:sonar-dependency-risks
/sonarqube:sonar-dependency-risks my-project --pr 42
```

Requires SonarQube Advanced Security on the connected organization (SonarQube Cloud Enterprise plan).

### List projects

```
/sonarqube:sonar-list-projects
/sonarqube:sonar-list-projects my-project
```

## What the agent does beyond skills

When a user request doesn't map cleanly to one of the eight skills, the agent calls SonarQube MCP tools directly — for example:

- Explaining a rule (`show_rule`)
- Fetching arbitrary metric values (`get_component_measures`, `search_metrics`)
- Listing quality gates configured in the organization (`list_quality_gates`)
- Working with Security Hotspots (`search_security_hotspots`, `show_security_hotspot`, `change_security_hotspot_status`)
- Accepting / marking false-positive / reopening an issue (`change_sonar_issue_status`)
- Pre-flight dependency checks before manifest edits (`check_dependency`) — **always called before adding or upgrading a third-party package**
- Cross-module code navigation and architecture-compliance checks (`get_current_architecture`, `get_intended_architecture`, `get_references`, `get_type_hierarchy`, `get_upstream_call_flow`, `get_downstream_call_flow`, `get_source_code`, `search_by_signature_patterns`, `search_by_body_patterns`) — preferred over `grep`/`find` in supported languages

See `agents/main.agent.md` for the full routing rules and operating principles.

## License

Copyright (C) 2025-2026 SonarSource Sàrl. Licensed under [SSAL-1.0](LICENSE).

## Support

- Issues: https://community.sonarsource.com/
