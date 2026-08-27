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
- uses: github/codeql-action/init@cdf488f595d80d6e07e03d4674febd5ab45fa938 # v4.37.9
  with:
    languages: ${{ inputs.language }}
    config-file: remote=callmegreg-demo-org/codeql-central-config@main:codeql/codeql-config.yml
    external-repository-token: ${{ steps.config-token.outputs.token }}  # short-lived GitHub App token
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
    secrets: inherit                    # passes the app secrets through (see step 2 below)
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

### 2. Give callers read access to this private config via a GitHub App

A caller's built-in `GITHUB_TOKEN` is scoped to the **caller** repo only, so it cannot read the config
file that lives here. Rather than a long-lived PAT, this setup uses a **GitHub App** and mints a
**short-lived installation token** at runtime with
[`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) — the reusable
workflow already does this.

1. **Create a GitHub App** owned by `callmegreg-demo-org`, least privilege:
   - **Repository permissions → Contents: Read-only** — the *only* permission needed (Metadata: Read is implicit).
   - **Webhook:** disabled. Name/homepage URL are arbitrary.
   - *(This demo's app is **CodeQL Config Reader (cmg-demo)**, Client ID `Iv23liqGU38HnNgNpGSi`.)*
2. **Install the app on only `codeql-central-config`** (Install → *Only select repositories* → `codeql-central-config`). It never needs access to any caller repo.
3. **Generate a private key** (App settings → *Private keys* → *Generate a private key* → downloads a `.pem`). Note the app's **Client ID** (shown on the app's *General* page, format `Iv23…`).
4. **Store two organization Actions secrets**, visibility *Private repositories* (or *Selected* → the callers):
   ```bash
   gh secret set CODEQL_CONFIG_APP_CLIENT_ID   --org callmegreg-demo-org --visibility private --body '<CLIENT_ID>'
   gh secret set CODEQL_CONFIG_APP_PRIVATE_KEY --org callmegreg-demo-org --visibility private < app-private-key.pem
   ```
   Org secrets mean **every** private caller inherits them — nothing to configure per repo.

The reusable workflow then mints and uses the token (already wired up):

```yaml
- name: Get token to read the central config
  id: config-token
  uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
  with:
    client-id: ${{ secrets.CODEQL_CONFIG_APP_CLIENT_ID }}
    private-key: ${{ secrets.CODEQL_CONFIG_APP_PRIVATE_KEY }}
    owner: callmegreg-demo-org
    repositories: codeql-central-config
- uses: github/codeql-action/init@cdf488f595d80d6e07e03d4674febd5ab45fa938 # v4.37.9
  with:
    config-file: remote=callmegreg-demo-org/codeql-central-config@main:codeql/codeql-config.yml
    external-repository-token: ${{ steps.config-token.outputs.token }}
```

> **Why an App instead of a PAT?** The token is **short-lived (~1h)**, **scoped to a single repo**, tied
> to no individual person, and the key can be rotated centrally. The secrets are declared **optional**,
> so public/internal setups that don't need them still work.

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

| Repo visibility | Reusable workflow sharing | App token needed? | Code Security (GHAS) needed? |
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
- uses: github/codeql-action/init@cdf488f595d80d6e07e03d4674febd5ab45fa938 # v4.37.9
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

## Advanced: branch on the caller's repository custom properties

**Yes — a reusable workflow can read the _calling_ repo's [custom property](https://docs.github.com/en/organizations/managing-organization-settings/managing-custom-properties-for-repositories-in-your-organization) values and use them to steer its own logic.** This lets you centralize *policy* ("Critical apps get a deeper scan, low-risk repos can opt out") while callers only supply *metadata* — no per-repo YAML.

**How it works:** the reusable workflow runs in the **caller's** context, so `github.repository` is the caller. Repo custom property values are readable by anyone with **read access to the repo**, which the built-in `GITHUB_TOKEN` already has — so this needs **no extra secret or App** (that App is only for reading this central *config* repo):

```yaml
- name: Derive scan policy from caller custom properties
  id: policy
  env:
    GH_TOKEN: ${{ github.token }}          # built-in token; repo-read is enough
    PROP: Application_Business_Criticality # org-wide property name (hardcoded, not an input)
  run: |
    set -euo pipefail

    # 1. READ the caller's value for that property (empty if unset).
    value="$(gh api "/repos/${GITHUB_REPOSITORY}/properties/values" \
      --jq '.[] | select(.property_name == env.PROP) | .value')"

    # 2. MAP the value to a policy: scan or not, and which config.
    case "$value" in
      Critical | High) should_scan=true;  config_path=codeql/codeql-config-strict.yml ;;  # deeper
      Low)             should_scan=false; config_path=codeql/codeql-config.yml ;;         # opt out
      *)               should_scan=true;  config_path=codeql/codeql-config.yml ;;         # default
    esac

    # 3. PUBLISH for later steps.
    echo "should_scan=$should_scan" >> "$GITHUB_OUTPUT"
    echo "config_path=$config_path" >> "$GITHUB_OUTPUT"
