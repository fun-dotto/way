---
name: setup-gh-repo
description: Copy Rulesets, Environments, and Repository variables from fun-dotto/api-template to another repository
argument-hint: <owner/repo>
license: MIT
metadata:
  author: Kanta Oikawa
  version: "1.0"
---

Copy the Rulesets, Environments (with environment variables), and Actions Repository variables from `fun-dotto/api-template` to the target repository `$ARGUMENTS`.

## Steps

### 1. Validate target repository

Confirm the target repository `$ARGUMENTS` exists by running:

```
gh api repos/$ARGUMENTS
```

If it fails, inform the user and stop.

### 2. Copy Rulesets

Get the ruleset from api-template and create it on the target repo.

Fetch the ruleset:

```
gh api repos/fun-dotto/api-template/rulesets/13371860
```

Then create the same ruleset on the target repo using `gh api -X POST repos/$ARGUMENTS/rulesets` with the following JSON body. Use the exact same structure (name, target, enforcement, conditions, rules, bypass_actors) from the source but omit fields like `id`, `node_id`, `created_at`, `updated_at`, `_links`, `source`, `source_type`, `current_user_can_bypass`.

IMPORTANT: Before creating, check if a ruleset with the same name already exists on the target repo:

```
gh api repos/$ARGUMENTS/rulesets
```

If one exists with the same name, skip creation and inform the user.

### 3. Copy Environments

Create these 4 environments on the target repo: `prod`, `stg`, `dev`, `qa`.

For each environment:

```
gh api -X PUT repos/$ARGUMENTS/environments/<env_name>
```

Then fetch the environment config from api-template to check for protection rules:

```
gh api repos/fun-dotto/api-template/environments/<env_name>
```

If the source environment has `protection_rules` (like required_reviewers on prod), recreate them on the target by updating the environment with the `reviewers` field:

```
gh api -X PUT repos/$ARGUMENTS/environments/<env_name> --input <json_with_reviewers>
```

For the prod environment with required_reviewers, the reviewers format is:

```json
{
  "reviewers": [
    {"type": "User", "id": <user_id>},
    ...
  ]
}
```

### 4. Copy Environment Variables

For each environment (prod, stg, dev, qa), fetch the variables from api-template:

```
gh api repos/fun-dotto/api-template/environments/<env_name>/variables
```

Then create each variable on the target repo:

```
gh api -X POST repos/$ARGUMENTS/environments/<env_name>/variables -f name=<name> -f value=<value>
```

IMPORTANT: Before creating, check if the variable already exists. If it does, update it instead:

```
gh api -X PATCH repos/$ARGUMENTS/environments/<env_name>/variables/<name> -f value=<value>
```

### 5. Copy Repository Variables

Fetch all repository-level variables from api-template:

```
gh api repos/fun-dotto/api-template/actions/variables
```

Then create each variable on the target repo:

```
gh api -X POST repos/$ARGUMENTS/actions/variables -f name=<name> -f value=<value>
```

IMPORTANT: Before creating, check if the variable already exists. If it does, update it instead:

```
gh api -X PATCH repos/$ARGUMENTS/actions/variables/<name> -f value=<value>
```

### 6. Summary

After all steps complete, show a summary table of what was copied:

- Rulesets (created or skipped)
- Environments created (with any protection rules)
- Environment variables per environment
- Repository variables

Run all independent API calls in parallel for efficiency.
