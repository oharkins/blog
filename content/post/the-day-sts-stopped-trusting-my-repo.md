---
title: "The Day STS Stopped Trusting My Repo"
date: 2025-07-29T12:00:00-06:00
cover: images/sts-stoped-trusting-my-repo.png
---

Your workflow has `id-token: write`. The role ARN is right. The trust policy looks exactly like the one in every tutorial. And `aws-actions/configure-aws-credentials` still tells you to get lost:

```text
Not authorized to perform sts:AssumeRoleWithWebIdentity
```

Nothing changed on the AWS side. Nothing changed in the workflow. So what broke?

## The thing that changed was GitHub

GitHub now issues **immutable** OIDC subject claims for some repositories — new repos created after the rollout, plus repos that opted in, were renamed, or were transferred. The `sub` claim quietly grew a pair of numeric IDs:

| Format | Example |
|--------|---------|
| Legacy | `repo:ORG/REPO:ref:refs/heads/main` |
| Immutable | `repo:ORG@ORG_ID/REPO@REPO_ID:ref:refs/heads/main` |

The IDs make the subject stable across renames and transfers, which is a genuinely good idea. It also means every trust policy written against the legacy string stops matching. `StringLike` on `repo:oharkins/blog:*` does not match `repo:oharkins@9144752/blog@1186576031:ref:refs/heads/main`. STS denies, and the error message says nothing about why.

## The debugging trap

The obvious instinct is to print the subject from inside the workflow and compare. Don't:

```yaml
# This builds a LEGACY sub — it is NOT the JWT `sub`
- run: echo "sub=repo:${{ github.repository }}:ref:${{ github.ref }}"
```

`github.repository` is `ORG/REPO` with no numeric IDs, so you reconstruct the old format and confirm your own wrong assumption. `gh` and the GraphQL `nameWithOwner` field won't surface the IDs either. You end up staring at a string that looks correct while STS keeps rejecting a different one.

## Getting the real subject

Two reliable options:

**CloudTrail** — best when the assume already failed. Look for event source `sts.amazonaws.com`, event name `AssumeRoleWithWebIdentity`. The `userName` / PrincipalId field on the event is the exact `sub` GitHub sent. If you're in GovCloud, check the GovCloud trail (`us-gov-west-1`), not commercial.

**Decode the JWT** — request the OIDC token in the workflow (you already have `id-token: write`) and decode the `sub` claim, or drop in a debugger action like `actions-oidc-debugger`. While you're there, check the org and repo *Actions OIDC subject claim* settings to see whether you're on immutable or customizable claims.

Ours came back as:

```text
repo:oharkins@9144752/blog@1186576031:ref:refs/heads/main
```

## The fix

Update the `StringLike` (or `StringEquals`) condition on `token.actions.githubusercontent.com:sub` to the exact immutable values:

```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": [
        "repo:oharkins@9144752/blog@1186576031:ref:refs/heads/main",
        "repo:oharkins@9144752/blog@1186576031:pull_request"
      ]
    }
  }
}
```

Re-run the workflow. `configure-aws-credentials` succeeds, and the workflow itself never needed a change.

## Takeaways

- The error message is about *matching*, not *authorization*. When the trust policy looks right, assume the incoming claim is not what you think it is.
- Never reconstruct an OIDC subject from workflow context. Read it from the token or from CloudTrail — the identity provider is the source of truth.
- If you manage trust policies at scale, audit them now rather than discovering this one pipeline at a time. Renames and transfers flip repos into immutable subjects, so a policy that works today can break on a rename with no code change at all.
