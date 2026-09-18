# **YES-DIRECT-MERGE** — A cloud run can merge its own PR via `mcp__github__merge_pull_request`; auto-merge is blocked because the repository does not have Allow Auto-Merge enabled in Settings.

---

## Step 1 — Identity and Access

**Call:** `mcp__github__get_me`

**Output:**
```json
{"login":"thomas-tahk","id":10647431,"profile_url":"https://github.com/thomas-tahk","avatar_url":"https://avatars.githubusercontent.com/u/10647431?v=4","details":{"name":"TTahk","public_repos":62,"public_gists":0,"followers":1,"following":2,"created_at":"2015-01-22T04:07:03Z","updated_at":"2026-09-09T13:36:52Z"}}
```

**Call:** `mcp__github__list_branches` (owner=thomas-tahk, repo=chief-probe)

**Output:**
```json
[{"name":"main","sha":"cad9729690f3442181fcb2a7a39d20ab1552bfc2","protected":false}]
```

Identity confirmed as `thomas-tahk`. Only `main` branch existed at session start.

---

## Step 2 — AUTO-MERGE PATH: Branch, Commit, Push, PR

**git push command and output:**
```
git checkout -b claude/probe-auto-merge
echo "1" > counter.txt
git add counter.txt && git commit -m "Probe A: set counter to 1"
git push -u origin claude/probe-auto-merge

Output:
remote: Create a pull request for 'claude/probe-auto-merge' on GitHub by visiting:
remote:      https://github.com/thomas-tahk/chief-probe/pull/new/claude/probe-auto-merge
To https://github.com/thomas-tahk/chief-probe
 * [new branch]      claude/probe-auto-merge -> claude/probe-auto-merge
branch 'claude/probe-auto-merge' set up to track 'origin/claude/probe-auto-merge'.
```

**Push succeeded.** `claude/`-prefixed branch pushed without complaint.

**Call:** `mcp__github__create_pull_request` (title="Probe A: auto-merge", head=claude/probe-auto-merge, base=main, body="throwaway")

**Output:**
```json
{"id":"4568776652","url":"https://github.com/thomas-tahk/chief-probe/pull/1"}
```

**PR #1 created:** https://github.com/thomas-tahk/chief-probe/pull/1

---

## Step 3 — AUTO-MERGE: Enable Attempt

### Tool availability

The tool `mcp__github__enable_pr_auto_merge` EXISTS in this session. Its description states it uses GraphQL (`enablePullRequestAutoMerge`) under the hood. A matching `mcp__github__disable_pr_auto_merge` also exists.

There is **no** `mcp__github__update_repository` tool (or any equivalent that takes `allow_auto_merge=true`). ToolSearch for "update repo allow_auto_merge" returned no repo-settings tool.

### Enable attempt on PR #1

**Call:** `mcp__github__enable_pr_auto_merge` (owner=thomas-tahk, repo=chief-probe, pullNumber=1, mergeMethod=SQUASH)

**Output (error):**
```
Auto-merge is not enabled for this repository. Enable it in repository Settings → General → Pull Requests → Allow auto-merge.
```

**Conclusion:** The tool can reach GraphQL (it attempted the `enablePullRequestAutoMerge` mutation), but the call was rejected by GitHub because the repository's `Allow auto-merge` setting is OFF. A cloud run cannot flip that setting — there is no MCP tool that updates repository settings. Auto-merge requires a human to enable it in the repo's Settings UI first.

---

## Step 4 — DIRECT MERGE PATH: Branch, Commit, Push, PR, Merge

**git push output:**
```
git checkout -b claude/probe-direct-merge
echo "2" > counter.txt
git add counter.txt && git commit -m "Probe B: set counter to 2"
git push -u origin claude/probe-direct-merge

Output:
remote: Create a pull request for 'claude/probe-direct-merge' on GitHub by visiting:
remote:      https://github.com/thomas-tahk/chief-probe/pull/new/claude/probe-direct-merge
To https://github.com/thomas-tahk/chief-probe
 * [new branch]      claude/probe-direct-merge -> claude/probe-direct-merge
branch 'claude/probe-direct-merge' set up to track 'origin/claude/probe-direct-merge'.
```

**Call:** `mcp__github__create_pull_request` (title="Probe B: direct merge", head=claude/probe-direct-merge, base=main, body="throwaway")

**Output:**
```json
{"id":"4568782836","url":"https://github.com/thomas-tahk/chief-probe/pull/2"}
```

**PR #2 created:** https://github.com/thomas-tahk/chief-probe/pull/2

**Call:** `mcp__github__merge_pull_request` (owner=thomas-tahk, repo=chief-probe, pullNumber=2, merge_method=squash)

**Output:**
```json
{"sha":"e9f3a8ed74d783eee5b85cf90056651331f36f32","merged":true,"message":"Pull Request successfully merged"}
```

**DIRECT MERGE SUCCEEDED.**

---

## Step 5 — Verify Merge Landed

**Call:** `mcp__github__pull_request_read` (method=get, pullNumber=2)

**Output (relevant fields):**
```json
{
  "number": 2,
  "title": "Probe B: direct merge",
  "state": "closed",
  "merged": true,
  "merged_by": "thomas-tahk",
  "merged_at": "2026-09-18T12:36:39Z",
  "head": {"ref": "claude/probe-direct-merge", "sha": "df3e977897ab313f77a81116e6a91311059794e3"},
  "base": {"ref": "main"}
}
```

**git log output:**
```
git fetch origin main && git log origin/main --oneline -3

e9f3a8e Probe B: set counter to 2 (#2)
cad9729 Seed scratch repo for chief Step 0 experiments
```

The squash-merged commit `e9f3a8e` is on `origin/main`. Merge confirmed.

---

## Step 6 — Branch-Name Check

Both `claude/probe-auto-merge` and `claude/probe-direct-merge` pushed without any complaint or rejection. The `claude/`-prefixed branch namespace is not restricted.

---

## Step 7 — PR Comments via `mcp__github__add_issue_comment`

YES. `mcp__github__add_issue_comment` accepts a PR number as `issue_number`. It posts a top-level comment on the PR thread. This is confirmed available in this session and is a viable fallback channel for posting status/summaries on PRs.

---

## Step 8 — Rate Limits, Quotas, Usage Caps

**None observed.** No rate-limit, quota, concurrency, or permission-policy messages were returned by any tool call in this run.

---

## What This Means

**A cloud agent can open and merge its own PRs via direct merge (`mcp__github__merge_pull_request`).** In a system where cloud agents open PRs, they can also close the loop — merging without any human approval step — provided:

1. The GitHub App is installed on the repo with write access.
2. There are no branch protection rules requiring reviews or status checks.
3. The token used by the MCP server has repo write scope.

**Auto-merge (`mcp__github__enable_pr_auto_merge`) works at the GraphQL layer but requires the repo owner to first enable "Allow auto-merge" in repository Settings.** The tool exists and reaches GitHub's GraphQL API, but the repo-level gate blocks it. No MCP tool can flip that setting programmatically — it requires a manual UI action by a repo admin.

For fully autonomous CI-gated merge flows, the pattern would be: human enables "Allow auto-merge" once in Settings → agent opens PR → agent calls `enable_pr_auto_merge` → GitHub merges automatically once checks pass. For repos with no required checks, direct merge via `merge_pull_request` works today with no extra configuration.
