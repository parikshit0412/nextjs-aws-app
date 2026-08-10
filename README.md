# 🚀 Next.js AWS OIDC Practice Application

This is your standalone Next.js 15 application created to practice **AWS IAM OIDC Authentication** with **GitHub Actions**.

---

## 📂 Project Structure
* `src/app/page.tsx` - Next.js App Router main page.
* `.github/workflows/aws-oidc-test.yml` - GitHub Actions workflow with OIDC configuration.

---

## 🛠️ Step-by-Step Hands-on Instructions

### Step 1: Replace Account ID in `.github/workflows/aws-oidc-test.yml`
Open [.github/workflows/aws-oidc-test.yml](file:///d:/learn_AWS_with_AI/nextjs-aws-app/.github/workflows/aws-oidc-test.yml) and replace `YOUR_AWS_ACCOUNT_ID` with your actual 12-digit AWS Account ID.

```yaml
role-to-assume: arn:aws:iam::123456789012:role/GitHubActions-OIDC-Role
```

---

### Step 2: Push this Repository to GitHub
In your terminal, navigate to this project folder:

```bash
cd d:\learn_AWS_with_AI\nextjs-aws-app
git add .
git commit -m "feat: setup Next.js app with AWS OIDC workflow"
```

Create a new repository on your GitHub account (e.g. `nextjs-aws-app`), then link and push:

```bash
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/nextjs-aws-app.git
git branch -M main
git push -u origin main
```

---

### Step 3: Run AWS OIDC Setup
Ensure you have completed **Step 1 & Step 2** from [PRACTICE_LAB_01_OIDC.md](file:///d:/learn_AWS_with_AI/express-api/aws_interview_prep/PRACTICE_LAB_01_OIDC.md) in your AWS Console:
1. Created OIDC Provider `https://token.actions.githubusercontent.com`.
2. Created `GitHubActions-OIDC-Role` in IAM with condition matching `repo:YOUR_GITHUB_USERNAME/nextjs-aws-app:*`.

---

### Step 4: Check GitHub Actions Execution
1. Go to `https://github.com/YOUR_GITHUB_USERNAME/nextjs-aws-app/actions`.
2. Open the **Deploy Next.js & Verify AWS OIDC** workflow run.
3. Observe Next.js building, followed by AWS OIDC authenticating via temporary STS credentials!
