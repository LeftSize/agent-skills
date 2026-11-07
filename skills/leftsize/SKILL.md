---
name: leftsize
description: Analyse and act on LeftSize cloud findings in GitHub Issues or Jira — list the open AWS, Azure and Cloudflare findings, read affected resources and savings, ask LeftSize for explanations, fix steps and remediation scripts, and snooze, ignore or report false positives. Use when working with a repository that has LeftSize installed (a .github/workflows/leftsize.yml workflow, issues labelled `leftsize` created by leftsize[bot], or Jira work items labelled `leftsize`), or when the user mentions LeftSize, cloud waste, cloud cost findings, or idle and orphaned resources.
---

# LeftSize

LeftSize is "Dependabot for cloud waste". A GitHub Action scans an AWS, Azure or Cloudflare account with read-only access and LeftSize opens an issue for each finding: idle and orphaned resources, upcoming service retirements, and security and governance gaps. The issues go to **GitHub Issues** (the default) or to **Jira Cloud**, chosen per repository. You ask for more detail, or manage a finding, by commenting a command on the issue (`@leftsize <command>` on GitHub, `/leftsize <command>` in Jira); LeftSize answers with a comment.

There is no LeftSize API, CLI or MCP server for agents. Everything below is the issue tracker's own tooling plus comments: `gh` (or the GitHub MCP server) for GitHub, and the Atlassian MCP server (Rovo) or `acli` for Jira. The steps show GitHub; [Findings in Jira](#findings-in-jira) maps each one to Jira.

## Step 0: GitHub or Jira?

Work out where this repository's findings are before anything else, and tell the user what you found:

