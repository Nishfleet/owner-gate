# owner-gate

Approval gate for owner-level GitHub API calls. The owner's full-power token
exists only as the environment secret `OWNER_TOKEN` in the environment `owner`,
which requires the owner as reviewer. An agent starts a run naming one API call;
GitHub pauses the job until the owner approves it in the GitHub mobile app. Only
then does the job get the secret and make that single call.

## Use (agents)

    gh workflow run owner-action.yml -R Nishfleet/owner-gate \
      -f method=PUT -f endpoint=repos/Nishfleet/x/rulesets \
      -f body='{"key":"value"}' -f reason='short plain-English why'

`method` is GET/POST/PUT/PATCH/DELETE. `body` is optional JSON. The log shows
only one line: method, endpoint and HTTP status. Response bodies are never
printed (the repo is public), so this is for write actions, not for reading data.

## Approval

The owner approves or rejects the pending run in the GitHub app. The run name
(`owner METHOD endpoint: reason`) is what they see.

## Rules

This repo is public. Inputs, run names and logs are public. Never put secrets
in inputs.

## Off-switch

- Delete the environment secret: `gh secret delete OWNER_TOKEN --env owner -R Nishfleet/owner-gate`
- Or disable the workflow: `gh workflow disable owner-action.yml -R Nishfleet/owner-gate`

Owner: nish3451
