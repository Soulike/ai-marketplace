---
name: github-copilot-review
description: "Automate the GitHub Copilot PR review-and-fix loop. Use this skill IMMEDIATELY after a GitHub pull request is created — whether you just created it with 'gh pr create', the user says they've opened a PR, or any GitHub PR URL/number is mentioned in the context of a newly created PR. Also use when the user explicitly asks to run Copilot review, fix Copilot comments, or handle Copilot feedback on a PR. The skill adds Copilot as a reviewer, polls for comments, triages, fixes code, commits, pushes, replies, resolves, and repeats until the PR is clean. Trigger on: PR just created, 'gh pr create' output, 'created a PR', 'opened a pull request', 'copilot review', 'fix copilot comments', 'run review on my PR'."
---

# GitHub Copilot Review & Fix

Automatically request a GitHub Copilot review on a PR, then triage and fix every comment in a loop until the PR is clean.

## Critical Rule

NEVER merge the PR. The PR must always be merged by the user manually. This skill only handles review feedback — the merge decision belongs to the human.

## Prerequisites

- `gh` CLI authenticated with access to the repository
- The repository must have GitHub Copilot code review enabled

## Workflow

### Overview

```
1. Ask user if they want to start Copilot review & fix
2. Check gh CLI is available
3. Identify the PR
4. Request Copilot review  →  gh pr edit <PR> --add-reviewer @copilot
5. Poll for Copilot review comments
6. For each new comment  →  triage → fix → commit & push → reply & resolve
7. Re-request Copilot review (back to step 4)
8. Stop when Copilot has no new comments or no fix is needed
```

### Step 1 — Confirm with the User

This skill triggers automatically when a PR is created. Before starting the review loop, ask the user:

> A PR has been created. Would you like me to start the GitHub Copilot review & fix process?

If the user declines, stop. If the user confirms, continue.

### Step 2 — Check `gh` CLI

Verify the GitHub CLI is installed and authenticated:

```bash
gh --version
```

If `gh` is not found, tell the user:

> `gh` (GitHub CLI) is not installed or not on your PATH. Please install it from https://cli.github.com/ and run `gh auth login`, then try again.

Stop here — do not proceed without `gh`.

### Step 3 — Identify the PR

The PR number should already be known from context (e.g., output of `gh pr create`, a URL the user shared, or the current branch). Use it directly.

If for some reason the PR number is not available, try detecting it from the current branch:

```bash
gh pr view --json number -q '.number'
```

If that also fails, ask the user for the PR number. Otherwise, do not prompt — just proceed.

### Step 4 — Request Copilot Review

```bash
gh pr edit <PR> --add-reviewer @copilot
```

This tells GitHub to run a Copilot code review on the PR. Copilot takes some time to post its review, so you need to poll.

### Step 5 — Poll for Copilot Review Completion

Copilot review takes time. Poll the reviews API until the review state is no longer `PENDING`.

**Important:** Copilot uses **different logins** in different APIs:
- In the **reviews API**: `copilot-pull-request-reviewer[bot]`
- In the **comments API**: `Copilot` (capital C)

**Polling for review completion via the Reviews API:**

```bash
gh api repos/{owner}/{repo}/pulls/<PR>/reviews \
  --jq '[.[] | select(.user.login == "copilot-pull-request-reviewer[bot]")] | last | .state'
```

Possible terminal states: `COMMENTED`, `APPROVED`, `CHANGES_REQUESTED`.

**Polling strategy:**
- Wait 15 seconds between checks.
- After 5 minutes with no review appearing, inform the user that Copilot may not be enabled for the repo. Stop.

**Also check the summary issue comment** — each Copilot review produces exactly one **issue comment** on the PR that summarizes the review and states how many review comments it left:

```bash
gh api repos/{owner}/{repo}/issues/<PR>/comments --jq '[.[] | select(.user.login == "copilot-pull-request-reviewer[bot]" or (.user.login | test("copilot"; "i")))]'
```

Keep track of how many Copilot summary comments you've already seen. Poll until a **new** one appears (i.e., the count increases).

**Once the review is complete:**

1. If the review state is `APPROVED` and the summary says 0 review comments, the PR is clean — report success and stop.
2. Otherwise, fetch the inline review comments:
   ```bash
   gh api repos/{owner}/{repo}/pulls/<PR>/comments \
     --jq '[.[] | select(.user.login == "Copilot") | {id, path, line, body}]'
   ```