```

Later steps then consume those outputs — pick the config file, and gate the scan:

```yaml
- name: Initialize CodeQL
  if: steps.policy.outputs.should_scan == 'true'
  uses: github/codeql-action/init@cdf488f595d80d6e07e03d4674febd5ab45fa938 # v4.37.9
  with:
    languages: ${{ inputs.language }}
    config-file: remote=callmegreg-demo-org/codeql-central-config@main:${{ steps.policy.outputs.config_path }}
    external-repository-token: ${{ steps.config-token.outputs.token }}
# ...Autobuild / Analyze also carry `if: steps.policy.outputs.should_scan == 'true'`
```

This repo ships a **working implementation** of exactly this in
[`.github/workflows/codeql-reusable.yml`](.github/workflows/codeql-reusable.yml). The property name
(`Application_Business_Criticality`) is **hardcoded** in the policy step's `PROP` constant — in a
centralized model it's a fixed org-wide schema constant, so callers supply only its *value*, never
the name (no workflow input needed). It selects between two configs —
[`codeql/codeql-config.yml`](codeql/codeql-config.yml) (default) and
[`codeql/codeql-config-strict.yml`](codeql/codeql-config-strict.yml) (`security-and-quality`).

| Caller's `Application_Business_Criticality` | Result |
| --- | --- |
| `Critical` / `High` | Strict config (`security-and-quality`) |
| `Medium` *(or unset / any other value)* | Default config (`security-extended`) |
| `Low` | **Scan skipped** — job still succeeds, logs a `::notice::` |

**Set the property** on a caller (org owners, or repo admins if the property is `values_editable_by: org_and_repo_actors`):

```bash
gh api -X PATCH /repos/OWNER/CALLER-REPO/properties/values \
  -f 'properties[][property_name]=Application_Business_Criticality' \
  -f 'properties[][value]=High'
```

**Notes & caveats:**
- Retarget the policy by editing the `PROP` constant in the workflow's policy step — point it at whatever governance property your org already defines. Nothing here is CodeQL-specific; the same pattern works for a boolean opt-in (`code-scanning`), a team/owner tag, environment, etc. *(If you genuinely need per-caller property names — e.g. sharing this workflow across orgs with different schemas — promote `PROP` back to a `workflow_call` input. That's rarely wanted in a single-org centralized model.)*
- The `config-file` still reads from `@main`, so the chosen config file (e.g. the strict one) must exist on `main`. If you add a new profile, land it on `main` before pointing callers at it.
- `gh` is preinstalled on GitHub-hosted runners; on self-hosted runners install the [GitHub CLI](https://github.com/cli/cli) or swap the step for a `curl` + `jq` call to the same endpoint.
- A single scan **job** that self-skips still reports success (green) — desirable for required checks. If you'd rather the check not appear at all for opted-out repos, gate at the *caller* with a job-level `if:` instead.

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
| `Resource not accessible by integration` / config file 404 in *Initialize CodeQL* | App secrets missing, the app isn't installed on `codeql-central-config`, or it lacks **Contents: Read** (step 2). Confirm callers use `secrets: inherit`. |
| `error parsing called workflow … not found` | Reusable-workflow access not granted (step 1), or the `uses:` path/ref is wrong. |
| `Advanced Security must be enabled for this repository` | Enable GHAS on the **caller** (step 3). |
| Alerts don't appear | Ensure the caller job has `security-events: write` and the run finished the *Perform CodeQL Analysis* step. |
| Config 404 only for high-criticality callers (custom-property example) | The selected profile (e.g. `codeql/codeql-config-strict.yml`) must exist on `main` — the `config-file` reads from `@main`. |
| Custom-property step shows `<unset>` unexpectedly | Property name is case-sensitive and must match the org/enterprise schema exactly; confirm it's assigned to the caller repo. |
