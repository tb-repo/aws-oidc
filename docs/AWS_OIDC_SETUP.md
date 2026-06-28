# AWS OIDC Setup for GitHub Actions

These are the steps to configure AWS so the `aws-oidc.yml` workflow can assume an
IAM role using GitHub's OIDC provider (no long-lived access keys).

## Prerequisites

- AWS account with permissions to create IAM resources.
- AWS CLI configured locally (`aws configure`) **or** access to the AWS Console.
- Your GitHub repository in the form `<OWNER>/<REPO>` (e.g. `govindmaloo/oidc`).

Set these shell variables to reuse in the commands below:

```bash
export AWS_ACCOUNT_ID="232818307988"     # your AWS account ID
export GH_OWNER="govindmaloo"            # GitHub org/user
export GH_REPO="oidc"                    # GitHub repo name
export ROLE_NAME="github-actions-oidc"
```

> This repository's resources have already been provisioned in account
> `232818307988`: the OIDC provider, the `github-actions-oidc` role (trust scoped to
> `repo:govindmaloo/oidc:*`), and the `AWS_ROLE_ARN` secret. The steps below document
> how to reproduce the setup from scratch.

---

## Step 1: Create the GitHub OIDC identity provider in IAM

This tells AWS to trust tokens issued by GitHub Actions. You only need **one** per
AWS account.

### CLI

```bash
aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "ffffffffffffffffffffffffffffffffffffffff"
```

> Note: AWS now validates GitHub's certificate automatically, so the thumbprint is no
> longer security-critical, but the API still requires a value. Any 40-char hex string
> is accepted.

### Console

1. Go to **IAM → Identity providers → Add provider**.
2. Provider type: **OpenID Connect**.
3. Provider URL: `https://token.actions.githubusercontent.com` → click **Get thumbprint**.
4. Audience: `sts.amazonaws.com`.
5. Click **Add provider**.

---

## Step 2: Create the IAM role with a trust policy

Create a trust policy that only allows your specific repository to assume the role.

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GH_OWNER}/${GH_REPO}:*"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name "${ROLE_NAME}" \
  --assume-role-policy-document file://trust-policy.json
```

### Tightening the `sub` condition (recommended)

The `:*` allows any branch/PR/tag. Restrict it as needed:

| Goal | `sub` value |
| --- | --- |
| Only the `main` branch | `repo:${GH_OWNER}/${GH_REPO}:ref:refs/heads/main` |
| Only a GitHub Environment | `repo:${GH_OWNER}/${GH_REPO}:environment:production` |
| Only pull requests | `repo:${GH_OWNER}/${GH_REPO}:pull_request` |
| Any branch/tag (default here) | `repo:${GH_OWNER}/${GH_REPO}:*` |

---

## Step 3: Attach permissions to the role

`aws sts get-caller-identity` needs **no** permissions, so the role works as-is for the
demo. To let the workflow actually do work, attach a policy. Example (read-only S3):

```bash
aws iam attach-role-policy \
  --role-name "${ROLE_NAME}" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
```

Prefer scoped, least-privilege custom policies for real workloads.

---

## Step 4: Get the role ARN

```bash
aws iam get-role --role-name "${ROLE_NAME}" --query "Role.Arn" --output text
```

Output looks like: `arn:aws:iam::123456789012:role/github-actions-oidc`

---

## Step 5: Add the role ARN to GitHub

In your repository:

1. **Settings → Secrets and variables → Actions**.
2. **Secrets** tab → **New repository secret**:
   - Name: `AWS_ROLE_ARN`
   - Value: the ARN from Step 4.
3. (Optional) **Variables** tab → **New repository variable**:
   - Name: `AWS_REGION`
   - Value: e.g. `ap-south-1`.

You can also set these with the GitHub CLI:

```bash
gh secret set AWS_ROLE_ARN --body "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ROLE_NAME}"
gh variable set AWS_REGION --body "ap-south-1"
```

---

## Step 6: Run and verify

1. Push to `main`, or trigger manually via **Actions → AWS OIDC Caller Identity → Run workflow**.
2. The **Get caller identity** step should print JSON similar to:

```json
{
  "UserId": "AROAEXAMPLEID:github-actions-1234567890",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/github-actions-oidc/github-actions-1234567890"
}
```

If you see this, OIDC auth and the role assumption are working.

---

## Troubleshooting

- **`Not authorized to perform sts:AssumeRoleWithWebIdentity`**: the `sub`/`aud`
  condition in the trust policy doesn't match your repo/branch, or the OIDC provider
  ARN is wrong.
- **`Credentials could not be loaded`**: the workflow is missing
  `permissions: id-token: write` (already set in `aws-oidc.yml`).
- **`No OpenIDConnect provider found`**: Step 1 wasn't completed in this account/region.
- **Wrong account**: confirm `AWS_ROLE_ARN` points to the intended account.