3. To get thread IDs (needed for resolving threads later), use GraphQL:
   ```graphql
   query {
     repository(owner: "<OWNER>", name: "<REPO>") {
       pullRequest(number: <PR>) {
         reviewThreads(first: 100) {
           nodes {
             id
             isResolved
             comments(first: 1) {
               nodes {
                 databaseId
                 body
                 path
                 author { login }
               }
             }
           }
         }
       }
     }
   }
   ```
   Filter threads where `author.login` is `copilot-pull-request-reviewer` and `isResolved` is `false`.
4. Filter to only the **new** review comments (ones you haven't already processed in a previous iteration) and proceed to Step 6.

### Step 6 — Triage and Fix Each Comment

For every new Copilot comment, decide whether it warrants a code change. Use your judgment:

#### Auto-fix (make the change, no user prompt)

| Category | Examples |
|---|---|
| Style / formatting | Whitespace, naming conventions, line length |
| Nit | Typos in comments, minor wording improvements |
| Missing guard / null check | Copilot suggests adding a defensive check |
| Unused import / variable | Dead code removal |
| Simple bug fix | Off-by-one, wrong comparison operator, missing return |
| Documentation | Missing or incorrect doc comments |

#### Skip (reply explaining why, do not change code)

| Category | Reason to skip |
|---|---|
| False positive | Copilot misunderstood the code — explain why the current code is correct |
| Design / architecture | The suggestion would require a significant refactor beyond the scope of this PR. Create an issue for it. |
| Intentional pattern | The code is written this way on purpose (e.g., performance, compatibility) |
| Already addressed | A previous fix in this same cycle already resolved the underlying issue |

#### For each comment you decide to fix:

1. **Read the relevant code** around the file and line Copilot referenced.
2. **Make the code change.** Focus on the **root cause**, not just the immediate symptom. Before editing, understand *why* the issue exists — then fix the underlying problem so it doesn't recur. For example, if Copilot flags a missing null check, ask whether the real issue is that the value should never be null at that point (fix the producer) rather than adding a defensive check at every consumer. Keep the change proportional: don't refactor unrelated code, but don't apply a surface-level band-aid when a root-cause fix prevents the same class of issue from recurring.
3. **Stage, commit, and push:**
   ```bash
   git add <changed-files>
   git commit -m "Fix Copilot review: <short description of what was fixed>"
   git push
   ```
   Prefer one commit per logical fix. If multiple comments touch the same file for trivial reasons, you can batch them into one commit.
4. **Reply to the comment** explaining what you did:
   ```bash
   gh api repos/{owner}/{repo}/pulls/comments/<comment-id>/replies \
     -f body="Fixed: <brief explanation of the change>"
   ```
5. **Resolve the review thread** using GraphQL:
   ```graphql
   mutation {
     resolveReviewThread(input: {threadId: "<THREAD_ID>"}) {
       thread { isResolved }
     }
   }
   ```
   The `THREAD_ID` is obtained from the GraphQL query in Step 5. Use `gh api graphql` to execute:
   ```bash
   gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "<THREAD_ID>"}) { thread { isResolved } } }'
   ```

#### For each comment you decide to skip:

1. **Reply to the comment** with a clear explanation:
   ```bash
   gh api repos/{owner}/{repo}/pulls/comments/<comment-id>/replies \
     -f body="Not fixing: <reason — e.g., 'This is intentional for backward compatibility'>"
   ```

### Step 7 — Re-request Review and Repeat

After all current comments have been addressed:

1. Re-request Copilot review to check the new changes:
   ```bash
   gh pr edit <PR> --add-reviewer @copilot
   ```
2. Go back to **Step 5** and poll for new comments.

### Step 8 — Termination

Stop the loop when any of these is true:
- Copilot posts no new comments after the latest push.
- All new comments are ones you decided to skip (no code changes were made in this iteration).
- You have completed 20 iterations of the loop (safety limit to avoid infinite cycles).

When done, summarize what happened:
- How many comments were fixed.
- How many comments were skipped (and why).
- How many review cycles it took.
- The final state of the PR.

## Important Notes

- **Do not prompt the user during the fix loop.** The whole point of this skill is hands-off automation. Triage using your own judgment.
- **Keep commits clean.** Each commit message should clearly describe what Copilot comment it addresses.
- **Respect the repo.** Only modify files that Copilot's comments reference. Do not make unrelated changes.
- **Watch for churn.** If Copilot keeps commenting on the same thing across iterations, skip it and note it in the summary — it may be a false positive or a pattern Copilot dislikes but the project intentionally uses.
- **Handle errors gracefully.** If `git push` fails (e.g., due to conflicts), stop and tell the user what happened instead of retrying blindly.
