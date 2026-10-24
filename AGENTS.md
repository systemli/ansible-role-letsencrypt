# AGENTS.md

Ansible role that installs certbot and requests Let's Encrypt certificates on Debian and RedHat family systems.

## Layout

- `defaults/main.yml`: all role variables with defaults
- `tasks/`: `main.yml` imports `install.yml` (with `Debian.yml`/`RedHat.yml`), `account.yml` and `certificate.yml`
- `templates/`: certbot account configuration and DNS hook script templates
- `molecule/default/`: integration test (Docker, Debian 13)
- `meta/`: role metadata for Ansible Galaxy

## Checks

- `yamllint .` must pass (config in `.yamllint`)
- `ansible-lint .` should not report new findings; it creates `.ansible/`, which is git ignored, never commit it
- `molecule test` runs the role in a container; CI runs it on Debian 12, Debian 13 and Ubuntu 24.04

## Conventions

- Use fully qualified module names, quoted octal `mode` values and `true`/`false` booleans
- Give every task a `name`
- New or changed role variables need an entry in `defaults/main.yml` and the README
- GitHub Actions are pinned to commit SHAs with a `# vX.Y.Z` comment; Dependabot updates them

## Commits and PRs

- Write in English, use conventional commit titles (`feat:`, `fix:`, `chore:`, `ci:`; `!` for breaking changes)
- The PR title sets the release-drafter label (`enhancement`, `bug`, `chore`, `breaking`); a breaking change bumps the major version
- Open PRs as draft and fill in `.github/pull_request_template.md`: what changed and why, plus a breaking changes section if needed
- Do not list changed files or claim test results in the description; CI covers that
