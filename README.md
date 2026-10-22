# owner-gate

Approval gate for owner-level GitHub API calls. The owner's full-power token
exists only as the environment secret `OWNER_TOKEN` in the environment `owner`,
which requires the owner as reviewer. An agent starts a run naming one fixed
action; GitHub pauses the `act` job until the owner approves it in the GitHub
mobile app. Only then does the job get the secret and make that single call.

There is no free-form method or endpoint. The org is hard-coded as `Nishfleet`.
Each named action maps to exactly one method and endpoint template in the
workflow, and the endpoint is never built from raw input.

## Use (agents)

    gh workflow run owner-action.yml -R Nishfleet/owner-gate \
      -f action=ruleset-update -f repo=x -f id=123 \
      -f body='{"enforcement":"active"}' -f reason='short plain-English why'

Inputs: `action` (required, fixed list), `repo` (`^[A-Za-z0-9._-]{1,100}$`),
`id` (digits only), `body` (JSON), `reason` (required). Unknown action, a
missing field, or a field the action does not take: the run fails.

## Named actions

| action | call | needs |
|---|---|---|
| `ruleset-create` | POST repos/Nishfleet/{repo}/rulesets | repo, body |
| `ruleset-update` | PUT repos/Nishfleet/{repo}/rulesets/{id} | repo, id, body |
| `ruleset-delete` | DELETE repos/Nishfleet/{repo}/rulesets/{id} | repo, id |
| `org-ruleset-create` | POST orgs/Nishfleet/rulesets | body |
| `org-ruleset-update` | PUT orgs/Nishfleet/rulesets/{id} | id, body |
| `repo-create` | POST orgs/Nishfleet/repos | body |
| `repo-settings` | PATCH repos/Nishfleet/{repo} | repo, body |
| `repo-archive` | PATCH repos/Nishfleet/{repo}, fixed body `{"archived":true}` | repo |
| `actions-workflow-permissions` | PUT repos/Nishfleet/{repo}/actions/permissions/workflow | repo, body |
| `vulnerability-alerts-enable` | PUT repos/Nishfleet/{repo}/vulnerability-alerts | repo |
| `test-read-org` | GET orgs/Nishfleet (harmless, for end-to-end tests) | nothing |

## Approval flow

1. The `announce` job runs with no environment and no secret. Its first log
   line shows the action, method, resolved endpoint, and the body's sha256 and
   first 200 characters (compacted JSON), so the owner can read exactly what is
   being approved. The run name (`owner ACTION repo id: reason`) is in the
   approval notice.
2. The `act` job waits for the owner to approve or reject in the GitHub app.
3. On approval, `act` re-validates, resolves the same way, and makes the call.
   It prints one line: action, endpoint, HTTP status. Response bodies are never
   printed. Non-2xx fails the run.

## Add an action

Open a PR that adds the action to the `options` list and to the `case` block in
BOTH jobs (`announce` and `act`) of `.github/workflows/owner-action.yml`. The
owner lifts the main ruleset to merge it.

## Rules

This repo is public. Inputs, run names and logs are public. Never put secrets
in them.

## Off-switch

- Delete the environment secret: `gh secret delete OWNER_TOKEN --env owner -R Nishfleet/owner-gate`
- Or disable the workflow: `gh workflow disable owner-action.yml -R Nishfleet/owner-gate`

Owner: nish3451
