# apm-marketplace

An **APM (Agent Package Manager) marketplace** that publishes one package,
`team-toolkit`, made of three primitives:

| Primitive | File | What it does |
|-----------|------|--------------|
| Skill | `.apm/skills/pr-description/SKILL.md` | Auto-activates to draft PR descriptions |
| Agent | `.apm/agents/code-reviewer.agent.md` | `@code-reviewer` persona that critiques diffs |
| Instruction | `.apm/instructions/commit-style.instructions.md` | Commit-message rules (`applyTo: "**"`) |

This repo is a **hybrid**: it is both the package (source in `.apm/`) and the
marketplace index (the `marketplace:` block in [apm.yml](apm.yml), compiled to
[.claude-plugin/marketplace.json](.claude-plugin/marketplace.json)).

Consumers install it from the companion repo
[apm-consumer](https://github.com/yelghali/apm-consumer).

> Full walkthrough — publishing, consuming with **GitHub Copilot** and
> **Claude Code**, plus the **GitLab** variant — is in [POC.md](POC.md).

## Quick reference

```bash
# Producer (this repo) — publish
apm marketplace check          # validate the marketplace block
apm pack                       # build .claude-plugin/marketplace.json
git tag v1.0.0 && git push --tags

# Consumer — install
apm marketplace add yelghali/apm-marketplace
apm install team-toolkit@apm-marketplace --target copilot   # or --target claude
```
