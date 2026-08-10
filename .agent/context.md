# Integrations — Repo Context

Thin wrapper repo: git submodule pointer(s) to individual HA integration repos
(currently `home-assistant/tariff-au` → `ha-tariff-au`). No application code of its
own.

## Org-wide rules
See `repos/.agent/context.md` (implementation-only role boundary, planning-ownership
of `planning/`, licensing/naming/branching standards, security/PII scan process).
Not repeated here. This repo is real and public
(`github.com/Open-Energy-Collective/integrations`) — run
`../.agent/scripts/security-scan.sh integrations` (`repos/.agent/security-scan.md`)
before any release.
