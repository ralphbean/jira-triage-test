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
override the triage harness URL to point at that branch.

The `agents:` list in `.fullsend/config.yaml` accepts raw GitHub URLs
pinned with a SHA-256 integrity hash. Replace the default triage entry
with one pointing at the `jira` branch:

```bash
# Get the SHA-256 of the harness file on the jira branch
HARNESS_SHA=$(curl -sfL \
  "https://raw.githubusercontent.com/fullsend-ai/agents/jira/harness/triage.yaml" \
  | sha256sum | awk '{print $1}')
echo "Harness SHA: ${HARNESS_SHA}"
```

Then update (or create) `.fullsend/config.yaml`:

```yaml
# .fullsend/config.yaml
version: "1"
agents:
  # Override triage to use the jira branch instead of main
  - https://raw.githubusercontent.com/fullsend-ai/agents/jira/harness/triage.yaml#sha256=<HARNESS_SHA>
allowed_remote_resources:
  - https://raw.githubusercontent.com/fullsend-ai/agents/
```

Replace `<HARNESS_SHA>` with the value from the curl command above.

> **How this works:** When fullsend dispatches an agent, it resolves the
> harness from URLs in the `agents:` list. On name collision, config-registered
> agents take precedence over built-in agents, so this entry overrides the
> stock triage harness. The URL points at the `jira` branch, so dispatch
> picks up `harness/triage.yaml` with its `forge.jira` block, the Jira
> skills, scripts, and policies from that branch. The `allowed_remote_resources`
> entry authorizes fetching from the agents repo.

You can also use the CLI to register the override:

```bash
fullsend agent add \
  "https://github.com/fullsend-ai/agents/blob/jira/harness/triage.yaml" \
  --fullsend-dir .fullsend
```

This auto-pins the SHA-256 and updates `allowed_remote_resources`.

Commit and push:

```bash
git add .fullsend/config.yaml
git commit -m "pin triage harness to jira branch for testing"
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

Since Jira Cloud doesn't natively fire GitHub webhook events, you need a
polling workflow. Create `.github/workflows/jira-triage-poll.yml`:

```yaml
name: Jira triage poller

on:
  schedule:
    # Poll every 15 minutes — adjust as needed
    - cron: '*/15 * * * *'
  workflow_dispatch:
    inputs:
      jql_override:
        description: 'JQL query override (optional)'
        required: false
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  poll-and-dispatch:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - name: Query Jira for new/updated issues
        id: poll
        env:
          JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
          JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
          JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
          JQL: ${{ inputs.jql_override || secrets.JIRA_JQL }}
        run: |
          set -euo pipefail

          # Query Jira REST API for issues matching the JQL
          RESPONSE=$(curl -sf -u "${JIRA_USER_EMAIL}:${JIRA_TOKEN}" \
            -H "Content-Type: application/json" \
            "${JIRA_BASE_URL}/rest/api/3/search?jql=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${JQL}'))")&fields=key&maxResults=50")

          # Extract issue keys
          ISSUES=$(echo "$RESPONSE" | jq -r '.issues[].key')

          if [ -z "$ISSUES" ]; then
            echo "No issues matched JQL: ${JQL}"
            echo "issues=" >> "$GITHUB_OUTPUT"
          else
            echo "Found issues: $ISSUES"
            # JSON array for matrix
            JSON=$(echo "$ISSUES" | jq -Rsc '[split("\n")[] | select(length > 0)]')
            echo "issues=${JSON}" >> "$GITHUB_OUTPUT"
          fi

      - name: Report poll results
        if: steps.poll.outputs.issues == ''
        run: echo "::notice::No Jira issues to triage this cycle."

    outputs:
      issues: ${{ steps.poll.outputs.issues }}

  triage:
    needs: poll-and-dispatch
    if: needs.poll-and-dispatch.outputs.issues != '' && needs.poll-and-dispatch.outputs.issues != '[]'
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        issue: ${{ fromJSON(needs.poll-and-dispatch.outputs.issues) }}
    steps:
      - uses: actions/checkout@v4

      # The vendored .fullsend/bin/fullsend from Step 3 is picked up
      # automatically by fullsend's action.yml — no build step needed.

      - name: Run triage
        env:
          JIRA_ISSUE_URL: ${{ secrets.JIRA_BASE_URL }}/browse/${{ matrix.issue }}
          JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
          JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
          JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
          JIRA_DUPLICATE_TRANSITION: ${{ secrets.JIRA_DUPLICATE_TRANSITION }}
          JIRA_NOT_PLANNED_TRANSITION: ${{ secrets.JIRA_NOT_PLANNED_TRANSITION }}
          JIRA_SPLIT_TRANSITION: ${{ secrets.JIRA_SPLIT_TRANSITION }}
          FULLSEND_FORGE: jira
          GOOGLE_APPLICATION_CREDENTIALS: ${{ secrets.GOOGLE_APPLICATION_CREDENTIALS }}
          ANTHROPIC_VERTEX_PROJECT_ID: ${{ secrets.ANTHROPIC_VERTEX_PROJECT_ID }}
          GOOGLE_CLOUD_PROJECT: ${{ secrets.GOOGLE_CLOUD_PROJECT }}
          CLOUD_ML_REGION: ${{ secrets.CLOUD_ML_REGION }}
        run: |
          echo "Triaging ${{ matrix.issue }}"
          fullsend run triage \
            --fullsend-dir . \
            --target-repo . \
            --forge jira \
            --output-dir "/tmp/fullsend-${{ matrix.issue }}"

      - name: Show triage result
        if: always()
        run: |
          RESULT=$(find /tmp/fullsend-${{ matrix.issue }} -name agent-result.json 2>/dev/null | head -1)
          if [ -n "$RESULT" ]; then
            echo "::group::Triage result for ${{ matrix.issue }}"
            jq . "$RESULT"
            echo "::endgroup::"
          else
            echo "::warning::No result file found for ${{ matrix.issue }}"
          fi
