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

The public mint ships the `main` branch of `fullsend-ai/agents`. The Jira
triage support lives on the `jira` branch, so you need to override the
agents ref.

Create `.fullsend/config.yaml` in your test repo to pin the agents source
to the `jira` branch:

```yaml
# .fullsend/config.yaml
agents:
  repo: fullsend-ai/agents
  ref: jira          # <-- use the branch with Jira support
```

> **How this works:** When fullsend dispatches an agent, it resolves the
> harness from the agents repo at the specified ref. By pointing `ref:` at
> `jira`, the dispatch picks up `harness/triage.yaml` with its `forge.jira`
> block, the Jira skills, scripts, and policies from that branch.

Commit and push:

```bash
mkdir -p .fullsend
# write the config.yaml above
git add .fullsend/config.yaml
git commit -m "pin agents to jira branch for testing"
git push
```

---

## Step 3 — Vendor a custom-built fullsend with Jira support

If the `fullsend` CLI on the public release channel doesn't yet include the
`fullsend issues post-comment --tracker jira` subcommand (needed by the
post-script), you need to build and vendor a custom fullsend binary.

```bash
# Clone fullsend and build from main (or a branch with Jira CLI support)
git clone https://github.com/fullsend-ai/fullsend.git /tmp/fullsend-build
cd /tmp/fullsend-build
make go-build

# Verify the Jira tracker flag exists
./bin/fullsend issues post-comment --help | grep -q 'tracker'
echo "Jira tracker support confirmed in custom build"
```

Install the custom binary so it takes precedence:

```bash
# Option A: replace your PATH fullsend
sudo cp /tmp/fullsend-build/bin/fullsend /usr/local/bin/fullsend

# Option B: use a local override for CI (if running in Actions)
# Copy into the test repo and reference via a wrapper workflow step.
```

For the GitHub Actions workflow (Step 4), the poller workflow will need
this custom binary too. Either:

- **Self-hosted runner:** pre-install the custom `fullsend` binary, or
- **GitHub-hosted runner:** add a build step in the workflow that clones and
  builds fullsend before calling `fullsend run` (see the pattern in
  `.github/workflows/functional-tests.yml` in the agents repo, lines 259-280).

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

      # -- Build or install fullsend (see Step 3) --
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Build fullsend from source
        run: |
          git clone --depth 1 https://github.com/fullsend-ai/fullsend.git /tmp/fullsend-src
          make -C /tmp/fullsend-src go-build
          echo "/tmp/fullsend-src/bin" >> "$GITHUB_PATH"

      - name: Install OpenShell
        run: |
          eval "$(grep -E '^OPENSHELL_(VERSION|SHA)=' /tmp/fullsend-src/.github/scripts/openshell-version.sh)"
          /tmp/fullsend-src/.github/scripts/install-openshell.sh

      - name: Install Podman
        run: |
          sudo apt-get update && sudo apt-get install -y podman
          whoami_user="$(whoami)"
          grep -q "^${whoami_user}:" /etc/subuid || sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 "${whoami_user}"
          podman system migrate

      - name: Start Podman API
        run: |
          SOCKET_PATH="${XDG_RUNTIME_DIR:-/run/user/$(id -u)}/podman/podman.sock"
          mkdir -p "$(dirname "${SOCKET_PATH}")"
          podman system service --time=0 "unix://${SOCKET_PATH}" &
          for i in $(seq 1 30); do
            [ -S "${SOCKET_PATH}" ] && podman --url "unix://${SOCKET_PATH}" info >/dev/null 2>&1 && break
            sleep 1
          done

      # -- Dispatch triage --
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
