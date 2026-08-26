# codeql-central-config

**Centrally-maintained CodeQL configuration + a reusable code-scanning workflow for every repository in `callmegreg-demo-org`.**

This repository is the single source of truth for how CodeQL runs across the org. It hosts:

| File | Purpose |
| --- | --- |
| [`codeql/codeql-config.yml`](codeql/codeql-config.yml) | The **one** CodeQL config file every caller uses (query suites, path exclusions, query filters). |
| [`.github/workflows/codeql-reusable.yml`](.github/workflows/codeql-reusable.yml) | The **reusable workflow** callers invoke. It points CodeQL at the config file above. |

Change the config once, on `main`, and **every caller repository picks it up on its next scan** — no per-repo edits required.

---

## How it works

```
 ┌─────────────────────────────────────────┐         ┌──────────────────────────────────────┐
 │  codeql-central-config  (this repo)      │         │  codeql-caller-demo  (any caller)     │
 │                                          │         │                                       │
 │  .github/workflows/codeql-reusable.yml ──┼──uses──▶│  .github/workflows/codeql.yml         │
 │  codeql/codeql-config.yml  ◀─────────────┼─remote──┤     (12 lines: just calls the         │
 │        (query suites, paths-ignore, …)   │ config  │      reusable workflow)               │
 └─────────────────────────────────────────┘         └──────────────────────────────────────┘
```

The reusable workflow loads the config with CodeQL's **external-repository** syntax:

```yaml
- uses: github/codeql-action/init@v3
  with:
    languages: ${{ inputs.language }}
    config-file: remote=callmegreg-demo-org/codeql-central-config@main:codeql/codeql-config.yml
    external-repository-token: ${{ secrets.CODEQL_CONFIG_TOKEN }}
```

`remote=OWNER/REPO@REF:FILEPATH` tells CodeQL to fetch the config file from another repository instead of the one being scanned. That single indirection is what makes the config **central**.

---

## Onboard a caller repository (the whole job)

Add this file to any repo you want scanned — that's the entire integration:

```yaml
# .github/workflows/codeql.yml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '30 5 * * 1'
  workflow_dispatch:

permissions:
  contents: read
  security-events: write
  actions: read

jobs:
  codeql:
    uses: callmegreg-demo-org/codeql-central-config/.github/workflows/codeql-reusable.yml@main
    with:
      language: javascript-typescript   # or python, java-kotlin, go, csharp, ruby, swift, actions
    secrets: inherit                    # passes CODEQL_CONFIG_TOKEN through (see below)
```

