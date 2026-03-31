---
name: gh-pr-review-thread-workflow
description: Use when you need to fetch PR review threads via gh api graphql, judge whether each review comment requires code changes, reply in each thread, and resolve all threads after replying. Use for workflows that include listing review threads, posting thread replies, and closing threads on a GitHub PR.
---

# Gh Pr Review Thread Workflow

## Overview

**AUTONOMOUS OPERATION**: This workflow handles PR review comments automatically without requiring user confirmation for each new comment. The agent will:

1. Fetch all unresolved review threads from the specified PR
2. Analyze each comment to determine if code changes are needed
3. Implement fixes or post explanatory replies as appropriate
4. Reply to EVERY thread before resolution
5. Resolve ALL processed threads
6. Verify zero unresolved threads remain

**CRITICAL**: This workflow MUST complete entirely. Do NOT stop after code fixes - you MUST reply to every thread AND resolve every thread before ending.

## Workflow

### 1. Preconditions

Check `gh` is installed and authenticated before any API calls.

```bash
which gh
gh auth status
```

If auth fails, run `gh auth login -h github.com` manually and retry.

### 2. Identify Repo/PR

Get owner/name from the current repo when possible:

```bash
gh repo view --json name,owner -q '.owner.login + " " + .name'
```

Use the PR number provided by the user (e.g., `2`).

### 3. Fetch Review Threads (GraphQL)

```bash
gh api graphql \
  -F owner="OWNER" \
  -F name="REPO" \
  -F number=2 \
  -f query='
query($owner: String!, $name: String!, $number: Int!) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      reviewThreads(last: 100) {
        nodes {
          id
          isResolved
          comments(first: 50) {
            totalCount
            edges {
              node {
                id
                author { login }
                body
                createdAt
              }
            }
          }
        }
      }
    }
  }
}'
```

**TRACK UNRESOLVED THREADS**: Extract all threads where `isResolved: false` and create a processing list:

```
Unresolved threads to process:
- PRRT_xxx: [brief description]
- PRRT_yyy: [brief description]
...
Total: N threads
```

**AUTONOMOUS HANDLING**: When new review comments are detected:

- ✅ **Do NOT ask for user confirmation** - proceed immediately with handling
- ✅ Analyze each `isResolved: false` thread to determine required action
- ✅ Implement code changes if needed, or post explanatory reply if already addressed
- ✅ Continue processing all unresolved threads until complete

Focus on threads where `comments.totalCount > 1` (indicating replies exist) and `isResolved: false`. Determine whether each latest reviewer comment needs changes.

### 4. Implement Code Changes (If Required)

For threads requiring code fixes:

1. Read the affected files
2. Apply the fix as specified in the review comment
3. Repeat for ALL threads requiring changes
4. Commit and push changes:

```bash
git add -A
git commit -m "Fix review comments from [reviewer]"
git push
```

**IMPORTANT**: After code fixes, you MUST still proceed to steps 5-7 (replying and resolving threads).

### 5. Reply to EVERY Thread

For EACH unresolved thread, post a reply explaining the fix or rationale.

**Reply Tracking**: Maintain a list of replied threads:

```
Threads replied:
✅ PRRT_xxx - replied
✅ PRRT_yyy - replied
...
```

**Important Guidelines:**

- Do NOT include reviewer mention (@username) in replies - it is not required
- Use actual line breaks (real newlines) instead of escaped `\n` characters in body text
- Ensure each thread receives exactly ONE reply before proceeding to resolution
- If code changes were required, summarize what was fixed

Reply mutation:

```bash
gh api graphql \
  -F threadId="PRRT_xxx" \
  -f query='
mutation($threadId: ID!, $body: String!) {
  addPullRequestReviewThreadReply(
    input: { pullRequestReviewThreadId: $threadId, body: $body }
  ) { __typename }
}' \
  -f body="Reason or fix summary here."
```

**MUST COMPLETE**: Reply to ALL unresolved threads before proceeding. Do NOT skip any threads.

### 6. Verify No Duplicate Comments Before Resolution