```

Commit and push:

```bash
git add .github/workflows/jira-triage-poll.yml
git commit -m "add Jira triage poller workflow"
git push
```

---

## Step 5 — Configure secrets for Jira credentials and JQL

In the test repo's GitHub settings, add these repository secrets:

| Secret | Value | Example |
|--------|-------|---------|
| `JIRA_BASE_URL` | Your Jira Cloud site URL | `https://mysite.atlassian.net` |
| `JIRA_USER_EMAIL` | Email of the Jira API user | `bot@example.com` |
| `JIRA_TOKEN` | Jira Cloud API token | `ATATT3x...` |
| `JIRA_JQL` | JQL expression selecting issues to triage | `project = TESTPROJ AND status = Open AND labels not in (triaged, needs-info) AND updated >= -15m ORDER BY updated DESC` |
| `JIRA_DUPLICATE_TRANSITION` | Jira transition name for duplicates | `Duplicate` |
| `JIRA_NOT_PLANNED_TRANSITION` | Transition name for not-planned | `Won't Do` |
| `JIRA_SPLIT_TRANSITION` | Transition name for splits | `Done` |

Also add inference credentials (GCP/Vertex AI):

| Secret | Value |
|--------|-------|
| `GOOGLE_APPLICATION_CREDENTIALS` | Path or content of GCP service account key |
| `ANTHROPIC_VERTEX_PROJECT_ID` | GCP project ID with Vertex AI access |
| `GOOGLE_CLOUD_PROJECT` | Same GCP project ID |
| `CLOUD_ML_REGION` | Region (e.g. `us-east5` or `global`) |

Configure secrets via the CLI:

```bash
gh secret set JIRA_BASE_URL --body "https://mysite.atlassian.net"
gh secret set JIRA_USER_EMAIL --body "bot@example.com"
gh secret set JIRA_TOKEN --body "<your-api-token>"
gh secret set JIRA_JQL --body "project = TESTPROJ AND status = Open AND updated >= -15m"
gh secret set JIRA_DUPLICATE_TRANSITION --body "Duplicate"
gh secret set JIRA_NOT_PLANNED_TRANSITION --body "Won't Do"
gh secret set JIRA_SPLIT_TRANSITION --body "Done"
# ... plus GCP secrets
```

**Tuning the JQL expression:**

The JQL should return only issues that haven't already been triaged and were
recently updated, to avoid re-triaging the entire backlog on every poll.
Adjust the `updated >= -15m` window to match your cron interval. Examples:

- Poll every 15 min: `updated >= -15m`
- Poll hourly: `updated >= -60m`
- First run (backfill): remove the `updated` clause, then add it back

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
