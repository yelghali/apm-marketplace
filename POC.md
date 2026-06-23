# APM Marketplace POC — Publish & Consume Agents / Skills / Instructions

This POC demonstrates the full **Agent Package Manager (APM)** loop:

1. **Publish** a package of primitives (an **agent**, a **skill**, and an
   **instruction**) to a **marketplace**.
2. **Consume** that package from another repo and have the primitives appear in
   **GitHub Copilot**.

| Role | Repo |
|------|------|
| Producer / Marketplace | https://github.com/yelghali/apm-marketplace (this repo) |
| Consumer | https://github.com/yelghali/apm-consumer |

APM docs: https://microsoft.github.io/apm/

---

## 0. Install APM

```powershell
# Windows
irm https://aka.ms/apm-windows | iex
apm --version
```

```bash
# macOS / Linux
curl -sSL https://aka.ms/apm-unix | sh
apm --version
```

> **Locked-down Windows note (this environment).** The native binary is blocked
> by an application-control policy (AppLocker / WDAC) when launched from a
> user-writable path. The installer automatically falls back to **pip**:
>
> ```powershell
> pip install --user apm-cli
> # add the scripts dir to PATH if `apm` is not found:
> $env:Path += ";$env:APPDATA\Python\Python312\Scripts"
> apm --version
> ```

---

## 1. Author the package (producer)

A package is a directory with `apm.yml` plus primitives under `.apm/`:

```
apm-marketplace/
├── apm.yml                                   # manifest + marketplace: block
└── .apm/
    ├── skills/pr-description/SKILL.md         # auto-activating skill
    ├── agents/code-reviewer.agent.md          # @code-reviewer persona
    └── instructions/commit-style.instructions.md  # applyTo "**"
```

- **Skill** — `SKILL.md` with `name` + `description` frontmatter. The runtime
  auto-activates it based on the description.
- **Agent** — `*.agent.md` with `name` + `description`. Invoked on demand
  (`@code-reviewer`).
- **Instruction** — `*.instructions.md` with `description` + **`applyTo`** glob
  (required — without it the rule is folded into a global context file instead
  of a per-file rule).

`includes: auto` in `apm.yml` ships everything under `.apm/` to consumers.

Validate before shipping:

```powershell
apm compile --validate
```

---

## 2. Turn the repo into a marketplace (producer)

The `marketplace:` block in [apm.yml](apm.yml) makes this repo a curated index.
It lists one package (`team-toolkit`) whose source is this same repo:

```yaml
marketplace:
  owner:
    name: yelghali
    url: https://github.com/yelghali
  outputs:
    claude: {}                       # marketplace index format (Anthropic-compatible spec)
  claude:
    output: .claude-plugin/marketplace.json
  packages:
    - name: team-toolkit
      description: Code-review agent, PR-description skill, and commit-message instructions.
      source: yelghali/apm-marketplace
      version: "^1.0.0"
```

> The `outputs`/`claude` keys select APM's marketplace **index format**. APM
> reuses the open Anthropic marketplace spec, so the generated index lives at
> `.claude-plugin/marketplace.json`. This is the marketplace mechanism itself —
> `apm marketplace add` auto-detects that path — and is independent of which
> coding agent ultimately consumes the package.

Scaffold it yourself with `apm marketplace init --owner yelghali`.

---

## 3. Publish (producer)

`apm pack` resolves each package's version range against the **git tags** of its
source repo, so push and tag **before** packing:

```powershell
git add apm.yml .apm/ .gitignore README.md
git commit -m "feat: team-toolkit package + marketplace index"
git push -u origin main

git tag v1.0.0
git push origin v1.0.0          # the tag MUST exist before `apm pack`

apm marketplace check           # every package ref/range resolves -> green
apm pack                        # writes the marketplace index

git add .claude-plugin/marketplace.json
git commit -m "build: marketplace index"
git push
```

The generated marketplace index pins the package to the resolved tag + commit
SHA. It is the standard APM marketplace artifact that `apm marketplace add`
reads.

---

## 4. Consume in GitHub Copilot (consumer repo)

> ⚠️ **Avoid the hang.** `apm marketplace add` / `apm install` shell out to git.
> If Git Credential Manager pops an interactive prompt, the command hangs. Make
> git non-interactive first (a public repo needs no auth):
>
> ```powershell
> $env:GIT_TERMINAL_PROMPT = "0"
> $env:GCM_INTERACTIVE     = "never"
> ```
>
> For **private** marketplaces/packages, set a token instead:
> `$env:GITHUB_APM_PAT = "<fine-grained-PAT-with-read>"`.

```powershell
git clone https://github.com/yelghali/apm-consumer.git
cd apm-consumer
apm init -y

# Register the marketplace (give it a clean alias; default alias = manifest name)
apm marketplace add yelghali/apm-marketplace --name yelghali-tools
apm marketplace browse yelghali-tools          # shows: team-toolkit

# Install the package from the marketplace, deploy to GitHub Copilot
apm install team-toolkit@yelghali-tools --target copilot
```

### Where the primitives land (GitHub Copilot)

| Primitive | Path |
|-----------|------|
| Skill | `.agents/skills/pr-description/SKILL.md` |
| Agent | `.github/agents/code-reviewer.agent.md` |
| Instruction | `.github/instructions/commit-style.instructions.md` |

`.agents/skills/` is the cross-client skill location (Copilot, Cursor, Codex,
Gemini, OpenCode all read it). Agents and instructions deploy under `.github/`
for Copilot.

