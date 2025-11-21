# List-IAM-Users
List AWS IAM users

# Github section
Create a github repo - I have created with a name "List-IAM-Users" where repo URL looks likes this: https://github.com/akshitkhosla/List-IAM-Users
where:
- "akshitkhosla" is the "organization",
- "List-IAM-Users" is the "repository" &
- "branch" I have kept as default one i.e. "main" 

# AWS Section
## 1. **Create a policy with a name "ListIAMUsers"**
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "iam:ListUsers",
            "Resource": "*"
        }
    ]
}
```

## 2. **Create an IDP (Identity Provider)**
```
Provider Type: OpenID Connect
Provider name: token.actions.githubusercontent.com
aud: sts.amazonaws.com
```
Reference:
- Above details of Github you can find in following document: https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
- To understand the terms like aud, iss and sub refer following document: https://docs.github.com/en/actions/reference/security/oidc


## 3. **Create a new Role "Github-Access-Role" with an option "Web Identity"**
- Mention Identity Provider, which you have created above- in my case it is saved with a name "token.actions.githubusercontent.com"
- Set the "Audience" as "sts.amazonaws.com"
- Set "GitHub organization" as "akshitkhosla"
- Set "GitHub repository" as "List-IAM-Users"
- Set "GitHub branch" as "main"
- Attach above created policy i.e. "ListIAMUsers" to it as it will give permission to list AWS IAM users.
- Mention the Role name and description as per your wish - In my case I have stated it as "Github-Access-Role"

It will create the role with a name "Github-Access-Role" with permission policy "ListIAMUsers" with Trust relationship as:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:akshitkhosla/List-IAM-Users:ref:refs/heads/main"
                }
            }
        }
    ]
}
```
# Github Actions Section
Create a "New Workflow" and select "Simple Workflow" or create a yml file under ".github/workflows/" in a project and edit the workflow as follow:
```
# This is a basic workflow to help you get started with Actions

name: List AWS IAM users 

# Controls when the workflow will run
on:
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

permissions:
  id-token: write     # REQUIRED for GitHub OIDC
  contents: read      # Required by checkout

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v4

      # Runs a single command using the runners shell
      - name: Run a one-line script
        run: echo Hello, world!

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          # role-to-assume: arn:aws:iam::<AWS_Account_ID>:role/Github-Access-Role
          aws-region: us-east-1
          
      # To run aws sts get-caller-identity, the IAM Role does NOT need any permissions and used to confirm you assumed the role.
      - name : Validate AWS Role
        run: aws sts get-caller-identity
          
      - name: List AWS Users
        run: aws iam list-users
```
### **Note:**
In place of role to assume you can add its value to github secret and call the variable in role to assume -  in my case it "${{ secrets.AWS_ROLE_TO_ASSUME }}"

**How to Add role ARN to your GitHub secrets (optional)**

You don’t need to store AWS keys — but you should store the role ARN in a repo secret (so the workflow can reference it):
- Repo → Settings → Secrets & variables → Actions → New repository secret
  - Name: AWS_ROLE_TO_ASSUME
  - Value: arn:aws:iam::123456789012:role/GitHubActionsOIDCRole

**Why a secret?** 

It keeps the role arn out of plain YAML and lets you change role without editing workflows.

# Workflow
```
[GitHub Actions Workflow Start]
            |
            v
[GitHub Runner requests id-token from GitHub OIDC endpoint]
            |
            v
[GitHub returns JWT id-token containing claims:
   iss, sub, aud (== sts.amazonaws.com), exp, iat, ...]
            |
            v
[aws-actions/configure-aws-credentials] (on runner)
  -> Calls AWS STS AssumeRoleWithWebIdentity(token=JWT)
            |
            v
[AWS STS validates:]
  - Is token issuer (iss) the provider we have? (token.actions...)
  - Does token audience (aud) match the provider's client-id-list?  <-- CHECK A
  - Does the role trust policy Condition evaluate to true?             <-- CHECK B
      - Check token.actions.githubusercontent.com:aud (StringEquals)
      - Check token.actions.githubusercontent.com:sub (StringLike / StringEquals)
            |
            v
If checks pass -> STS returns temporary credentials (AccessKeyId, SecretAccessKey, SessionToken)
            |
            v
Runner uses temporary credentials to run AWS CLI commands (e.g., aws iam list-users)
            |
            v
[Workflow continues] 
```
**Note:**
- CHECK A: AWS validates the aud from the token against the OIDC provider client-id-list.
- CHECK B: The role trust policy may further validate aud and sub. Both must pass.