A working example lives in **[`callmegreg-demo-org/codeql-caller-demo`](https://github.com/callmegreg-demo-org/codeql-caller-demo)**.

---

## Required configuration ⚙️

Because this repo and the callers are **private**, three one-time settings are needed. (See the [visibility matrix](#visibility-matrix) for how this changes if the repos are public or internal.)

### 1. Let org repos use this private reusable workflow

By default a private repo's workflows can't be reused by other repos. Enable sharing here:

- **UI:** this repo → **Settings → Actions → General → Access** → **“Accessible from repositories in the `callmegreg-demo-org` organization.”**
- **API:**
  ```bash
  gh api -X PUT /repos/callmegreg-demo-org/codeql-central-config/actions/permissions/access \
    -f access_level=organization
  ```

### 2. Provide a token so callers can read this private config file

A caller's built-in `GITHUB_TOKEN` is scoped to the **caller** repo only, so it cannot read the config file that lives here. Supply a read-only token via `external-repository-token`:

1. Create a **fine-grained PAT** (recommended):
   - **Resource owner:** `callmegreg-demo-org`
   - **Repository access:** only `codeql-central-config`
   - **Permissions:** **Contents → Read-only**
   - *(A classic PAT with the `repo` scope also works but is broader.)*
2. Store it as an **organization Actions secret** named `CODEQL_CONFIG_TOKEN`, and scope its visibility to the caller repos (or “All repositories”):
   ```bash
   gh secret set CODEQL_CONFIG_TOKEN --org callmegreg-demo-org --visibility all --body '<PAT>'
   # or scope to selected repos:
   # gh secret set CODEQL_CONFIG_TOKEN --org callmegreg-demo-org --repos codeql-caller-demo --body '<PAT>'
   ```
   An org secret means **every** caller inherits it — nothing to configure per repo. (You can also set it as a repo secret on a single caller with `gh secret set CODEQL_CONFIG_TOKEN --repo callmegreg-demo-org/codeql-caller-demo`.)

> The reusable workflow declares this secret as **optional**, so public/internal setups that don't need it still work.

### 3. Enable Code Security (GHAS) on each caller

Code scanning on a **private or internal** repo requires **Code Security** (part of GitHub Advanced
Security). Public repos skip this entirely.

In `callmegreg-demo-org`, Code Security is governed by **Security Configurations**, not a per-repo
toggle (org → **Settings → Advanced Security → Configurations**). Attach a configuration whose
`advanced_security` includes **Code Security** to the caller repos. Prefer one whose
`code_scanning_default_setup` is **not_set/disabled** so it doesn't fight this advanced-setup workflow:

```bash
# Find a configuration that enables Code Security but NOT default setup:
gh api /orgs/callmegreg-demo-org/code-security/configurations \
  --jq '.[] | select(.advanced_security=="enabled" or .advanced_security=="code_security")
             | {id, name, advanced_security, code_scanning_default_setup}'

# Attach it to the caller (requires org-owner + a token with the admin:org scope):
REPO_ID=$(gh api /repos/callmegreg-demo-org/codeql-caller-demo --jq .id)
gh api -X POST /orgs/callmegreg-demo-org/code-security/configurations/<CONFIG_ID>/attach \
  -f scope=selected -F 'selected_repository_ids[]='"$REPO_ID"
```

> ⚠️ **This org's current state:** `codeql-caller-demo` is presently covered by an **enterprise-enforced**
> configuration (`testsully1`) that grants only **Secret Protection** and **disables code scanning**.
> Enterprise-enforced configurations **cannot be overridden** by an org owner, so private code scanning
> here is blocked until an **enterprise owner** applies a Code-Security-enabled configuration to (or
> excludes) these repos. Making the repo **internal does not help** — internal still requires Code
> Security. Only making the caller **public** removes the requirement.

*(The `codeql-central-config` repo itself does **not** need Code Security — it only stores config and a
workflow; it is never scanned.)*

---

## Visibility matrix

The user asked for the repos to be **as private as possible**. Private works end-to-end with the config above. Here's what changes by visibility, so you can relax it if desired:

| Repo visibility | Reusable workflow sharing | `CODEQL_CONFIG_TOKEN` needed? | Code Security (GHAS) needed? |
| --- | --- | --- | --- |
| **Public** | Automatic | ❌ No (public config is world-readable) | ❌ No (code scanning is free) |
| **Internal** | Automatic across the enterprise | ✅ Yes (advanced setup can't read internal config without a token) | ✅ Yes |
| **Private** *(this demo)* | Must enable **Access → organization** (step 1) | ✅ Yes (step 2) | ✅ Yes (step 3) |

**Bottom line:** the central pattern works with **private** repos — you do **not** need *internal*
(internal wouldn't reduce any requirement here). The extra cost of private is the three settings above.
The one thing outside an org owner's control in this org is **Code Security enablement**, which is
currently blocked by an enterprise-enforced configuration (see step 3). If you need a green run
immediately without enterprise involvement, make the **caller** public (only public — not internal —
removes the Code Security requirement), or use the no-token / self-contained options below.

---

## No-token alternative (inline config, still central)

If you don't want to manage a PAT, keep the config central by embedding it in the reusable workflow with the `config:` input instead of `config-file:`. The config still lives in this repo (in the reusable workflow), so it's still maintained in one place — you just lose the separate `.yml` file:

```yaml
- uses: github/codeql-action/init@v3
  with:
    languages: ${{ inputs.language }}
    config: |
      name: "Central CodeQL config"
      queries:
        - uses: security-extended
      paths-ignore:
        - '**/node_modules/**'
```

No `external-repository-token` required, because nothing is fetched cross-repo. Trade-off: config is expressed as a YAML string in the workflow rather than a standalone file.

---

## Verify it's working

1. In a caller repo, open **Actions** → the **CodeQL** run → the **Analyze** job.
2. In the *Initialize CodeQL* step logs you'll see the config being loaded from this repo, e.g. *“Loaded configuration … codeql-central-config …”*.
3. Open the caller's **Security → Code scanning** tab to see alerts produced under the central policy.

---

## Updating the central policy

Edit [`codeql/codeql-config.yml`](codeql/codeql-config.yml) on `main`. Every caller uses the new settings on its next run — no PRs to caller repos, no version bumps. Pin callers to `@main` for always-latest, or to a tag/SHA (e.g. `@v1`) if you want callers to adopt changes deliberately.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `Resource not accessible by integration` / config file 404 in *Initialize CodeQL* | `CODEQL_CONFIG_TOKEN` missing or lacks **Contents: Read** on this repo (step 2). Confirm callers use `secrets: inherit`. |
| `error parsing called workflow … not found` | Reusable-workflow access not granted (step 1), or the `uses:` path/ref is wrong. |
| `Advanced Security must be enabled for this repository` | Enable GHAS on the **caller** (step 3). |
| Alerts don't appear | Ensure the caller job has `security-events: write` and the run finished the *Perform CodeQL Analysis* step. |