### What gets committed

| File | Commit? | Why |
|------|---------|-----|
| `apm.yml` | ✅ | Declares the dependency |
| `apm.lock.yaml` | ✅ | Pins commit SHA + content hashes |
| `.github/`, `.agents/` | ✅ | Deployed primitives — available on clone |
| `apm_modules/` | ❌ | Cache; rebuilt by `apm install` (auto-gitignored) |

> The consumer's `apm.yml` records the dependency as the **concrete git ref**
> `yelghali/apm-marketplace#v1.0.0`. A teammate who clones the consumer repo only
> needs `apm install` — they do **not** need the marketplace registered locally.
> The marketplace registration is only required for the first
> `install <pkg>@<marketplace>` resolution.

---

## 5. Use it

**GitHub Copilot** (VS Code): open the consumer repo. Ask *"draft a PR
description for my staged changes"* → the `pr-description` skill activates.
Type `@code-reviewer review my staged changes` → the agent runs. The
`commit-style` instruction applies automatically to all files.

---

## 6. Update flow

Producer ships a new version:

```powershell
# bump version in apm.yml to 1.1.0, edit primitives
git commit -am "feat: v1.1.0"; git push
git tag v1.1.0; git push origin v1.1.0
apm pack; git commit -am "build: marketplace index v1.1.0"; git push
```

Consumer pulls it:

```powershell
apm update                      # refresh to latest matching ref (consent prompt)
apm install --target copilot    # redeploy
```

---

## 7. GitLab variant (production target)

The actual production setup uses **GitLab** repos with **GitHub Copilot**. Only
the host and authentication change — the package layout, primitives, and
`apm pack` step are identical.

### 7a. Producer on GitLab

`apm.yml` is unchanged except the marketplace package `source`. Point it at the
GitLab path (use `sourceBase` when several packages share a base):

```yaml
marketplace:
  owner:
    name: acme
    url: https://gitlab.com/acme
  sourceBase: https://gitlab.com/acme/agent-marketplace   # optional shared base
  outputs:
    claude: {}                       # marketplace index format (host-independent)
  claude:
    output: .claude-plugin/marketplace.json
  packages:
    - name: team-toolkit
      source: https://gitlab.com/acme/team-toolkit.git     # or relative to sourceBase
      version: "^1.0.0"
```

Publish exactly as in step 3 (`git push`, `git tag v1.0.0`, `git push --tags`,
`apm pack`, commit the marketplace index). For **private** GitLab repos set a
token so `apm pack`'s `git ls-remote` is non-interactive:

```bash
export GITLAB_APM_PAT=glpat-xxxxxxxx          # or GITLAB_TOKEN
# self-managed GitLab also needs the host:
export GITLAB_HOST=gitlab.acme.internal       # single host
# or multiple:
export APM_GITLAB_HOSTS=gitlab.acme.internal,gitlab.partner.io
```

### 7b. Consumer on GitLab (GitHub Copilot)

```bash
export GITLAB_APM_PAT=glpat-xxxxxxxx
# self-managed only:
export GITLAB_HOST=gitlab.acme.internal

# gitlab.com shorthand
apm marketplace add gitlab.com/acme/agent-marketplace --host gitlab.com --name acme-tools
# self-managed / pinned ref / deep subgroups → full HTTPS URL
apm marketplace add https://gitlab.acme.internal/acme/agent-marketplace.git#v1.0.0 --name acme-tools

apm install team-toolkit@acme-tools --target copilot
```

Direct dependency form (no marketplace) in a consumer `apm.yml`:

```yaml
dependencies:
  apm:
    - gitlab.com/acme/team-toolkit#v1.0.0
    # self-managed FQDN:
    - gitlab.acme.internal/acme/team-toolkit#v1.0.0
    # deep subgroups that shorthand can't disambiguate → object form:
    - git: https://gitlab.com/acme/platform/team/team-toolkit.git
      ref: v1.0.0
```

### GitHub vs GitLab — what changes

| Concern | GitHub | GitLab |
|---------|--------|--------|
| Marketplace add | `apm marketplace add owner/repo` | `apm marketplace add gitlab.com/owner/repo --host gitlab.com` (or full URL for self-managed) |
| Dependency ref | `owner/repo#v1.0.0` | `gitlab.com/owner/repo#v1.0.0` / `gitlab.acme.internal/...` |
| Auth token | `GITHUB_APM_PAT` | `GITLAB_APM_PAT` (or `GITLAB_TOKEN`) |
| Self-managed host | `GITHUB_HOST` (GHES) | `GITLAB_HOST` or `APM_GITLAB_HOSTS` |
| Package layout, `apm pack`, deploy dirs | identical | identical |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `apm marketplace add` / `apm install` hangs | Git Credential Manager interactive prompt | `$env:GIT_TERMINAL_PROMPT="0"; $env:GCM_INTERACTIVE="never"` (public) or set the right `*_APM_PAT` (private) |
| `apm` not found after install | pip scripts dir not on PATH | `$env:Path += ";$env:APPDATA\Python\Python312\Scripts"` |
| `apm pack` can't resolve a version range | tag not pushed yet | `git push origin v1.0.0` **before** `apm pack`, or pin with `ref:` instead of `version:` |
| Marketplace alias clashes with package name | alias defaults to `manifest.name` | `apm marketplace add … --name <distinct-alias>` |
| `NOASSERTION` license warning on `apm pack` | no `license:` field | add `license: MIT` (or your SPDX id) to `apm.yml` |
