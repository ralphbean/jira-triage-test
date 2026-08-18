# Jira Triage Functional Test Plan

End-to-end validation that a fresh repo, enrolled in fullsend with Jira
support from the `jira` branch, can poll a Jira Cloud project and dispatch
triage against a real issue.

## Prerequisites

- A Jira Cloud site (`https://<site>.atlassian.net`) with at least one
  project containing an open issue you can mutate.
- A Jira Cloud API token for a user with edit permissions on that project
  ([Create API token](https://id.atlassian.com/manage-profile/security/api-tokens)).
- `gh` CLI authenticated (`gh auth status`).
- `fullsend` CLI on your `PATH`
  ([Running agents locally](https://fullsend.sh/docs/guides/user/running-agents-locally.html)).
- `openshell`, `openshell-gateway`, and `podman` installed per `LOCAL.md`.
- A GitHub org where you can create a test repo and configure secrets.

---

## Step 1 — Create a test repo and install fullsend via the public mint

This is normal fullsend enrollment — nothing Jira-specific yet.

```bash
# Create a fresh repo
gh repo create <your-org>/jira-triage-test --public --clone
cd jira-triage-test
git commit --allow-empty -m "initial commit"
git push -u origin main
```

Enroll the repo with fullsend:

```bash
fullsend inference provision <repo> --project=blah
fullsend github setup <repo> --inference-project=blah
```

This scaffolds `.github/workflows/fullsend.yaml` (the shim workflow) and
registers the repo with the token mint. Verify:

```bash
cat .github/workflows/fullsend.yaml   # should exist, managed by fullsend
```

Commit and push the scaffolded files if `fullsend` didn't push them
automatically.

---

## Step 2 — Point triage at the `jira` branch implementation

The default enrollment ships the `main` branch of `fullsend-ai/agents`.
The Jira triage support lives on the `jira` branch, so you need to
override the triage harness URL to point at that branch. You also need
to configure Jira workflow transition names — these are project-specific
configuration, not credentials, so they belong in the harness via
`base:` composition rather than in CI secrets.

### Register the upstream harness and pin the SHA

The `agents:` list in `.fullsend/config.yaml` accepts raw GitHub URLs
pinned with a SHA-256 integrity hash:

```bash
# Pin the jira branch harness via CLI (auto-computes SHA-256)
fullsend agent add \
  "https://github.com/fullsend-ai/agents/blob/jira/harness/triage.yaml" \
  --fullsend-dir .fullsend
```

Or manually:

```bash
HARNESS_SHA=$(curl -sfL \
  "https://raw.githubusercontent.com/fullsend-ai/agents/jira/harness/triage.yaml" \
  | sha256sum | awk '{print $1}')
echo "Harness SHA: ${HARNESS_SHA}"
```

### Jira transition defaults

The upstream harness ships sensible defaults for Jira workflow
transitions in `forge.jira.env.runner`:

| Variable | Default |
|----------|---------|
| `JIRA_DUPLICATE_TRANSITION` | `Duplicate` |
| `JIRA_NOT_PLANNED_TRANSITION` | `Not Planned` |
| `JIRA_SPLIT_TRANSITION` | `Obsolete` |

If these match your Jira project's workflow, no override is needed. If
not, create a local harness override via `base:` composition.

Create `.fullsend/harness/triage.yaml`:

```yaml
# .fullsend/harness/triage.yaml
base: https://raw.githubusercontent.com/fullsend-ai/agents/jira/harness/triage.yaml#sha256=<HARNESS_SHA>

forge:
  jira:
    env:
      runner:
        JIRA_DUPLICATE_TRANSITION: "Won't Do"
        JIRA_NOT_PLANNED_TRANSITION: "Rejected"
        JIRA_SPLIT_TRANSITION: "Done"
```

Replace `<HARNESS_SHA>` with the value from above.

Then update `.fullsend/config.yaml` to point at the local harness
instead of the upstream URL directly:

```yaml
# .fullsend/config.yaml
version: "1"
agents:
  # Local harness that inherits from the jira branch via base: composition
  - name: triage
    source: harness/triage.yaml
allowed_remote_resources:
  - https://raw.githubusercontent.com/fullsend-ai/agents/
```

> **How this works:** `source: harness/triage.yaml` resolves relative to
> the `.fullsend/` directory. That file's `base:` pulls in the full upstream
> harness from the `jira` branch — `forge.jira` block, Jira skills,
> scripts, and policies — then the local `forge.jira.env.runner` values
> merge on top (child wins for env map keys).

If the defaults are fine, skip the local harness and register the
upstream URL directly in `config.yaml`:

```yaml
# .fullsend/config.yaml
version: "1"
agents:
  - https://raw.githubusercontent.com/fullsend-ai/agents/jira/harness/triage.yaml#sha256=<HARNESS_SHA>
allowed_remote_resources:
  - https://raw.githubusercontent.com/fullsend-ai/agents/
```

Commit and push:

```bash
git add .fullsend/config.yaml
# If using local harness override:
# git add .fullsend/harness/triage.yaml
git commit -m "pin triage harness to jira branch"
git push
```

---

## Step 3 — Vendor a custom-built fullsend with Jira support

If the `fullsend` CLI on the public release channel doesn't yet include the
`fullsend issues post-comment --tracker jira` subcommand (needed by the
post-script), you need to vendor a custom-built binary.

Fullsend's CI automatically detects a vendored binary at
`.fullsend/bin/fullsend` in the enrolled repo and uses it instead of
downloading a release — no environment variables or workflow changes needed.

```bash
# Clone fullsend and build from main (or a branch with Jira CLI support)
git clone https://github.com/fullsend-ai/fullsend.git /tmp/fullsend-build
cd /tmp/fullsend-build
make go-build

# Verify the Jira tracker flag exists
./bin/fullsend issues post-comment --help | grep -q 'tracker'
echo "Jira tracker support confirmed in custom build"
```

Copy the binary into the test repo:

```bash
cd /path/to/jira-triage-test
mkdir -p .fullsend/bin
cp /tmp/fullsend-build/bin/fullsend .fullsend/bin/fullsend
chmod +x .fullsend/bin/fullsend
```

Commit and push:

```bash
git add .fullsend/bin/fullsend
git commit -m "vendor custom fullsend with Jira support"
git push
```

> **How this works:** `action.yml` in fullsend checks for
> `.fullsend/bin/fullsend` (per-repo) or `bin/fullsend` (per-org) before
> attempting any release download. When found, the vendored binary takes
> absolute precedence — it's copied to the runner's temp directory, added
> to `$GITHUB_PATH`, and used for `fullsend run`. No CI variables need to
> be set.
>
> You can also use `fullsend github setup --vendor --fullsend-source /tmp/fullsend-build`
> to automate the vendoring and generate a `vendor-manifest.yaml` for
> cleanup tracking.

---

## Step 4 — Install a GitHub Actions workflow to poll Jira and dispatch triage

Fullsend already has a built-in Jira poller (`fullsend poll --input-driver
jira-poll`) that produces dispatch records in the same `NormalizedEvent`
format used by GitHub and GitLab events. The poller writes a
`dispatches.json` file, and a shell loop dispatches each record to the
matching stage workflow via `gh workflow run` — reusing the same reusable
dispatch infrastructure that all other agents use.

This pattern is documented in fullsend's
[Jira Integration guide](https://fullsend.sh/docs/guides/user/jira-integration.html)
(pre-alpha). The workflow below is adapted from that guide.

Create `.github/workflows/fullsend-poll-jira.yml`:

```yaml
name: fullsend Jira poll

on:
  schedule:
    - cron: "*/5 * * * *"  # every 5 minutes
  workflow_dispatch: {}     # allow manual runs

permissions:
  actions: write
  contents: read

jobs:
  poll:
    runs-on: ubuntu-24.04
    concurrency:
      group: fullsend-jira-poll
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4

      - name: Install fullsend
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          # Use the vendored binary if present (Step 3), otherwise download
          if [[ -x ".fullsend/bin/fullsend" ]]; then
            sudo cp .fullsend/bin/fullsend /usr/local/bin/fullsend
          else
            gh release download --repo fullsend-ai/fullsend \
              -p 'fullsend_*_linux_amd64.tar.gz' -O - | tar xz
            sudo mv fullsend /usr/local/bin/
          fi

      - name: Poll Jira
        env:
          JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
          JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
          JIRA_BASE_URL: ${{ vars.JIRA_BASE_URL }}
        run: |
          fullsend poll \
            --input-driver jira-poll \
            --jira-url "${JIRA_BASE_URL}" \
            --jira-project PROJ \
            --target-repo "${{ github.repository }}" \
            --output dispatches.json \
            --fullsend-dir .fullsend

      - name: Dispatch agent workflows
        env:
          GH_TOKEN: ${{ github.token }}
          JIRA_BASE_URL: ${{ vars.JIRA_BASE_URL }}
        run: |
          set -euo pipefail

          if ! jq -e 'length > 0' dispatches.json > /dev/null 2>&1; then
            echo "No dispatches to process."
            exit 0
          fi

          dispatched=0
          count=$(jq 'length' dispatches.json)

          for i in $(seq 0 $((count - 1))); do
            record=$(jq -c ".[$i]" dispatches.json)
            STAGE=$(echo "$record" | jq -r '.stage')
            RESOURCE_KEY=$(echo "$record" | jq -r '.resource_key')
            EVENT_TYPE=$(echo "$record" | jq -r '.event_type')
            ISSUE_ID=$(echo "$record" | jq -r '.iid // 0')

            # Extract the Jira issue key from the resource key
            # (e.g. "issue-PROJ-101" -> "PROJ-101").
            ISSUE_KEY="${RESOURCE_KEY#issue-}"
            ISSUE_URL="${JIRA_BASE_URL}/browse/${ISSUE_KEY}"

            # Build a minimal event payload compatible with the scaffold
            # agent workflows. The concurrency group uses
            # fromJSON(event_payload).issue.number, so it must stay a
            # number.
            EVENT_PAYLOAD=$(jq -nc \
              --argjson number "$ISSUE_ID" \
              --arg url "$ISSUE_URL" \
              '{issue: {number: $number, html_url: $url}}')

            # Find the workflow file for this stage by scanning for the
            # "# fullsend-stage: <stage>" marker in workflow files.
            WORKFLOW_NAME=""
            for wf in .github/workflows/*.yml .github/workflows/*.yaml; do
              [[ -f "$wf" ]] || continue
              if grep -qxF "# fullsend-stage: ${STAGE}" "$wf"; then
                WORKFLOW_NAME=$(basename "$wf")
                break
              fi
            done
            if [[ -z "$WORKFLOW_NAME" ]]; then
              echo "::warning::No workflow found for stage ${STAGE}, skipping ${RESOURCE_KEY}"
              continue
            fi

            echo "Dispatching ${WORKFLOW_NAME} for ${ISSUE_KEY} (${STAGE})"
            gh workflow run "$WORKFLOW_NAME" \
              -f event_type="$EVENT_TYPE" \
              -f source_repo="${{ github.repository }}" \
              -f event_payload="$EVENT_PAYLOAD"

            dispatched=$((dispatched + 1))
          done

          echo "::notice::Dispatched ${dispatched} agent workflow(s)"
```

Replace `PROJ` with your Jira project key.

> **Why the two-step dance:** Today, `fullsend poll` only produces
> dispatch records — it doesn't trigger workflows itself. The shell loop
> that reads `dispatches.json` and calls `gh workflow run` is glue you
> write yourself. Eventually the poller will gain a built-in output
> driver that dispatches directly, eliminating this manual wiring. Until
> then, you connect the two dots in this workflow file.
>
> **How this works:** `fullsend poll` queries Jira for recently updated
> issues, detects new comments and label changes since the last poll, and
> converts each change to a `NormalizedEvent` — the same forge-neutral
> event struct that GitHub and GitLab input drivers produce. The dispatch
> loop then finds the matching stage workflow (by the `# fullsend-stage:`
> marker that `fullsend github setup` scaffolds into each agent workflow)
> and triggers it via `gh workflow run`. From that point on, the standard
> fullsend dispatch pipeline takes over — the same `action.yml`, harness
> resolution, and agent execution used by GitHub-native events.
>
> The poller uses Jira entity properties for coordination state (lock +
> checkpoint per issue, namespaced by target repo), so only changes newer
> than the last successful poll trigger dispatch.
>
> **Key caveat:** The upstream Jira integration guide notes that built-in
> agent pre/post scripts don't yet understand Jira-keyed payloads
> (#2264) — but that's exactly what the `jira` branch (Step 2) adds for
> triage. The combination of the poller (upstream) + `forge.jira` harness
> (this branch) is what this test validates end-to-end.

Commit and push:

```bash
git add .github/workflows/fullsend-poll-jira.yml
git commit -m "add Jira triage poller workflow"
git push
```

---

## Step 5 — Configure secrets, variables, and Jira credentials

The poller workflow uses GitHub Actions **secrets** for credentials and
**variables** for non-sensitive configuration. The harness (`forge.jira`
in `harness/triage.yaml`) picks up Jira credentials from the runner
environment, so the same secrets serve both the poller and the agent.

### Secrets (Settings > Secrets and variables > Actions > Secrets)

| Secret | Value | Example |
|--------|-------|---------|
| `JIRA_TOKEN` | Jira Cloud API token | `ATATT3x...` |
| `JIRA_USER_EMAIL` | Email of the Jira API user | `bot@example.com` |

### Variables (Settings > Secrets and variables > Actions > Variables)

| Variable | Value | Example |
|----------|-------|---------|
| `JIRA_BASE_URL` | Your Jira Cloud site URL | `https://mysite.atlassian.net` |

### Inference credentials

These are provisioned by `fullsend inference provision` (Step 1) — no
manual secret setup needed. The mint provides scoped tokens at runtime
via GitHub OIDC.

### Configure via CLI

```bash
# Secrets
gh secret set JIRA_TOKEN --body "<your-api-token>"
gh secret set JIRA_USER_EMAIL --body "bot@example.com"

# Variables
gh variable set JIRA_BASE_URL --body "https://mysite.atlassian.net"
```

Jira transition names (`JIRA_DUPLICATE_TRANSITION`, etc.) are configured
in the harness override (Step 2), not here.

### Custom JQL (optional)

The poller defaults to searching all non-done issues in the project,
ordered by most recently updated. To narrow the scope, edit the
`fullsend poll` invocation in the workflow to add `--jql`:

```bash
fullsend poll \
  --input-driver jira-poll \
  --jira-url "${JIRA_BASE_URL}" \
  --jira-project PROJ \
  --jql 'project = PROJ AND labels = "fullsend" AND statusCategory != Done ORDER BY updated DESC' \
  --target-repo "${{ github.repository }}" \
  --output dispatches.json \
  --fullsend-dir .fullsend
```

`--jira-project` is required alongside `--jql` if your project uses
slash commands — without it, all actors resolve to `external` and every
`/fs-*` command silently fails the role gate.

---

## Step 6 — Confirm the poller dispatches triage on a Jira issue

### 6a. Create a test issue in Jira

In your Jira Cloud project, create a new issue that matches your JQL. For
example, if your JQL filters on `status = Open`, create an issue titled:

> **Test issue for fullsend triage**
>
> This is a test issue to validate that the Jira poller triggers triage.
> The application crashes when the user clicks the submit button on the
> settings page.

### 6b. Trigger the poller manually

Don't wait for the cron schedule — trigger a manual run:

```bash
gh workflow run jira-triage-poll.yml
```

Or with a JQL override targeting your specific test issue:

```bash
gh workflow run jira-triage-poll.yml \
  -f jql_override="project = TESTPROJ AND key = TESTPROJ-42"
```

### 6c. Watch the workflow

```bash
gh run watch --exit-status
```

Or find the run in the Actions tab:

```bash
gh run list --workflow=jira-triage-poll.yml --limit 3
```

### 6d. Verify triage ran

Check the workflow logs for the triage result JSON. Look for:

1. **Poll job** found your issue key in the JQL results.
2. **Triage job** ran `fullsend run triage` with `FULLSEND_FORGE=jira`.
3. **Triage result** JSON contains a valid `action` (e.g. `ready-to-code`,
   `needs-info`, `triaged`).

### 6e. Verify Jira mutations

Go to the Jira issue in your browser and confirm:

- A **triage comment** was posted by the API user, containing the triage
  summary and decision.
- **Labels** were applied (e.g. `needs-info`, `triaged`, `bug`).
- If the action was `duplicate` or `not-planned`, the issue was
  **transitioned** to the configured status.

### 6f. Verify re-triage on update

If the initial triage returned `needs-info`:

1. Add a comment on the Jira issue answering the clarifying questions.
2. Trigger the poller again (the `updated >= -15m` JQL should now pick up
   the issue again since it was just commented on).
3. Confirm triage re-runs and the decision updates (e.g. from `needs-info`
   to `ready-to-code`).

---

## Teardown

```bash
# Delete the test repo when done
gh repo delete <your-org>/jira-triage-test --yes

# Remove the custom fullsend binary if you installed one
# sudo rm /usr/local/bin/fullsend && reinstall from release channel
```
