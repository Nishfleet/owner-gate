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
      -f body='{"enforcement":"active"}' \
      -f reason='tighten the main ruleset on repo x'

Inputs: `action` (required, fixed list), `repo` (`^[A-Za-z0-9._-]{1,100}$`),
`id` (digits only), `body` (a single JSON object), `reason` (required).
Unknown action, a missing field, or a field the action does not take: the run
fails. `repo=owner-gate` is always refused: the gate cannot act on itself.

Ids are not resolved for the owner, so `reason` must name the target in plain
English (which ruleset, on which repo, and why).

## Named actions

| action | call | needs |
|---|---|---|
| `ruleset-create` | POST repos/Nishfleet/{repo}/rulesets | repo, body |
| `ruleset-update` | PUT repos/Nishfleet/{repo}/rulesets/{id} | repo, id, body |
| `ruleset-delete` | DELETE repos/Nishfleet/{repo}/rulesets/{id} | repo, id |
| `repo-create` | POST orgs/Nishfleet/repos | body |
| `repo-settings` | PATCH repos/Nishfleet/{repo} | repo, body |
| `repo-archive` | PATCH repos/Nishfleet/{repo}, fixed body `{"archived":true}` | repo |
| `actions-workflow-permissions` | PUT repos/Nishfleet/{repo}/actions/permissions/workflow | repo, body |
| `vulnerability-alerts-enable` | PUT repos/Nishfleet/{repo}/vulnerability-alerts | repo |
| `test-read-org` | GET orgs/Nishfleet (harmless, for end-to-end tests) | nothing |

Body restrictions (loosening protections stays a by-hand owner job):

- `repo-settings`: the body may not contain the keys `visibility`, `private`,
  `security_and_analysis`, `archived` (use `repo-archive`) or `default_branch`.
- `actions-workflow-permissions`: `can_approve_pull_request_reviews` must be
  absent or false.
- `ruleset-create` and `ruleset-update`: no `"enforcement":"disabled"`, and
  `bypass_actors` must be absent or empty.

Org rulesets (`orgs/Nishfleet/rulesets`) have no action: the owner edits them by
hand. There is no `org-ruleset-*` action on purpose, so the gate can never
retarget or loosen an org-wide ruleset that covers itself.

## Approval flow

1. The `announce` job runs with no environment and no secret. It prints, in its
   log and in the run's step summary, the action, method, resolved endpoint, the
   body's sha256 and the full compacted body. The run name also carries the
   action, repo, id, the body, and the `reason (agent-provided)`. The reason is
   written by the agent, not by the system.
2. The `act` job waits for the owner to approve or reject in the GitHub app.
3. On approval, `act` re-validates and resolves the same way, and compares the
   method, endpoint and body sha256 with what `announce` published. Any mismatch
   fails the run before the call.
4. `act` makes the call and prints one line: action, endpoint, HTTP status.
   Response bodies and `gh` error text are never printed. Non-2xx fails the run.

Re-running an approved `act` job repeats the same call with the same inputs.
There is no `concurrency` setting on purpose: a shared group would let a newer
dispatch cancel or replace a run that is waiting for the owner. Each request is
its own run and needs its own approval.

## Trust chain

The workflow is trusted only as it exists on `main`. `workflow_dispatch` runs the
file from the dispatched ref, so the guarantee depends on all of these:

- the environment `owner` keeps "deployment branches: main only", so no branch
  copy of this workflow can obtain the secret;
- the environment's required reviewer is the owner, and admins cannot bypass it;
- `main` is frozen by a ruleset, so changes land only when the owner lifts it.

Removing any of these removes the guarantee.

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
