# Jira Triage Functional Test Plan

End-to-end validation that a fresh repo, enrolled in fullsend with Jira
support (merged to `fullsend-ai/agents` main via PR #827), can poll a Jira
Cloud project and dispatch triage against a real issue.

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

## Step 2 — Configure the triage harness with Jira support

Jira triage support shipped in `fullsend-ai/agents` main via
[PR #827](https://github.com/fullsend-ai/agents/pull/827). The default
enrollment already points at `main`, so no branch override is needed. You
do need to configure Jira workflow transition names and add a CEL `trigger:`
expression — both via a local harness override using `base:` composition.

### Get the current harness SHA

```bash
HARNESS_SHA=$(curl -sfL \
  "https://raw.githubusercontent.com/fullsend-ai/agents/main/harness/triage.yaml" \
  | sha256sum | awk '{print $1}')
echo "Harness SHA: ${HARNESS_SHA}"
```

### Create the local harness override

Create `.fullsend/harness/triage.yaml` with three things:

1. `base:` — inherits the full upstream harness from `main`
2. `trigger:` — CEL expression that the poll binary evaluates to decide
   whether to dispatch triage for a given Jira event. Without this field
   the harness is not considered a dispatch candidate and the poller
   produces no output.
3. `forge.jira.env.runner` — Jira workflow transition name overrides (if
   your project's names differ from the upstream defaults)

```yaml
# .fullsend/harness/triage.yaml
base: https://raw.githubusercontent.com/fullsend-ai/agents/main/harness/triage.yaml#sha256=<HARNESS_SHA>

trigger: >
  event.source.system == "jira" && event.entity.kind == "work_item"

forge:
  jira:
    env:
      runner:
        JIRA_DUPLICATE_TRANSITION: "Duplicate"
        JIRA_NOT_PLANNED_TRANSITION: "Won't Do"
        JIRA_SPLIT_TRANSITION: "Done"
```

Replace `<HARNESS_SHA>` with the value from above. Adjust the transition
names to match your Jira project's workflow.

The `trigger:` expression is evaluated against a `normevent.Event` map
(ADR 0061). The available fields are:

| Field | Example |
|-------|---------|
| `event.source.system` | `"jira"` |
| `event.entity.kind` | `"work_item"` |
| `event.transition.kind` | `"opened"`, `"comment_added"`, ... |
| `event.actor.role` | `"write"`, `"external"`, ... |
| `event.state.labels` | `["bug", "needs-info"]` |

Then update `.fullsend/config.yaml` to point at the local harness:

```yaml
# .fullsend/config.yaml
version: "1"
agents:
  # Local harness that inherits from main via base: composition
  - name: triage
    source: harness/triage.yaml
allowed_remote_resources:
  - https://raw.githubusercontent.com/fullsend-ai/agents/
```

> **How this works:** `source: harness/triage.yaml` resolves relative to
> the `.fullsend/` directory. That file's `base:` pulls in the full upstream
> harness — `forge.jira` block, Jira skills, scripts, and policies — then
> the local `trigger:` and `forge.jira.env.runner` values merge on top
> (child wins for env map keys).

Commit and push:

```bash
git add .fullsend/config.yaml .fullsend/harness/triage.yaml
git commit -m "configure triage harness for Jira with CEL trigger"
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

## Step 4 — Install a GitHub Actions workflow to poll Jira and run triage

`fullsend poll --input-driver jira-poll` (with the binary from Step 3, which
carries PR #6340) queries Jira, evaluates the repo's registered harness CEL
`trigger:` expressions, and writes `dispatches.json` — an array of execution
refs with the same fields the `harness-run` job in `reusable-dispatch.yml`
expects from its matrix (`agent`, `role`, `source_repo`, `event_payload`,
`status_repo`, `status_number`).

The poll workflow has two jobs:

1. **`poll`** — runs `fullsend poll`, converts `dispatches.json` to a GHA
   matrix, and outputs it.
2. **`harness-run`** — consumes the matrix via
   `fullsend-ai/fullsend/.github/workflows/reusable-harness-run.yml`, passing
   the JIRA secrets so they're available to pre-scripts and the agent
   environment.

No per-stage `triage.yaml` workflow is needed. The repo's `fullsend.yaml`
shim (which routes GitHub events via `workflow_call`) is left untouched; the
Jira poll path is entirely self-contained in `fullsend-poll-jira.yaml`.

`FULLSEND_REF` at the top of the workflow pins the `fullsend-ai/fullsend`
commit to the branch that carries PR #6340. Update it when that PR merges to
`main`.

The workflow is at `.github/workflows/fullsend-poll-jira.yaml` in this repo.
Replace `KONFLUX` and the `--jql` filter with your project key and query.

The workflow passes JIRA secrets to `reusable-harness-run.yml`:

```yaml
secrets:
  FULLSEND_GCP_WIF_PROVIDER: ${{ secrets.FULLSEND_GCP_WIF_PROVIDER }}
  FULLSEND_GCP_PROJECT_ID: ${{ secrets.FULLSEND_GCP_PROJECT_ID }}
  OTEL_EXPORTER_OTLP_TRACES_HEADERS: ${{ secrets.OTEL_EXPORTER_OTLP_TRACES_HEADERS }}
  OTEL_EXPORTER_OTLP_HEADERS: ${{ secrets.OTEL_EXPORTER_OTLP_HEADERS }}
  JIRA_TOKEN: ${{ secrets.JIRA_TOKEN }}
  JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
  JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
```

These are exposed as environment variables to pre-scripts and the agent runtime.

Commit and push:

```bash
git add .github/workflows/fullsend-poll-jira.yaml
git commit -m "poll Jira and run triage via harness-run matrix job"
git push
```

---

## Step 5 — Configure secrets and Jira credentials

The poller workflow uses GitHub Actions **secrets** for Jira credentials.
The harness (`forge.jira` in `harness/triage.yaml`) picks up these
credentials from the runner environment, so the same secrets serve both
the poller and the agent.

### Secrets (Settings > Secrets and variables > Actions > Secrets)

| Secret | Value | Example |
|--------|-------|---------|
| `JIRA_TOKEN` | Jira Cloud API token | `ATATT3x...` |
| `JIRA_USER_EMAIL` | Email of the Jira API user | `bot@example.com` |
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
gh secret set JIRA_BASE_URL --body "https://mysite.atlassian.net"
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