**CRITICAL CHECK**: Before resolving any thread, verify that exactly one reply has been posted to each thread. This prevents duplicate replies and ensures no threads are missed.

To check for duplicates:

```bash
gh api graphql \
  -F owner="OWNER" \
  -F name="REPO" \
  -F number=PR_NUMBER \
  -f query='
query($owner: String!, $name: String!, $number: Int!, $threadId: ID!) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      reviewThreads(first: 10) {
        nodes {
          id
          isResolved
          comments(last: 10) {
            totalCount
            edges {
              node {
                author { login }
                body
              }
            }
          }
        }
      }
    }
  }
}' \
  -F threadId="PRRT_xxx"
```

Look for threads where `comments.totalCount > 1` and multiple replies are from the same author. If duplicates found, either delete them or proceed only if confirmed as intentional.

### 7. Resolve ALL Threads

Only resolve a thread after:

- ✅ Exactly ONE reply has been posted (no duplicates)
- ✅ No other threads were missed in this PR review cycle

Resolve each verified thread:

```bash
gh api graphql \
  -F threadId="PRRT_xxx" \
  -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { id }
  }
}'
```

**Resolution Tracking**: Maintain a list of resolved threads:

```
Threads resolved:
✅ PRRT_xxx - resolved
✅ PRRT_yyy - resolved
...
```

**MUST COMPLETE**: Resolve ALL threads that received replies. Do NOT skip any threads.

### 8. FINAL VERIFICATION (MANDATORY)

**This step is REQUIRED before ending the workflow.**

Query again to confirm ZERO unresolved threads remain:

```bash
gh api graphql \
  -F owner="OWNER" \
  -F name="REPO" \
  -F number=PR_NUMBER \
  -f query='
query($owner: String!, $name: String!, $number: Int!) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $number) {
      reviewThreads(last: 100) {
        nodes {
          id
          isResolved
        }
      }
    }
  }
}' | jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false)'
```

**SUCCESS CRITERIA**: The query returns empty output (no unresolved threads).

If unresolved threads remain:
1. Identify which threads were missed
2. Repeat steps 5-7 for remaining threads
3. Re-run verification

## Workflow Checklist

Before ending, verify ALL of the following:

- [ ] All unresolved threads identified and tracked
- [ ] Code changes implemented (if required)
- [ ] Code changes committed and pushed
- [ ] Reply posted to EVERY thread
- [ ] No duplicate replies exist
- [ ] ALL threads resolved
- [ ] Final verification confirms zero unresolved threads

## Notes

**AUTONOMOUS OPERATION GUIDELINES:**

- ✅ **No user confirmation required** for new review comments - handle them immediately
- ✅ `addPullRequestReviewThreadReply` uses `pullRequestReviewThreadId` (not `threadId`) in the input object.
- ✅ Resolve only after every thread has been verified to have exactly ONE reply.
- ✅ Always check for duplicate comments before resolving any thread.
- ✅ Ensure no threads are missed - all non-resolved threads from step 3 should be processed.
- ✅ **NEVER stop mid-workflow** - if you identify threads to process, you MUST complete the entire cycle (reply + resolve) before ending.
- ✅ **Final verification is mandatory** - always confirm zero unresolved threads remain.

## Common Failure Modes to Avoid

| Failure Mode | Prevention |
|-------------|------------|
| Stopping after code fixes without replying | Remember: code fix ≠ complete workflow. Always proceed to reply + resolve. |
| Skipping thread resolution | Use tracking checklist. Verify each thread is resolved. |
| Missing threads in batch processing | Track thread IDs explicitly. Count before and after. |
| Not verifying completion | Always run final verification query. |

## Example Complete Workflow

```
1. Fetch threads → Found 5 unresolved
2. Analyze → 3 need code fixes, 2 need explanation only
3. Fix code → Commit and push
4. Reply to all 5 threads
5. Verify no duplicates
6. Resolve all 5 threads
7. Final verification → 0 unresolved ✓
```

**Only end after step 7 confirms success.**
