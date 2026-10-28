---
name: leftsize
description: Analyse and act on LeftSize cloud findings from the command line — list the open AWS and Azure findings, read affected resources and savings, ask LeftSize for explanations, fix steps and remediation scripts, and snooze, ignore or report false positives. Use when working with a repository that has LeftSize installed (a .github/workflows/leftsize.yml workflow, or issues labelled `leftsize` created by leftsize[bot]), or when the user mentions LeftSize, cloud waste, cloud cost findings, or idle and orphaned resources.
---

# LeftSize

LeftSize is "Dependabot for cloud waste". A GitHub Action scans an AWS or Azure account with read-only access and LeftSize opens a GitHub issue for each finding: idle and orphaned resources, upcoming service retirements, and security and governance gaps. You ask for more detail, or manage a finding, by commenting `@leftsize <command>` on the issue; LeftSize answers with a comment.

There is no LeftSize API, CLI or MCP server for agents. Everything below is `gh` plus issue comments.

## Read this first — six things that will trip you up

1. **Any `@leftsize` in a comment runs a command.** LeftSize matches the first `@leftsize` anywhere in a comment, in any case, and runs the rest of that line. "cc @leftsize", a quoted earlier command or an email address like `support@leftsize.com` all trigger a reply. When you write a note on a LeftSize issue, write "LeftSize" without the `@`. Only the first command in a comment runs, so post one command per comment.
2. **`snooze`, `ignore` and `wrong` close the issue immediately.** `ignore` and `wrong` are permanent for that rule and scope: LeftSize will never reopen the issue or open a new one for it, also not for resources that appear there later. Closing the issue with `gh issue close` has the same permanent effect. Confirm with the user before any of these unless they have explicitly told you to go ahead.
3. **Commands must come from a person's account.** Comments from bot accounts (`github-actions[bot]`, any GitHub App) are silently ignored. Run these commands with the user's own `gh auth login`, not from a workflow token.
4. **Replies are asynchronous.** Posting a command returns immediately; LeftSize's answer appears as a new comment from `leftsize[bot]`, usually within a few seconds. Wait briefly, then read the comments again. Editing a comment does not run it again — post a new one.
5. **`script` defaults to Azure CLI, also on AWS issues.** Check the `cloud:` label and pass the type explicitly: `@leftsize script aws-cli`.
6. **LeftSize only reads.** It never changes infrastructure, and commands like `stop`, `delete` or `resize` are refused. Any change to cloud resources is made by you or the user, with the user's approval, through their own tools (az, aws, Terraform).

## Step 1: Find the findings

Every LeftSize issue has the `leftsize` label and is opened by `leftsize[bot]`.

```bash
gh issue list --repo {owner}/{repo} --label leftsize --state open --limit 50
```

### Labels LeftSize applies

| Label | Values |
|---|---|
| `leftsize` | on every LeftSize issue |
| category | `cost-optimization`, `deprecation`, `security`, `governance` |
| `cloud:` | `aws`, `azure` (GitHub matches labels case-insensitively) |
| `severity:` | `critical`, `high`, `medium`, `low`, `info` |
| `rule:` | the rule id, e.g. `rule:leftsize-unattached-disk` |
| `resource-type:` | e.g. `resource-type:disk`, `resource-type:lambda` |

Some rules add more, such as `action:upgrade-required` or `cleanup-candidate`. Filter on labels rather than reading every issue:

```bash
# Cost findings on AWS
gh issue list --repo {owner}/{repo} --label leftsize --label cost-optimization --label cloud:aws

# Highest severity first
gh issue list --repo {owner}/{repo} --label leftsize --label severity:critical
gh issue list --repo {owner}/{repo} --label leftsize --label severity:high

# Every finding for one rule
gh issue list --repo {owner}/{repo} --label rule:leftsize-unattached-disk --state all
```

Titles start with a category prefix such as `[COST]`, `[DEPRECATION]`, `[SECURITY]` or `[PRICE CHANGE]`, and cost titles end with the estimated savings, e.g. `[COST] 3 unattached disks (≈ $42/mo)`. To rank cost findings by savings, list titles as JSON and parse the `≈ $…/mo` part:

```bash
gh issue list --repo {owner}/{repo} --label leftsize --label cost-optimization \
  --limit 200 --json number,title,labels
```

Across an organization, use search (it runs against an index, so a brand-new issue may not show up yet):

```bash
gh search issues --owner {org} --label leftsize --state open --limit 100
```

## Step 2: Read one finding

```bash
gh issue view {n} --repo {owner}/{repo} --comments
```

