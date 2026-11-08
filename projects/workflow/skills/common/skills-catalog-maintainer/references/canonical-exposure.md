# Canonical Source and Exposure Model

## Canonical source

This repository is the authoring source of truth.

- edit skills only inside `skills/`
- edit app exposure rules in `config/apps.json`
- edit sync/copy legacy mappings in `config/sync-map.sh`

## Exposure targets

Exposure targets are consumer locations such as:

- `~/.copilot/skills`
- `~/.claude/skills`
- `~/.agents/skills`
- OpenCode managed config targets

They are install/sync outputs, not places to maintain canonical content by hand.

## Rules

- one canonical skill name per directory
- one canonical source path per skill
- no duplicate skill names within a single app exposure set
- if a rename happens, update routing references and managed config in the same change