1. **The user says so**, or names a Jira project or a key like `OPS-123`: use that.
2. **GitHub**: `gh issue list --repo {owner}/{repo} --label leftsize --state open --limit 5`. Open issues from `leftsize[bot]` show that findings are on GitHub, not that new ones still go there: when a repository is switched to Jira without moving its open issues, those stay open on GitHub while new findings go to Jira. If the newest LeftSize issue is much older than the last `leftsize.yml` run, ask the user whether new findings go to Jira.
3. **Moves**: when LeftSize moves an issue it closes the old one with a comment such as *LeftSize moved this issue to Jira: [OPS-123](https://acme.atlassian.net/browse/OPS-123)*; the key gives the project and the link the Jira site. Find those with `gh search issues --repo {owner}/{repo} --label leftsize "moved this issue to Jira"`. Go by the most recent move: a repository moved back to GitHub has Jira issues closed with *moved this issue to GitHub Issues*.
4. **Jira**: with the Atlassian MCP server or `acli` you can check that the site has LeftSize issues (JQL `labels = leftsize ORDER BY created DESC`). That doesn't tell you which repository they belong to: several repositories can send findings to one project, and the issues don't name their repository. Confirm the project with the user (or take it from a move comment), then scope every search to it with `project = KEY`. If other repositories also send findings to that project, say so when you report totals, and check with the user before acting on an issue that may belong to another repository.
5. **Still unclear** (no open GitHub issues and no Jira access, or findings in both places): ask the user where this repository's LeftSize issues go. Don't guess.

## Read this first — six things that will trip you up

1. **Any `@leftsize` mention in a comment runs a command.** In Jira, `/leftsize` and `leftsize:` do too. LeftSize matches the first `@leftsize` that stands as a word of its own, anywhere in a comment, in any case, and runs the rest of that line only. "cc @leftsize" or a quoted earlier command trigger a reply; an email address like `support@leftsize.com` doesn't. When you write a note on a LeftSize issue, write "LeftSize" without the `@`, the `/` or a colon after it. Only the first command in a comment runs, so post one command per comment.
2. **`snooze`, `ignore` and `wrong` close the issue immediately.** `ignore` and `wrong` stick for that rule and scope: LeftSize won't reopen the issue or open a new one for it, also not for resources that appear there later. Closing the issue yourself (`gh issue close`, or moving a Jira issue to Done) has the same effect. Only reopening the issue (`gh issue reopen`, or moving the Jira issue out of Done) undoes it; LeftSize then tracks it again from the next scan. Confirm with the user before any of these unless they have explicitly told you to go ahead.
3. **On GitHub, commands must come from a person's account.** Comments from bot accounts (`github-actions[bot]`, any GitHub App) are silently ignored. Run these commands with the user's own `gh auth login`, not from a workflow token. In Jira, only LeftSize's own comments are skipped: a comment from any other account runs, including Rovo agents and automation rules.
4. **Replies are asynchronous.** Posting a command returns immediately; LeftSize's answer appears as a new comment from `leftsize[bot]` (in Jira: the **LeftSize** app user), usually within a few seconds. Wait briefly, then read the comments again. Editing a comment does not run it again — post a new one.
5. **`script` without a type gives the CLI of the issue's cloud** (`aws-cli` on AWS, `azure-cli` on Azure). Cloudflare rules have one script each (Cloudflare API calls with `curl`, plus Terraform), whatever type you ask for. Not every rule has every type: the reply says which types it has and shows the default one.
6. **LeftSize only reads.** It never changes infrastructure, and commands like `stop`, `delete` or `resize` are refused. Any change to cloud resources is made by you or the user, with the user's approval, through their own tools (az, aws, the Cloudflare dashboard or API, Terraform).

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
| `cloud:` | `aws`, `azure`, `cloudflare` (GitHub matches labels case-insensitively) |
| `severity:` | `critical`, `high`, `medium`, `low` |
| `rule:` | the rule id, e.g. `rule:leftsize-unattached-disk`; labels max 50 chars, so long ids drop `leftsize-` (`rule:cloudflare-worker-outdated-compatibility-date`) |
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

Titles start with a category prefix such as `[COST]`, `[DEPRECATION]`, `[SECURITY]` or `[PRICE CHANGE]`, and most cost titles end with the estimated savings, e.g. `[COST] 3 unattached disks (≈ $42/mo)` (monthly list prices in USD). Some cost rules show the amount only in the body and some have no price, so don't treat a title without an amount as zero savings. To rank cost findings by savings, list titles as JSON and parse the `≈ $…/mo` part:

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

One issue groups every resource that matches one rule in one **scope**: an Azure resource group (`azure:subscription/{id}/resourceGroup/{name}`), an AWS account and region (`aws:account/{id}/region/{region}`), or a Cloudflare account (`cloudflare:account/{id}`; resources are `cloudflare:account/{id}/{kind}/{name}`, e.g. `.../zone/example.com`). The body has:

- **Summary** — the rule id, the scope, and for cost rules the estimated monthly savings (list prices; no discounts, reservations or savings plans)
- **Affected Resources** — each resource with its id or ARN and the details the rule used (size, SKU, version, environment)
- **Available Actions** — the `@leftsize` commands that apply to this rule
- **Quick Fix** — example CLI commands

While the issue is open, each scan that still finds the problem rewrites the body with the current resources and counts, without posting a comment. When the problem is gone, the body keeps the last list: LeftSize doesn't close issues or mark them resolved by itself. The comments hold the conversation: earlier command replies from `leftsize[bot]` and what the team said.

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
| `@leftsize script [type]` | Remediation script: `aws-cli`, `azure-cli`, `powershell`, `terraform` (default: the CLI of the issue's cloud) |
| `@leftsize prompt` | A ready-made prompt with the finding's context |
| `@leftsize help` | The list of commands (also `@leftsize` on its own) |

These change nothing; they only add a comment. Before asking, check whether the answer is already in the comments — someone may have asked before.

## Step 4: Act on the finding

| Command | Effect |
|---|---|
| `@leftsize snooze <duration>` | Closes the issue as completed; the first scan after the period that still finds the problem reopens it. Duration: `14d` (default), `2w`, `3m`, `1y` |
| `@leftsize ignore <reason>` | Closes as not planned, for good for this rule and scope (until a person reopens the issue) |
| `@leftsize wrong <reason>` | Reports a false positive; closes as not planned, for good (aliases `false-positive`, `fp`) |
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

## Findings in Jira

Same issues, same commands, different tooling. One issue per rule and scope, created as the **LeftSize** app user; summary and description are the same as on GitHub (long descriptions are cut to fit Jira and say so).

| Step | GitHub | Jira |
|---|---|---|
| Tooling | `gh`, GitHub MCP server | Atlassian MCP server (Rovo): `searchJiraIssuesUsingJql`, `getJiraIssue`, `addCommentToJiraIssue`, `getTransitionsForJiraIssue`, `transitionJiraIssue`; or `acli jira workitem …` |
| List | `gh issue list --label leftsize` | JQL `project = KEY AND labels = leftsize AND statusCategory != Done` |
| Filter | `--label cloud:aws --label severity:high` | Labels use `-` instead of `:`: `labels = cloud-aws AND labels = severity-high`, `labels = rule-leftsize-unattached-disk`, `labels = cost-optimization` |
| Read | `gh issue view {n} --comments` | Get the issue with its comments (MCP `getJiraIssue`: `fields` including `comment`; `acli jira workitem view {KEY} --fields "summary,description,status,comment"`). MCP `properties: ["leftsize"]` also returns the rule, cloud, severity, resource count and monthly savings as data |
| Command | comment `@leftsize howto` | comment `/leftsize howto` |
| Fixed | `gh issue close --reason completed` | Transition to a Done status with a resolution like **Done** or **Fixed** — counts the savings as realised |
| Dismiss | `@leftsize ignore <reason>` | `/leftsize ignore <reason>` — use the command, not a transition: it records the dismissal without savings, and sets a "Won't Do"-like resolution when the workflow asks for one |
| Reopen | `gh issue reopen` | Transition out of Done |
| Link a fix | `Fixes {owner}/{repo}#{n}` in the PR | With the GitHub for Jira app, the key (`OPS-123`) in the branch, commit or PR title links them; close the issue yourself after the merge |

```bash
# With acli (check `acli jira workitem --help` for your version's flags)
acli jira workitem search --jql 'project = OPS AND labels = leftsize AND labels = cost-optimization AND statusCategory != Done' \
  --fields "key,summary,status,labels" --json
acli jira workitem comment create --key OPS-123 --body '/leftsize script terraform'
acli jira workitem comment list --key OPS-123
```

With the Atlassian MCP server, call `getAccessibleAtlassianResources` first for the `cloudId`, then the tools above. A comment you add through it is posted as the user, so LeftSize answers it.

Jira-specific things to know:

- **Which repository?** Neither the description nor the issue property names the GitHub repository the findings come from; the **LeftSize** panel in Jira's issue view shows it to people, but agents can't read the panel. Ask the user when you need it, for example to run a scan, which still happens in that repository's GitHub Actions (see [Running a scan](#running-a-scan)).
- **Workflows differ per project.** Get the available transitions before you transition, and pick a status in the Done category. In projects where Jira sets the resolution itself (often team-managed), every close counts as fixed, so dismiss with `/leftsize ignore` or `/leftsize wrong` instead of closing.
- **Snooze** moves the issue to Done; when it expires and the problem is still there, LeftSize moves it back to an open status (To Do, or In Progress if there's none) and comments.
- **Moving an issue to another project** with Jira's Move is fine: LeftSize follows the new key.
- **Scans rewrite the summary and description.** Put notes in comments; assignee, priority, labels and other fields are yours.

## Documentation

Read the docs rather than guessing; every page is available as Markdown:

```bash
curl https://support.leftsize.com/llms.txt          # index of every docs page
curl https://support.leftsize.com/commands.md       # full command reference
curl https://support.leftsize.com/how-it-works.md   # scan cycle and issue lifecycle
curl https://support.leftsize.com/jira.md           # LeftSize in Jira: labels, commands, closing
curl https://leftsize.com/llms.txt                  # product overview and every rule
```

Each rule has a page with what it checks and how to fix it: <https://leftsize.com/rules/>

## Further reading

- Coding agents guide: <https://support.leftsize.com/agents.md>
- Commands: <https://support.leftsize.com/commands.md>
- Jira: <https://support.leftsize.com/jira.md>
- Troubleshooting: <https://support.leftsize.com/troubleshooting.md>