One issue groups every resource that matches one rule in one **scope**: an Azure resource group (`azure:subscription/{id}/resourceGroup/{name}`) or an AWS account and region (`aws:account/{id}/region/{region}`). The body has:

- **Summary** — the rule id, the scope, and for cost rules the estimated monthly savings (list prices; no discounts, reservations or savings plans)
- **Affected Resources** — each resource with its id or ARN and the details the rule used (size, SKU, version, environment)
- **Available Actions** — the `@leftsize` commands that apply to this rule
- **Quick Fix** — example CLI commands

Each scan rewrites the body with the current resources and counts, without posting a comment, so the body is always the latest state. The comments hold the conversation: earlier command replies from `leftsize[bot]` and what the team said.

## Step 3: Ask LeftSize for more

Post one command per comment, then read the comments again for the answer.

```bash
gh issue comment {n} --repo {owner}/{repo} --body '@leftsize howto'
# a few seconds later
gh issue view {n} --repo {owner}/{repo} --comments
```

| Command | Answer |
|---|---|
| `@leftsize explain` | Why the resources were flagged and what the finding means |
| `@leftsize howto` | Step-by-step fix (alias `how-to`) |
| `@leftsize impact` | Cost, performance or security impact |
| `@leftsize alternatives` | Other ways to address it (alias `options`) |
| `@leftsize script <type>` | Remediation script: `aws-cli`, `azure-cli` (default), `powershell`, `terraform` |
| `@leftsize prompt` | A ready-made prompt with the finding's context |
| `@leftsize` | The list of commands |

These change nothing; they only add a comment. Before asking, check whether the answer is already in the comments — someone may have asked before.

## Step 4: Act on the finding

| Command | Effect |
|---|---|
| `@leftsize snooze <duration>` | Closes the issue as completed; the first scan after the period that still finds the problem reopens it. Duration: `14d` (default), `2w`, `3m`, `1y` |
| `@leftsize ignore <reason>` | Closes as not planned, **permanently** for this rule and scope |
| `@leftsize wrong <reason>` | Reports a false positive; closes as not planned, **permanently** (aliases `false-positive`, `fp`) |
| `@leftsize feedback <message>` | Records feedback; the issue stays open |

Always give a reason with `ignore` and `wrong` — it is the record of why the team dismissed the finding:

```bash
gh issue comment {n} --repo {owner}/{repo} \
  --body '@leftsize ignore Shared dev environment, cost accepted by the platform team'
```

**After fixing the resources**, close the issue yourself. LeftSize does not close issues when resources disappear, and it does not reopen an issue you closed:

```bash
gh issue close {n} --repo {owner}/{repo} --reason completed \
  --comment 'Deleted the 3 unattached disks after taking snapshots.'
```

Ordinary GitHub actions work as usual: assign (`gh issue edit --add-assignee`), add your own labels (scans keep them), link a pull request with `Fixes {owner}/{repo}#{n}` (or `Fixes #{n}` in the same repository) so merging it closes the issue.

## A good workflow

1. List the open findings and group them by category, cloud and severity; for cost, total the savings.
2. Read the issues that matter most, including comments.
3. Ask `@leftsize howto` or `@leftsize script aws-cli|azure-cli|terraform` where you need the fix.
4. Propose a plan to the user: what to fix, what to snooze, what to ignore and why.
5. Make infrastructure changes only with the user's approval — ideally as a pull request to their infrastructure code that references the issue.
6. Close fixed issues; snooze or ignore the rest with a reason.

## Running a scan

Scans run daily from `.github/workflows/leftsize.yml`. To scan now:

```bash
gh workflow run leftsize.yml --repo {owner}/{repo}
gh run list --repo {owner}/{repo} --workflow leftsize.yml --limit 1
```

The run's job summary lists findings per rule and the estimated savings, including findings that don't get issues on the Free plan (only cost rules open issues there). To skip rules permanently for a repository, set the action's `exclude-policies` input in that workflow; there is no other config file.

## Documentation

Read the docs rather than guessing; every page is available as Markdown:

```bash
curl https://support.leftsize.com/llms.txt          # index of every docs page
curl https://support.leftsize.com/commands.md       # full command reference
curl https://support.leftsize.com/how-it-works.md   # scan cycle and issue lifecycle
curl https://leftsize.com/llms.txt                  # product overview and every rule
```

Each rule has a page with what it checks and how to fix it: <https://leftsize.com/rules/>

## Further reading

- Coding agents guide: <https://support.leftsize.com/agents.md>
- Commands: <https://support.leftsize.com/commands.md>
- Troubleshooting: <https://support.leftsize.com/troubleshooting.md>
