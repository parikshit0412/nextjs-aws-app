# 🔐 Next.js + AWS OIDC Authentication: Engineering Architecture & Debugging Guide

This document details **how AWS OIDC authentication works** in this Next.js project, the **exact real-world production bug encountered**, and **how it was solved**.

---

## 💡 1. What Exactly Happens in this Next.js Project?

```
┌──────────────────┐    1. Request JWT    ┌──────────────────┐    2. Signed JWT    ┌──────────────────┐
│ GitHub Actions   │────────────────────> │  GitHub OIDC     │────────────────────>│ GitHub Actions   │
│ Runner           │                      │  Token Issuer    │                     │ Runner           │
└──────────────────┘                      └──────────────────┘                     └─────────┬────────┘
                                                                                             │ 3. sts:AssumeRoleWithWebIdentity
                                                                                             ▼
┌──────────────────┐  5. Ephemeral Token  ┌──────────────────┐   4. Validate JWT   ┌──────────────────┐
│ Next.js App      │<──────────────────── │    AWS STS       │<────────────────────│ AWS IAM          │
│ Deployment       │   (Valid max 1 hr)   └──────────────────┘   & Trust Policy    │ OIDC Provider    │
└──────────────────┘                                                               └──────────────────┘
```

1. When you push code to `main`, GitHub Actions triggers `.github/workflows/aws-oidc-test.yml`.
2. The GitHub runner requests a signed JSON Web Token (JWT) from GitHub's OIDC issuer (`https://token.actions.githubusercontent.com`).
3. The runner calls `aws-actions/configure-aws-credentials@v4` to send the JWT to **AWS STS** (Security Token Service).
4. AWS STS checks the **Identity Provider** and evaluates the **Trust Policy** on `GitHubActions-OIDC-Role`.
5. Once verified, AWS STS returns short-lived temporary security credentials (valid for max 1 hour) to the runner.

---

## 🚨 2. What Exact Problem Was Faced?

### Error Observed in GitHub Actions Logs:
```text
Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

### The Root Cause:
Standard AWS documentation examples state that the `sub` (subject) claim in the IAM Trust Policy should be formatted as:
```text
repo:YOUR_USERNAME/YOUR_REPO:ref:refs/heads/main
```

However, GitHub's OIDC issuer actually transmitted internal numeric user IDs and repository IDs inside the JWT `sub` payload:
```text
repo:parikshit0412@45668288/nextjs-aws-app@1330291112:ref:refs/heads/main
```

* **User ID:** `@45668288`
* **Repo ID:** `@1330291112`

Because `@45668288` and `@1330291112` were inside the token string, strict string comparison (`StringEquals`) in the AWS IAM Trust Policy against `repo:parikshit0412/nextjs-aws-app:ref:refs/heads/main` **failed**, causing AWS STS to reject role assumption.

---

## 🛠️ 3. How Was It Solved Step-by-Step?

### Step 1: Live JWT Claim Inspection in CI/CD
We added a temporary debug step in `.github/workflows/aws-oidc-test.yml` to fetch the raw JWT token from GitHub's OIDC runtime endpoint (`$ACTIONS_ID_TOKEN_REQUEST_URL`) and decode the payload:

```bash
TOKEN=$(curl -sS -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
  "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r '.value')
echo $TOKEN | cut -d'.' -f2 | python3 -c "import sys,base64,json; d=sys.stdin.read().strip(); d+='=='*(-len(d)%4); print(json.dumps(json.loads(base64.b64decode(d)), indent=2))"
```

This exposed the exact claim payload being sent by GitHub:
`"sub": "repo:parikshit0412@45668288/nextjs-aws-app@1330291112:ref:refs/heads/main"`

### Step 2: Updated IAM Trust Policy to Pattern Matching (`StringLike`)
We updated the `GitHubActions-OIDC-Role` Trust Policy in AWS IAM to use `StringLike` wildcard matching:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::463470979621:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:parikshit0412*/nextjs-aws-app*:*"
        }
      }
    }
  ]
}
```

### Step 3: Result
The workflow authenticated in **11 seconds** with zero static credentials stored in GitHub Secrets!

```json
{
  "UserId": "AROAWX2IF5ISSID56PJT4:GitHubActions",
  "Account": "463470979621",
  "Arn": "arn:aws:sts::463470979621:assumed-role/GitHubActions-OIDC-Role/GitHubActions"
}
```
