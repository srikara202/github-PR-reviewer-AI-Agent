# AI Code Reviewer: Full Production Setup Guide

This guide walks you all the way from an empty slate to a complete, live production system.
Every command shown here is written for the Windows Command Prompt (CMD). To open CMD, press Win + R, type `cmd`, and press Enter.
Please work through the steps in the exact order they appear. Do not skip any of them.

---

## Progress Tracker

| Phase | Status |
|---|---|
| Phase 1: Accounts | DONE |
| Phase 2: Install Tools | DONE |
| Phase 3: AWS Account Setup | DONE |
| Phase 4: GitHub Repository Setup | DONE |
| Phase 5: Create GitHub App | DONE |
| Phase 6: LangFuse Setup | DONE |
| Phase 7: OpenAI API Key | DONE |
| Phase 8: Provision AWS Infrastructure | DONE |
| Phase 9: Configure GitHub Secrets | DONE |
| Phase 10: First Deployment via GitHub Actions | DONE |
| Phase 11: Connect kubectl to EKS | DONE |
| Phase 12: Apply Kubernetes Manifests | DONE |
| Phase 13: Set Up Grafana and Prometheus | DONE |
| Phase 14: Set Up LangFuse Tracing | DONE |
| Phase 15: Update GitHub App Webhook URL | DONE |
| Phase 16: End-to-End Test | DONE |

---

## Table of Contents

1. [What This System Does](#1-what-this-system-does)
2. [How the Architecture Fits Together](#2-how-the-architecture-fits-together)
3. [What Your Computer Needs](#3-what-your-computer-needs)
4. [Phase 1: The Accounts You Need](#4-phase-1-the-accounts-you-need)
5. [Phase 2: Install the Tools on Windows](#5-phase-2-install-the-tools-on-windows)
6. [Phase 3: Setting Up Your AWS Account](#6-phase-3-setting-up-your-aws-account)
7. [Phase 4: Setting Up the GitHub Repository](#7-phase-4-setting-up-the-github-repository)
8. [Phase 5: Creating the GitHub App](#8-phase-5-creating-the-github-app)
9. [Phase 6: Setting Up LangFuse](#9-phase-6-setting-up-langfuse)
10. [Phase 7: Getting Your OpenAI API Key](#10-phase-7-getting-your-openai-api-key)
11. [Phase 8: Building the AWS Infrastructure with Terraform](#11-phase-8-building-the-aws-infrastructure-with-terraform)
12. [Phase 9: Adding Your GitHub Secrets](#12-phase-9-adding-your-github-secrets)
13. [Phase 10: Your First Deployment with GitHub Actions](#13-phase-10-your-first-deployment-with-github-actions)
14. [Phase 11: Connecting kubectl to EKS](#14-phase-11-connecting-kubectl-to-eks)
15. [Phase 12: Applying the Kubernetes Manifests](#15-phase-12-applying-the-kubernetes-manifests)
16. [Phase 13: Setting Up Grafana and Prometheus](#16-phase-13-setting-up-grafana-and-prometheus)
17. [Phase 14: Setting Up LangFuse Tracing](#17-phase-14-setting-up-langfuse-tracing)
18. [Phase 15: Updating the GitHub App Webhook URL](#18-phase-15-updating-the-github-app-webhook-url)
19. [Phase 16: The Full End-to-End Test](#19-phase-16-the-full-end-to-end-test)
20. [Reaching All the Dashboards and UIs](#20-reaching-all-the-dashboards-and-uis)
21. [How the Weekly Evaluation Works](#21-how-the-weekly-evaluation-works)
22. [Checking That Everything Is Running](#22-checking-that-everything-is-running)
23. [Troubleshooting](#23-troubleshooting)
24. [What It Costs](#24-what-it-costs)
25. [Teardown: Cleaning Everything Up](#25-teardown-cleaning-everything-up)
26. [Cleanup: The Right Order to Avoid Errors](#26-cleanup-the-right-order-to-avoid-errors)

---

## 1. What This System Does

Here is what happens, start to finish, whenever a developer opens a Pull Request in any GitHub repository that has your GitHub App installed:

1. GitHub fires off a webhook event to your system.
2. The gateway service confirms that the request really did come from GitHub.
3. The webhook service saves the PR into your database and adds an analysis job to the queue.
4. The orchestrator service pulls the real code diff from GitHub.
5. Four AI agents run side by side and inspect the code for:
   - Static analysis problems (high complexity, unused variables, poor naming)
   - Security weaknesses (the OWASP Top 10, hardcoded secrets, SQL injection)
   - Style problems (formatting and readability)
   - Architecture problems (mixed responsibilities, missing error handling)
6. The reviewer service writes the results back as inline comments right on the GitHub PR.
7. Once the PR is merged, the learner service saves the patterns so future reviews get better.

---

## 2. How the Architecture Fits Together

```
Internet
    |
    v
AWS Load Balancer (public IP)
    |
    v
gateway service (port 8000)      <- verifies GitHub HMAC signature
    |
    v
webhook service (port 8001)      <- parses event, writes to RDS PostgreSQL
    |
    v
Celery Worker (Redis broker)     <- runs background tasks
    |
    v
orchestrator service (port 8002) <- calls GitHub API, runs LangGraph AI agents
    |
    +-- static analysis agent  -+
    +-- security agent          +-- run in parallel via LangGraph
    +-- style agent             |
    +-- architecture agent     -+
    |
    v
reviewer service (port 8003)    <- posts comments to GitHub PR
    |
    v
[on merge] learner (port 8004)  <- stores patterns in PostgreSQL

Infrastructure:
- EKS (Kubernetes)      runs all services
- RDS PostgreSQL 15     stores PRs, findings, patterns
- ElastiCache Redis     Celery task queue
- ECR                   stores Docker images
- S3                    report storage
- LangFuse              AI call tracing and observability
- Prometheus + Grafana  metrics and dashboards
```

---

## 3. What Your Computer Needs

You will run all the deployment commands from your own Windows machine. The application itself lives entirely on AWS, so your computer only has to run a few command-line tools.

**What your computer needs:**
- Windows 10 or Windows 11
- At least 4 GB of RAM (8 GB is better)
- 5 GB of free disk space
- A reliable internet connection

**Your computer never builds Docker images and never runs the application itself.**
All of that work is handled by GitHub Actions, which runs on GitHub's own servers.
Your CMD window is used only for Terraform, kubectl, the AWS CLI, and Git.

**What actually runs on AWS (not on your machine):**
- All 5 services run as pods on EKS nodes (each node is a t3.medium with 2 vCPU and 4 GB of RAM, and there are 2 nodes)
- PostgreSQL runs on RDS
- Redis runs on ElastiCache

---

## 4. Phase 1: The Accounts You Need

Set up every one of these accounts before you do anything else.

### 4.1 AWS Account

1. Visit https://aws.amazon.com
2. Click "Create an AWS Account"
3. Type in your email, a password, and a name for the account
4. Add your credit card (you pay only for what you use, and the cost section at the end breaks this down)
5. Finish the phone verification
6. Pick "Basic support", which is free
7. Sign in to the AWS Console at https://console.aws.amazon.com

### 4.2 GitHub Account

1. Visit https://github.com
2. Create an account if you don't already have one
3. Confirm your email address

### 4.3 OpenAI Account

1. Visit https://platform.openai.com
2. Sign up using your email
3. Open Billing and add a payment method
4. Top up with at least $10 in credits, which is what lets the system call GPT-4o-mini
5. Open Billing, then Usage limits, and set a monthly cap of $20 so there are no surprises

### 4.4 LangFuse Account

LangFuse keeps a record of every single OpenAI API call your system makes. You can see the prompt that was sent, the reply that came back, how many tokens were used, and the cost. It is free.

1. Visit https://langfuse.com
2. Click "Get Started Free"
3. Sign up with GitHub or with your email
4. Once you are signed in, create a new project and name it `ai-code-reviewer`
5. Open Settings in the left sidebar
6. Click "API Keys"
7. Click "Create new API key"
8. Copy both keys and keep them safe:
   - Public Key (begins with `pk-lf-`): you will need this later
   - Secret Key (begins with `sk-lf-`): you will need this later

---

## 5. Phase 2: Install the Tools on Windows

Run CMD as an Administrator for all of the installation steps. To do that:
press Win, type `cmd`, right-click "Command Prompt", and choose "Run as administrator".

### 5.1 Install the AWS CLI

1. Open your browser and go to:
   https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html
2. Click the link that downloads the Windows installer (`AWSCLIV2.msi`)
3. Double-click the file you downloaded to start the installer
4. Leave every setting at its default and click through to the end
5. Close CMD and open a new one
6. Confirm it worked by running:
   ```
   aws --version
   ```
   You should see something like: `aws-cli/2.x.x Python/3.x.x Windows/...`

### 5.2 Install Terraform

1. Go to https://developer.hashicorp.com/terraform/install
2. Under Windows, click "AMD64" to download the zip file
3. Open the zip file you downloaded
4. Pull `terraform.exe` out of the zip
5. Move `terraform.exe` into `C:\Windows\System32\`
   (Doing this lets you run the `terraform` command from any folder in CMD)
6. Close CMD and open a new one
7. Confirm it worked by running:
   ```
   terraform --version
   ```
   You should see something like: `Terraform v1.x.x`

### 5.3 Install kubectl

1. Go to https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
2. Download the newest `kubectl.exe` file (that page has a direct download link)
3. Move `kubectl.exe` into `C:\Windows\System32\`
4. Close CMD and open a new one
5. Confirm it worked by running:
   ```
   kubectl version --client
   ```
   You should see a version number appear

### 5.4 Install Git

1. Go to https://git-scm.com/download/win
2. Download the 64-bit installer
3. Run the installer and leave every setting at its default
4. Click through to the end
5. Close CMD and open a new one
6. Confirm it worked by running:
   ```
   git --version
   ```
   You should see something like: `git version 2.x.x.windows.x`

### 5.5 Install Helm

You need Helm in order to install Prometheus and Grafana onto your Kubernetes cluster.

1. Go to https://helm.sh/docs/intro/install/
2. In the "From the Binary Releases" section, click the Windows link
3. Download the windows-amd64 zip
4. Pull `helm.exe` out of the zip
5. Move `helm.exe` into `C:\Windows\System32\`
6. Close CMD and open a new one
7. Confirm it worked by running:
   ```
   helm version
   ```

### 5.6 Confirm Every Tool Is Installed

Open a brand new CMD window and run all five commands. Each one has to print a version number:

```
aws --version
terraform --version
kubectl version --client
git --version
helm version
```

If any of them fail, return to that tool's step above and install it again.

---

## 6. Phase 3: Setting Up Your AWS Account

### 6.1 Create an IAM User for CLI Access

1. Go to https://console.aws.amazon.com
2. In the search bar at the top, type `IAM` and click IAM
3. In the left sidebar, click "Users"
4. Click "Create user"
5. Set the username to `ai-reviewer-deployer`
6. Click Next
7. Choose "Attach policies directly"
8. In the search box, type `AdministratorAccess`
9. Tick the box next to `AdministratorAccess`
10. Click Next
11. Click "Create user"
12. Click the `ai-reviewer-deployer` user you just made
13. Open the "Security credentials" tab
14. Scroll to "Access keys" and click "Create access key"
15. Choose "Command Line Interface (CLI)"
16. Tick the confirmation box at the bottom
17. Click Next, then "Create access key"
18. You will now see the Access Key ID and the Secret Access Key
19. **IMPORTANT: Copy both of these into a Notepad file right now.**
    The moment you leave this page, the Secret Access Key is gone for good and you can never view it again.

### 6.2 Connect the AWS CLI to Your Credentials

Open CMD and run:

```
aws configure
```

You will be asked four questions. Answer them exactly like this:

```
AWS Access Key ID [None]: paste your Access Key ID here and press Enter
AWS Secret Access Key [None]: paste your Secret Access Key here and press Enter
Default region name [None]: us-east-1
Default output format [None]: json
```

Now check that it works:

```
aws sts get-caller-identity
```

You should get back a response showing your account ID and user ARN. If you get an error instead, your credentials are wrong, so go back to step 6.1 and create fresh access keys.

### 6.3 Save Your AWS Account ID

Run this command:

```
aws sts get-caller-identity --query Account --output text
```

It prints a 12-digit number such as `123456789012`. Save that number. It is your `AWS_ACCOUNT_ID`, and you will use it in later steps.

### 6.4 Create an OIDC Identity Provider for GitHub Actions

This lets GitHub Actions deploy to AWS without you having to store long-lived AWS keys inside GitHub. It is the safe, recommended approach.

1. In the AWS Console, go to IAM
2. In the left sidebar, click "Identity providers"
3. Click "Add provider"
4. For Provider type, choose "OpenID Connect"
5. For Provider URL, enter `https://token.actions.githubusercontent.com`
6. Click "Get thumbprint", which fills itself in automatically
7. For Audience, enter `sts.amazonaws.com`
8. Click "Add provider"

### 6.5 Create an IAM Role for GitHub Actions

1. Go to IAM, then Roles, then "Create role"
2. For Trusted entity type, choose "Web identity"
3. For Identity provider, choose `token.actions.githubusercontent.com`
4. For Audience, choose `sts.amazonaws.com`
5. Click Next
6. You now need to add a condition. Scroll down and click "Add condition":
   - Condition operator: `StringLike`
   - Condition key: `token.actions.githubusercontent.com:sub`
   - Value: `repo:YOUR_GITHUB_USERNAME/ai-code-reviewer:*`
   (Swap YOUR_GITHUB_USERNAME for your real GitHub username, for example `repo:johnsmith/ai-code-reviewer:*`)
7. Click Next
8. In the search box, type `AdministratorAccess` and tick it
9. Click Next
10. Set the role name to `github-actions-ai-reviewer`
11. Click "Create role"
12. Click the `github-actions-ai-reviewer` role you just made
13. Copy the Role ARN at the top. It looks like:
    `arn:aws:iam::123456789012:role/github-actions-ai-reviewer`
14. Save this. It is your `AWS_ROLE_ARN`.

---

## 7. Phase 4: Setting Up the GitHub Repository

### 7.1 Create the Repository on GitHub

1. Go to https://github.com
2. Click the "+" icon at the top right, then "New repository"
3. Set the repository name to `ai-code-reviewer`
4. Set Visibility to Private
5. Leave "Add a README file" unticked
6. Click "Create repository"

### 7.2 Create a Personal Access Token for Pushing Code

1. On GitHub, click your profile picture, then Settings
2. Scroll right to the bottom of the left sidebar and click "Developer settings"
3. Click "Personal access tokens", then "Tokens (classic)"
4. Click "Generate new token", then "Generate new token (classic)"
5. For Note (the name), enter `ai-code-reviewer-push`
6. Set Expiration to 90 days
7. Tick these two scopes: `repo` and `workflow`
8. Click "Generate token"
9. Copy the token. It begins with `ghp_...`
10. Save it. You will use it as your password whenever Git asks for one.

### 7.3 Push the Project Code to GitHub

Open CMD and run these commands one at a time:

```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer"
```

```
git init
```

```
git add .
```

```
git commit -m "initial: full project setup"
```

```
git branch -M main
```

```
git remote add origin https://github.com/YOUR_USERNAME/ai-code-reviewer.git
```

```
git push -u origin main
```

When CMD asks for your username, type your GitHub username.
When it asks for your password, paste in the Personal Access Token from step 7.2 (not the password you use to log in to GitHub).

After this finishes, open https://github.com/YOUR_USERNAME/ai-code-reviewer and you should see all of the project files sitting there.

---

## 8. Phase 5: Creating the GitHub App

The GitHub App is the identity your bot uses to read PR diffs and to post its review comments on GitHub.

### 8.1 Open the GitHub App Creation Page

1. On GitHub, click your profile picture, then Settings
2. Scroll down the left sidebar and click "Developer settings"
3. Click "GitHub Apps"
4. Click "New GitHub App"

### 8.2 Fill In the App Details

Complete each field:

- **GitHub App name:** `ai-code-reviewer-bot`
  (This name has to be unique across all of GitHub. If it is already taken, tack on some numbers, like `ai-code-reviewer-bot-2024`)
- **Homepage URL:** `https://github.com/YOUR_USERNAME/ai-code-reviewer`
- **Webhook** section: make sure "Active" is ticked
- **Webhook URL:** `https://placeholder.com`
  (You will swap this for the real URL back in Phase 15, after the system is deployed)
- **Webhook secret:** Type in a long, random string. For example: `myWebhookSecret_abc123XYZ789`
  Write this down. It is your `GITHUB_WEBHOOK_SECRET`, and you will need it later.

### 8.3 Set the Repository Permissions

Scroll down to "Repository permissions" and change these two:

- **Contents:** Read-only
- **Pull requests:** Read and write

Leave every other permission set to "No access".

### 8.4 Subscribe to Events

Scroll down to "Subscribe to events" and tick this box:

- Pull request

### 8.5 Choose Where the App Can Be Installed

Select "Any account" so that you are free to install it on any of your repositories.

### 8.6 Create the App and Save the Credentials

1. Click "Create GitHub App" at the bottom
2. You are now on the app's settings page
3. Find the **App ID** near the top. It is a number like `12345678`.
   Save it. It is your `GITHUB_APP_ID`.
4. Scroll down to the "Private keys" section
5. Click "Generate a private key"
6. A `.pem` file downloads automatically into your Downloads folder
7. Open that `.pem` file in Notepad:
   - Open your Downloads folder
   - Right-click the `.pem` file, choose Open with, then Notepad
8. The file holds text that starts with `-----BEGIN RSA PRIVATE KEY-----`
9. Press Ctrl+A to select everything, then Ctrl+C to copy it
10. Save it somewhere safe. It is your `GITHUB_APP_PRIVATE_KEY`.

### 8.7 Install the App on Your Repository

1. On the GitHub App page, find "Install App" in the left sidebar and click it
2. Your GitHub account appears in the list, so click "Install"
3. Choose "Only select repositories"
4. Pick the `ai-code-reviewer` repository (along with any other repos you want the bot to review)
5. Click "Install"

---

## 9. Phase 6: Setting Up LangFuse

LangFuse is already built into the orchestrator service's code. You do not need to change any code. You only need your API keys.

### 9.1 Get Your LangFuse API Keys

1. Go to https://cloud.langfuse.com and log in
2. Click your `ai-code-reviewer` project
3. Click "Settings" in the left sidebar
4. Click "API Keys"
5. Click "Create new API key"
6. Copy and save both of them:
   - **Public Key:** begins with `pk-lf-`, and this is your `LANGFUSE_PUBLIC_KEY`
   - **Secret Key:** begins with `sk-lf-`, and this is your `LANGFUSE_SECRET_KEY`

---

## 10. Phase 7: Getting Your OpenAI API Key

1. Go to https://platform.openai.com
2. Click your profile icon at the top right, then "API keys"
3. Click "Create new secret key"
4. Name it `ai-code-reviewer`
5. Click "Create secret key"
6. Copy the key. It begins with `sk-proj-...` or `sk-...`
7. Save it. This is your `OPENAI_API_KEY`.

---

## 11. Phase 8: Building the AWS Infrastructure with Terraform

This step creates every AWS resource your system depends on: the Kubernetes cluster, the database, Redis, the container registry, and storage. You do this only once.

### 11.1 Open CMD and Go to the Terraform Folder

```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\terraform"
```

### 11.2 Initialize Terraform

```
terraform init
```

This downloads every provider and module it needs from the internet. You will see plenty of output. It ends with:
```
Terraform has been successfully initialized!
```

If you get errors, double-check your internet connection and make sure you are in the right folder.

### 11.3 Preview What Terraform Is About to Create

```
terraform plan -var="cluster_name=ai-code-reviewer" -var="db_password=YourStrongPassword123!" -var="environment=production"
```

Read through the output. It lists everything it intends to build:
- 1 VPC (a private network on AWS)
- 1 EKS cluster with 2 worker nodes
- 1 RDS PostgreSQL database
- 1 ElastiCache Redis cluster
- 1 S3 bucket
- 5 ECR repositories (one for each service)

If you see errors, check your AWS credentials with:
```
aws sts get-caller-identity
```

### 11.4 Create Everything on AWS

```
terraform apply -var="cluster_name=ai-code-reviewer" -var="db_password=YourStrongPassword123!" -var="environment=production"
```

Terraform shows you the plan one more time and then asks:
```
Do you want to perform these actions? Enter a value:
```

Type `yes` and press Enter.

**This takes somewhere between 15 and 25 minutes.** The EKS cluster is the slowest piece. Keep CMD open and wait it out.

You will see lines such as:
```
aws_vpc.main: Creating...
aws_eks_cluster.main: Creating...
aws_db_instance.postgres: Creating...
```

When it is done, you will see:
```
Apply complete! Resources: XX added, 0 changed, 0 destroyed.
```

### 11.5 Save All of the Output Values

```
terraform output
```

Copy every line of this output into a text file. It will look like this:

```
eks_cluster_endpoint   = "https://XXXXX.gr7.us-east-1.eks.amazonaws.com"
rds_endpoint           = "ai-code-reviewer-postgres.XXXX.us-east-1.rds.amazonaws.com:5432"
redis_endpoint         = "ai-code-reviewer-redis.XXXX.cache.amazonaws.com:6379"
ecr_gateway_url        = "123456789012.dkr.ecr.us-east-1.amazonaws.com/gateway"
ecr_webhook_url        = "123456789012.dkr.ecr.us-east-1.amazonaws.com/webhook"
ecr_orchestrator_url   = "123456789012.dkr.ecr.us-east-1.amazonaws.com/orchestrator"
ecr_reviewer_url       = "123456789012.dkr.ecr.us-east-1.amazonaws.com/reviewer"
ecr_learner_url        = "123456789012.dkr.ecr.us-east-1.amazonaws.com/learner"
```

You will need these values in the steps that follow.

---

## 12. Phase 9: Adding Your GitHub Secrets

GitHub Actions uses these secrets when it automatically builds and deploys your code. You add them just once, and GitHub stores them securely.

### 12.1 Open Your Repository's Secrets Page

1. Go to https://github.com/YOUR_USERNAME/ai-code-reviewer
2. Click "Settings" (the tab along the top of the repo page)
3. In the left sidebar, click "Secrets and variables"
4. Click "Actions"
5. Click "New repository secret"

### 12.2 Add Each Secret One at a Time

For every secret: click "New repository secret", fill in the Name and the Value, then click "Add secret".

**AWS secrets, three in total:**

| Name | Value |
|---|---|
| `AWS_ACCOUNT_ID` | the 12-digit number from Step 6.3 |
| `AWS_ROLE_ARN` | the full ARN from Step 6.5, which looks like `arn:aws:iam::123456789012:role/github-actions-ai-reviewer` |
| `EKS_CLUSTER_NAME` | `ai-code-reviewer` |

Once you have added them all, you should have **3 secrets** in total: `AWS_ACCOUNT_ID`, `AWS_ROLE_ARN`, and `EKS_CLUSTER_NAME`.

> **Note: `OPENAI_API_KEY` and `DATABASE_URL` do NOT belong here.**
> The evaluate job runs inside the Kubernetes cluster and reads these values straight from `infra/k8s/secret.yaml` while it runs. They do not need to be GitHub Actions secrets.

> **Important: here is what does NOT go here.**
> - `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` are not GitHub Actions secrets. No pipeline uses them. They go only into `infra/k8s/secret.yaml` in Phase 12, so that the Kubernetes pods can read them while running.
> - `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`, and `GITHUB_WEBHOOK_SECRET` also go only into `secret.yaml`. GitHub blocks any Actions secret whose name starts with `GITHUB_`.
> - The pipelines only build Docker images and run `kubectl set image`. They never create or change the Kubernetes secret. That is done by hand, once, in Phase 12.

---

## 13. Phase 10: Your First Deployment with GitHub Actions

> **IMPORTANT:** If you run the pipelines right now, only the **test** and **build-and-push** jobs will succeed. The **deploy** job will fail with "deployment not found", and that is completely expected. The deploy job needs the Kubernetes deployments to exist first, which only happens once you finish Phase 12. After Phase 12 is done, re-run the pipelines and all 3 jobs will pass.

### How the CI/CD Pipelines Are Built (One per Service)

This project uses **5 separate pipelines**, one for each service. Each one runs on its own:

| Pipeline file | Runs when you change |
|---|---|
| `gateway.yml` | anything inside `services/gateway/` |
| `webhook.yml` | anything inside `services/webhook/` |
| `orchestrator.yml` | anything inside `services/orchestrator/` |
| `reviewer.yml` | anything inside `services/reviewer/` |
| `learner.yml` | anything inside `services/learner/` |

In other words, if you fix a bug in the orchestrator, only the orchestrator pipeline runs and the other 4 services are left alone. Every pipeline runs the same 3 jobs in order: **test, then build-and-push, then deploy**.

### 13.1 Trigger All 5 Pipelines for the First Deployment

Since this is the very first run, you need to kick off all 5 pipelines at once. The simplest way is to make a small change to a file inside each service:

Open CMD:

```
for %s in (gateway webhook orchestrator reviewer learner) do echo 6666 > services\%s\deploy.txt
```

```
git add services/
```

```
git commit -m "deploy: trigger initial deployment for all services"
```

```
git push origin main
```

This push touches every service folder, so all 5 pipelines start at the same time.

### 13.2 Watch the Pipelines Run

1. Go to https://github.com/data-guru0/MAJOR-PROJECT-KRISH-SIR
2. Click the "Actions" tab at the top
3. You will see **5 separate workflow runs** all kicking off together, one for each service
4. Click any of them to watch it go

Each workflow runs three jobs in sequence:

**Job 1: test (1 to 2 minutes)**
- Sets up Python 3.11
- Installs the pip packages for that one service
- Runs pytest for that one service

**Job 2: build-and-push (3 to 5 minutes)**
- Logs in to your AWS ECR
- Builds the Docker image for that one service
- Pushes it to ECR, tagged with the git commit SHA

**Job 3: deploy (1 minute)**
- Connects to your EKS cluster
- Runs `kubectl set image` for that one service

**Every one of the three jobs, across all 5 workflows, has to show a green checkmark.**

If a job fails, click it, then click the failed step to read the error message.


### 13.3 Re-Trigger All 5 Pipelines After Phase 12 Is Done

Once you finish Phase 12 (applying all the Kubernetes manifests), you have to run all 5 pipelines again so that the deploy job can finish successfully. The deploy job only works when the Kubernetes deployments already exist.

Every service folder has a `deploy.txt` file made for exactly this. Bumping the number inside it triggers the pipelines without you having to touch any real code:

```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer"
```

```
for %s in (gateway webhook orchestrator reviewer learner) do echo 6666 > services\%s\deploy.txt
```

```
git add services\
```

```
git commit -m "deploy: trigger all 5 pipelines after k8s setup"
```

```
git push origin main
```

Next time you need to trigger them, use `echo 3 >`, then `echo 4 >`, and so on.

Go to GitHub, open the Actions tab, and wait for all 3 jobs (test, build-and-push, deploy) in all 5 pipelines to turn green.

### 13.4 How to Redeploy a Single Service Later

When you change just one service, only that service's pipeline runs by itself. For example, if you edit `services/orchestrator/graph.py`:

```
git add services/orchestrator/graph.py
git commit -m "fix: improve security agent prompt"
git push origin main
```

Only the **Orchestrator CI/CD** pipeline runs, and the other 4 services stay untouched.

---

## 14. Phase 11: Connecting kubectl to EKS

kubectl is the command-line tool you use to manage your Kubernetes cluster. You have to point it at your EKS cluster on AWS.

### 14.1 Update Your kubeconfig File

Open CMD and run:

```
aws eks update-kubeconfig --name ai-code-reviewer --region us-east-1
```

You will see:
```
Added new context arn:aws:eks:us-east-1:123456789012:cluster/ai-code-reviewer to C:\Users\YourName\.kube\config
```

This writes a config file on your computer that kubectl uses to reach AWS.

### 14.2 Check the Connection

```
kubectl get nodes
```

You should see two nodes, each with STATUS set to Ready:

```
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-1-xxx.ec2.internal    Ready    <none>   5m    v1.29.x
ip-10-0-2-xxx.ec2.internal    Ready    <none>   5m    v1.29.x
```

If they say `NotReady`, wait 3 minutes and run the command again. EKS nodes need a few minutes to come fully online.

If you get "no server found" or a connection refused error, your `aws configure` credentials may be wrong. Check them with `aws sts get-caller-identity`.

---

## 15. Phase 12: Applying the Kubernetes Manifests

This deploys all of your services onto the EKS cluster.

### 15.1 Update the ConfigMap with Your Real Redis URL

**Why this matters:** The `configmap.yaml` file ships with a placeholder Redis URL. If you skip this step, every service will fail to reach Redis (ElastiCache), and the whole system will quietly return 500 errors.

First, get your Redis endpoint from Terraform:
```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\terraform"
terraform output
```

Copy the `redis_endpoint` value. It looks like:
```
ai-code-reviewer-redis.khmhzg.0001.use1.cache.amazonaws.com:6379
```

Now open the ConfigMap file:
```
notepad "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\k8s\configmap.yaml"
```

Find this line:
```
REDIS_URL: "redis://redis:6379/0"
```

Replace it with your real endpoint (drop the `:6379` from the endpoint you copied, because the URL already includes it):
```
REDIS_URL: "redis://YOUR_REDIS_ENDPOINT:6379/0"
```

For example:
```
REDIS_URL: "redis://ai-code-reviewer-redis.khmhzg.0001.use1.cache.amazonaws.com:6379/0"
```

Save the file.

### 15.2 Fill In the Kubernetes Secret File

You need to open `infra\k8s\secret.yaml` and put your real secret values into it.

Open CMD:

```
notepad "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\k8s\secret.yaml"
```

Replace all the blank values with your real ones. When you are finished, the file should look like this:

```yaml
# Fill in all values before applying with: kubectl apply -f secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  GITHUB_WEBHOOK_SECRET: "myWebhookSecret_abc123XYZ789"
  GITHUB_APP_ID: "12345678"
  GITHUB_APP_PRIVATE_KEY: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEA1234...
    (all lines of your private key go here, keeping the line breaks)
    -----END RSA PRIVATE KEY-----
  OPENAI_API_KEY: "sk-proj-abc123..."
  LANGFUSE_PUBLIC_KEY: "pk-lf-abc123..."
  LANGFUSE_SECRET_KEY: "sk-lf-abc123..."
  DATABASE_URL: "postgresql+asyncpg://dbadmin:YourStrongPassword123!@ai-code-reviewer-postgres.abc123.us-east-1.rds.amazonaws.com/codereviewer"
```

**A few important notes:**
- The `DATABASE_URL` here has to use `postgresql+asyncpg://` (with asyncpg). This is different from the one you put into the GitHub secrets.
- The private key must keep every line break exactly as it appears in the `.pem` file.
- `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` go here only. Do not add them to the GitHub Actions secrets. The pipeline never touches this file, and the orchestrator pod reads these keys directly while it runs.
- Save the file with Ctrl+S, then close Notepad.

### 15.3 Go to the k8s Folder

```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\k8s"
```

### 15.4 Apply Everything in the Exact Order Below

Run each command and let it finish before you start the next one.

**Step 1: Apply the ConfigMap (the non-secret environment variables):**
```
kubectl apply -f configmap.yaml
```
Expected output: `configmap/app-config created`

**Step 2: Apply the Secrets:**
```
kubectl apply -f secret.yaml
```
Expected output: `secret/app-secrets created`

**Step 3: Run the database migration.**

The `migration-job.yaml` always uses the `latest` tag, which the webhook pipeline pushes automatically on every deployment. You do not need to change anything by hand.

Just apply it as is:
```
kubectl apply -f migration-job.yaml
```
Expected output: `job.batch/db-migrate created`

Now wait for the migration to finish. This is what creates all the tables in your RDS database:
```
kubectl wait --for=condition=complete job/db-migrate --timeout=120s
```
Expected output: `job.batch/db-migrate condition met`

If it times out, take a look at what went wrong:
```
kubectl logs job/db-migrate
```

**Step 4: Deploy all 5 services:**
```
kubectl apply -f gateway.yaml
kubectl apply -f webhook.yaml
kubectl apply -f orchestrator.yaml
kubectl apply -f reviewer.yaml
kubectl apply -f learner.yaml
```

**Step 5: Deploy the Celery workers:**
```
kubectl apply -f webhook-worker.yaml
kubectl apply -f learner-worker.yaml
```

**Step 6: Apply the autoscaler for the orchestrator:**
```
kubectl apply -f hpa.yaml
```

**Step 7: Install the AWS Load Balancer Controller (needed for ingress).**

This controller is what turns your ingress definition into an actual AWS ALB. Without it, the ingress will never get an ADDRESS.

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=ai-code-reviewer --set serviceAccount.create=true
```

Give the controller about 2 minutes to start, then check:
```
kubectl get pods -n kube-system
```

Every pod should say `Running`.

**Step 8: Apply the ingress (this is what creates your public URL):**
```
kubectl apply -f ingress.yaml
```

### 15.5 Confirm All Pods Are Running

```
kubectl get pods
```

Give the pods 3 to 5 minutes to start. Run this command a few times until every pod says `Running`:

```
NAME                                READY   STATUS    RESTARTS   AGE
gateway-7d4f9b-xxx                  1/1     Running   0          3m
gateway-7d4f9b-yyy                  1/1     Running   0          3m
webhook-6c8d7b-xxx                  1/1     Running   0          3m
webhook-6c8d7b-yyy                  1/1     Running   0          3m
webhook-worker-5f6g7h-xxx           1/1     Running   0          2m
orchestrator-9h2j3k-xxx             1/1     Running   0          3m
orchestrator-9h2j3k-yyy             1/1     Running   0          3m
reviewer-4l5m6n-xxx                 1/1     Running   0          3m
reviewer-4l5m6n-yyy                 1/1     Running   0          3m
learner-7p8q9r-xxx                  1/1     Running   0          3m
learner-7p8q9r-yyy                  1/1     Running   0          3m
learner-worker-2s3t4u-xxx           1/1     Running   0          2m
```

If any pod shows `CrashLoopBackOff` or `Error`, jump to the Troubleshooting section at the end.

### 15.6 Get Your Public URL

```
kubectl get ingress gateway-ingress
```

Run this once a minute for about 3 minutes. The moment the ADDRESS column shows a value, your system is live:

```
NAME              CLASS   HOSTS   ADDRESS                                                  PORTS
gateway-ingress   alb     *       k8s-default-gateway-abc123.us-east-1.elb.amazonaws.com   80,443
```

Copy the ADDRESS value and save it. This is your system's public URL.
Your complete webhook endpoint is: `https://ADDRESS/webhook/github`

Note: if ADDRESS is empty, wait 3 to 5 minutes and run the command again. The ALB takes a little while to come up.

---

## 16. Phase 13: Setting Up Grafana and Prometheus

Grafana is the dashboard you look at. Prometheus is what gathers the metrics from your services. You install both onto your EKS cluster using Helm.

### 16.1 Add the Helm Chart Repositories

Open CMD and run:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```
helm repo add grafana https://grafana.github.io/helm-charts
```

```
helm repo update
```

### 16.2 Install the EBS CSI Driver (Needed for Prometheus Storage)

Prometheus needs storage that sticks around. For EKS to hand out EBS volumes to pods, it needs the EBS CSI driver.

**Step 1: Find your node group role name:**
```
aws eks list-nodegroups --cluster-name ai-code-reviewer --region us-east-1
```
```
aws eks describe-nodegroup --cluster-name ai-code-reviewer --nodegroup-name YOUR_NODEGROUP_NAME --region us-east-1
```
Look for the `nodeRole` field and copy just the part of the name after the final `/`. It looks like `default-eks-node-group-XXXXXXXXXXXX`.

**Step 2: Attach the EBS CSI policy to that node group role.**

**Windows CMD:**
```
aws iam attach-role-policy ^
  --role-name YOUR_NODE_ROLE_NAME ^
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

**Mac/Linux:**
```
aws iam attach-role-policy \
  --role-name YOUR_NODE_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

**Step 3: Install the EBS CSI driver add-on:**
```
aws eks create-addon --cluster-name ai-code-reviewer --addon-name aws-ebs-csi-driver --region us-east-1
```

**Step 4: Wait until it reads ACTIVE:**
```
aws eks describe-addon --cluster-name ai-code-reviewer --addon-name aws-ebs-csi-driver --region us-east-1 --query addon.status --output text
```

Run Step 4 once a minute until it says `ACTIVE`, and only then move on.

Do not continue until the output reads `"ACTIVE"`.

### 16.3 Install Prometheus

**Windows CMD:**
```
helm install prometheus prometheus-community/prometheus ^
  --namespace monitoring --create-namespace ^
  --set server.persistentVolume.size=8Gi ^
  --set server.persistentVolume.storageClass=gp2 ^
  --set alertmanager.persistence.storageClass=gp2
```

**Mac/Linux:**
```
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  --set server.persistentVolume.size=8Gi \
  --set server.persistentVolume.storageClass=gp2 \
  --set alertmanager.persistence.storageClass=gp2
```

This installs Prometheus into a brand new namespace called `monitoring`. Give it about 2 minutes.

Check that it is up:
```
kubectl get pods -n monitoring
```

Every pod should say `Running`. This can take a few minutes.

### 16.4 Apply Your Prometheus Scrape Config

Your project already includes a config file that tells Prometheus where to find all 5 services. Apply it:

```
kubectl create configmap prometheus-config --from-file=prometheus.yml="D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\monitoring\prometheus.yml" -n monitoring --dry-run=client -o yaml | kubectl apply -f -
```

### 16.5 Install Grafana

Grafana needs some NLB annotations so that the AWS Load Balancer Controller gives it a public IP. Without those annotations, its EXTERNAL-IP stays stuck on `<pending>` forever.

If Grafana is already installed from an earlier attempt, remove it first:
```
helm uninstall grafana -n monitoring
```

**Windows CMD:**
```
helm install grafana grafana/grafana ^
  --namespace monitoring ^
  --set persistence.enabled=true ^
  --set persistence.size=5Gi ^
  --set persistence.storageClassName=gp2 ^
  --set service.type=LoadBalancer ^
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-type=external" ^
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type=ip" ^
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-scheme=internet-facing"
```

**Mac/Linux:**
```
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=5Gi \
  --set persistence.storageClassName=gp2 \
  --set service.type=LoadBalancer \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-type=external" \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type=ip" \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-scheme=internet-facing"
```

Wait 2 to 3 minutes, then check:
```
kubectl get svc grafana -n monitoring
```
The EXTERNAL-IP column shows a URL once the NLB is ready.

### 16.6 Get the Grafana Admin Password

Run this command, and it will print the password right in your CMD window:

```
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" > %TEMP%\grafana_pass.txt && certutil -decode %TEMP%\grafana_pass.txt %TEMP%\grafana_pass_decoded.txt && type %TEMP%\grafana_pass_decoded.txt
```

If that command feels too fiddly, here is a simpler option. Install Git Bash and run:
```
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

Save the password it prints. The username is always `admin`.

### 16.7 Get the Grafana URL

```
kubectl get svc -n monitoring grafana
```

Wait 2 to 3 minutes. The EXTERNAL-IP column will show a URL such as `xxx.us-east-1.elb.amazonaws.com`.
Open your browser and go to `http://EXTERNAL-IP`.

### 16.8 Log In to Grafana

1. Open the Grafana URL in Chrome or any other browser
2. A login page appears
3. Username: `admin`
4. Password: the one you got in step 16.6
5. It will ask you to set a new password, so choose one and save it

### 16.9 Add Prometheus as a Data Source

1. In Grafana, look in the left sidebar and click the gear icon (Configuration)
2. Click "Data sources"
3. Click "Add data source"
4. Click "Prometheus"
5. In the URL field, enter: `http://prometheus-server.monitoring.svc.cluster.local`
6. Scroll down and click "Save & Test"
7. You should see a green message: "Data source is working"

### 16.10 Import Your Ready-Made Dashboard

Your project already includes a complete Grafana dashboard saved as a JSON file.

1. In Grafana, click the "+" icon in the left sidebar
2. Click "Import"
3. Click "Upload JSON file"
4. Browse to `D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\monitoring\grafana-dashboard.json`
5. Select the Prometheus data source you just added
6. Click "Import"

You now have a live dashboard with 5 sections, one for each service. Every section has three panels:
- **Request Rate:** how many HTTP requests per minute each service is handling
- **p99 Latency:** the response time for the slowest 1% of requests (a good way to spot something running slow)
- **Error Rate:** how many 5xx errors per minute (a good way to spot something broken)

The dashboard refreshes itself every 30 seconds.

---

## 17. Phase 14: Setting Up LangFuse Tracing

LangFuse is already built into the orchestrator code. You just need to point it at the right host and make sure the keys in the Kubernetes secret are correct.

### 17.1 Check Which LangFuse Region You Are On

LangFuse has two regions. Log in at https://langfuse.com and look at the URL in your browser right after you log in:
- `cloud.langfuse.com` means you are on the **EU** region
- `us.cloud.langfuse.com` means you are on the **US** region

### 17.2 Update configmap.yaml with the Right Host

Open `infra/k8s/configmap.yaml` and set `LANGFUSE_HOST` to match your region:

**EU:**
```yaml
LANGFUSE_HOST: "https://cloud.langfuse.com"
```

**US:**
```yaml
LANGFUSE_HOST: "https://us.cloud.langfuse.com"
```

Save the file, then apply it and restart the orchestrator:
```
kubectl apply -f infra/k8s/configmap.yaml
kubectl rollout restart deployment orchestrator
```

### 17.3 Double-Check the LangFuse Keys in the Kubernetes Secret

The keys live in the Kubernetes secret named `app-secrets` (which you set during Phase 12). They are NOT read from the GitHub Actions secrets, and the pipeline never touches `app-secrets`.

If you created new LangFuse keys after Phase 12, you have to update the secret by hand:
```
kubectl get secret app-secrets -o jsonpath='{.data.LANGFUSE_PUBLIC_KEY}' | base64 --decode
```

If the key it shows is wrong, update it:
```
kubectl create secret generic app-secrets --dry-run=client -o yaml ^
  --from-literal=LANGFUSE_PUBLIC_KEY=pk-lf-YOUR-KEY ^
  --from-literal=LANGFUSE_SECRET_KEY=sk-lf-YOUR-KEY ^
  | kubectl apply -f -
kubectl rollout restart deployment orchestrator
```

### 17.4 Confirm Traces Are Showing Up

After you open a PR, check the orchestrator logs for errors:
```
kubectl logs -l app=orchestrator --tail=50 | grep -i "langfuse\|401\|export"
```

If you see `Failed to export span batch code: 401`, it means either the keys are wrong or the host region is wrong. Fix it using 17.2 and 17.3.

### 17.5 Open the LangFuse Dashboard

Go to your region's URL, log in, click your `ai-code-reviewer` project, then click **Traces** in the left sidebar.

After a PR has been analyzed, you will see:
- One trace for each PR
- 4 spans inside each trace running side by side (static_analysis, security, style, architecture)
- The exact prompt sent and the response received for each span
- The token count, cost, and latency for each call

---

## 18. Phase 15: Updating the GitHub App Webhook URL

Now that your system is deployed and you have a public URL, you need to tell GitHub where to send its webhook events.

### 18.1 Update the Webhook URL in Your GitHub App

First, get your ingress address:
```
kubectl get ingress gateway-ingress
```

Your current ingress address is:
```
k8s-default-gatewayi-d0c66c7699-299311758.us-east-1.elb.amazonaws.com
```

Your complete webhook URL is:
```
http://k8s-default-gatewayi-d0c66c7699-299311758.us-east-1.elb.amazonaws.com/webhook/github
```

Now update the GitHub App:
1. On GitHub, click your profile picture, then Settings
2. In the left sidebar, click "Developer settings", then "GitHub Apps"
3. Click "Edit" next to `ai-code-reviewer-bot`
4. Find the "Webhook URL" field
5. Delete `https://placeholder.com`
6. Enter: `http://k8s-default-gatewayi-d0c66c7699-299311758.us-east-1.elb.amazonaws.com/webhook/github`
7. The "Webhook secret" field should still hold the value you set in Step 8.2, so leave it alone
8. Click "Save changes"

Your system is now fully connected. GitHub will start sending events to your running service.

---

## 19. Phase 16: The Full End-to-End Test

### 19.1 Install the GitHub App on Your Test Repository

The GitHub App has to be installed on whichever repository you want the bot to review.

1. On GitHub, click your profile picture, then **Settings**
2. In the left sidebar, click **Developer settings**, then **GitHub Apps**
3. Click **Edit** next to `ai-code-reviewer-bot`
4. In the left sidebar, click **Install App**
5. Click **Install** next to your account
6. Choose **Only select repositories**
7. Pick the repository you want to test with (for example, `gmail-ai-analyze`)
8. Click **Install**

### 19.2 Make Sure Every Pod Is Running

Before you test, confirm that every service is up:

```
kubectl get pods
```

Every pod has to say `Running`. If any say `Pending` or `CrashLoopBackOff`, wait a bit and run it again.

### 19.3 Trigger All 5 Pipelines and Wait for Green

Bump the deploy.txt files to set off all the pipelines:

```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer"
```

```
echo 2 > services\gateway\deploy.txt && echo 2 > services\webhook\deploy.txt && echo 2 > services\orchestrator\deploy.txt && echo 2 > services\reviewer\deploy.txt && echo 2 > services\learner\deploy.txt
```

```
git add services\
git commit -m "deploy: trigger all pipelines for end-to-end test"
git push origin main
```

Go to GitHub and open the **Actions** tab. Wait until all 5 pipelines show green checkmarks on all 3 jobs (test, build-and-push, deploy).

Then check once more that every pod is `Running`:
```
kubectl get pods
```

### 19.4 Create a Branch and Open a Pull Request on Your Test Repository

You can do all of this right on the GitHub website, with no CMD needed.

**Step 1: Open your test repository on GitHub**
(for example, `https://github.com/YOUR_USERNAME/gmail-ai-analyze`)

**Step 2: Create a new branch:**
1. Click the branch dropdown at the top left (it shows `main`)
2. Type `test-ai-review` into the box
3. Click **"Create branch: test-ai-review"**

**Step 3: Create a test file:**
1. Make sure you are on the `test-ai-review` branch
2. Click **"Add file"**, then **"Create new file"**
3. Name the file `test_code.py`
4. Paste this code into the editor:
```python
def calculate(a,b):
    password = "admin123"
    result = eval(a+b)
    return result
```
5. Click **"Commit changes"**, then **"Commit directly to test-ai-review branch"**, then **"Commit changes"**

**Step 4: Open a Pull Request:**
1. GitHub shows a yellow banner that says **"Compare & pull request"**, so click it
2. Click **"Create pull request"**

**Step 5: Wait 30 to 60 seconds.**

The bot `ai-code-reviewer-bot` will post its review comments on the PR all by itself.

**Step 6: Check for the comments:**
1. Click the **"Files changed"** tab on the PR
2. You should see inline comments from `ai-code-reviewer-bot` flagging issues such as the hardcoded password and the dangerous use of `eval()`

If those comments show up, your system is working end to end.

---

## 20. Reaching All the Dashboards and UIs

### Grafana: The Metrics Dashboard

**How to get the URL:**
```
kubectl get svc -n monitoring grafana
```
Copy the EXTERNAL-IP and open `http://EXTERNAL-IP` in your browser.

**Login:**
- Username: `admin`
- Password: the one you set in Step 16.6

**What to look at:**
- Pick your imported dashboard from the list of dashboards
- Each service has three panels: request rate, p99 latency, and error rate
- A spike in the error rate means something is broken
- A spike in p99 latency means something is slow
- The dashboard refreshes itself every 30 seconds

---

### Prometheus: The Raw Metrics

**How to open it (it runs locally through a port-forward):**

Open CMD and run:
```
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```
Leave this CMD window open. Then open your browser and go to `http://localhost:9090`.

**Some handy queries to try:**
- Type `up` and click Execute to see which services are being scraped (1 means up, 0 means down)
- Type `http_requests_total` to see the total request count for each service
- Type `rate(http_requests_total[5m])` to see requests per second, averaged over 5 minutes

Press Ctrl+C in the CMD window when you want to stop the port-forward.

---

### LangFuse: AI Call Tracing

**How to open it:**
- EU accounts: https://cloud.langfuse.com
- US accounts: https://us.cloud.langfuse.com

Log in and click the `ai-code-reviewer` project.

**The sections that matter:**
- **Traces:** one entry per PR analyzed, showing the full timeline of all 4 agent calls
- **Generations:** every single OpenAI API call, with its full prompt and response
- **Dashboard:** total token usage, cost over time, and average latency
- **Users:** which repos set off the most analyses

---

### GitHub Actions: The CI/CD Pipeline

**How to open it:**
Go to https://github.com/YOUR_USERNAME/ai-code-reviewer and click the "Actions" tab.

**What to look at:**
- Every row is one deployment, set off by a `git push`
- A green checkmark means it deployed successfully
- A red X means something failed, so click it to see which step broke and read the error

---

### Live Service Logs in CMD

To watch what is happening inside any service right now, open CMD and run:

```
kubectl logs -l app=gateway -f
```

Swap `gateway` for any service name: `webhook`, `orchestrator`, `reviewer`, `learner`, `webhook-worker`, or `learner-worker`.

The `-f` flag means "follow", so it streams new log lines as they come in, like a live feed. Press Ctrl+C to stop.

---

### Kubernetes Pod Status

To see the status of every running container:
```
kubectl get pods
```

To see the CPU and memory used by each pod:
```
kubectl top pods
```

To see full details about one specific pod (handy when debugging a crash):
```
kubectl describe pod POD_NAME
```
Swap POD_NAME for a real pod name from `kubectl get pods`.

---

## 21. How the Weekly Evaluation Works

Every Monday at 9am UTC, GitHub Actions runs `scripts/evaluate.py` on its own. This checks the quality of the AI-generated review comments.

### What It Does, Step by Step:

1. Connects to your RDS PostgreSQL database using the DATABASE_URL secret
2. Pulls the last 50 findings (AI comments) generated across every reviewed PR
3. Builds a dataset out of those findings
4. Runs a RAGAS evaluation, which is a framework for measuring the quality of AI answers
5. Checks two things:
   - **Faithfulness:** are the comments actually grounded in the real code diff?
   - **Answer Relevancy:** are the comments relevant to the question "what issues exist in this code?"
6. Prints the scores
7. If the faithfulness score drops below 0.7 (70%), the job fails and GitHub emails you a notification

### How to See the Evaluation Results:

1. Go to GitHub and open the Actions tab
2. Find the "Evaluate" workflow in the left sidebar
3. Click the most recent run
4. Click the "evaluate" job
5. Expand the "Run evaluation" step
6. You will see output like:
   ```
   Mean faithfulness: 0.84
   ```

### What to Do If It Fails:

A score under 0.7 means the AI is producing low-quality or hallucinated comments.
To dig into it:
1. Go to LangFuse, then Traces
2. Look at the most recent PR analyses
3. Read the AI responses inside each span
4. If the responses are off-topic, the system prompts in `services/orchestrator/graph.py` need some tuning
5. If the responses are broken JSON, the parsing in graph.py needs a fix

---

## 22. Checking That Everything Is Running

Once the full setup is done, use these commands to confirm your system is healthy.

**Confirm every pod is running:**
```
kubectl get pods
```
Each pod should say `Running` and show `1/1` under READY.

**Confirm all the Kubernetes services exist:**
```
kubectl get svc
```

**Check the gateway's health endpoint:**

Open a second CMD window and run:
```
kubectl port-forward svc/gateway 8000:8000
```

Back in the first CMD window, run:
```
curl http://localhost:8000/health
```
Expected response: `{"status":"ok"}`

If `curl` is not available in CMD, open PowerShell and run:
```
Invoke-WebRequest -Uri http://localhost:8000/health
```

**Confirm the database tables were created:**
```
kubectl exec -it deployment/webhook -- python -c "import asyncio, sqlalchemy, os; from sqlalchemy.ext.asyncio import create_async_engine; e = create_async_engine(os.environ['DATABASE_URL']); asyncio.run(e.connect()).__enter__(); print('DB OK')"
```

**Confirm Prometheus is scraping your services:**
1. Open `http://localhost:9090` (after port-forwarding Prometheus as shown in Step 20)
2. Click "Status" in the top menu, then "Targets"
3. All 5 service targets (gateway, webhook, orchestrator, reviewer, learner) should show a green "UP" badge

---

## 23. Troubleshooting

### A Pod Says `Pending` and Never Starts

```
kubectl describe pod POD_NAME
```
Look at the "Events" section at the bottom. The usual causes are:
- `Insufficient cpu` or `Insufficient memory`: your EKS nodes are full. Go to Terraform, raise `desired_size` to 3, and run `terraform apply`.
- `ImagePullBackOff`: the Docker image is not in ECR. Go to GitHub Actions and re-run the `build-and-push` job.

### A Pod Says `CrashLoopBackOff`

```
kubectl logs POD_NAME --previous
```
This shows the logs from just before it crashed. The usual causes are:
- A bad `DATABASE_URL` format: it has to start with `postgresql+asyncpg://`
- A missing environment variable: make sure your `secret.yaml` has every required key and that you applied it
- A Python import error: read the full traceback in the logs

### GitHub Webhook Deliveries Keep Failing

1. Go to GitHub, then Settings, then Developer settings, then GitHub Apps, then ai-code-reviewer-bot
2. Click "Advanced" in the left sidebar
3. Under "Recent Deliveries", you can see every webhook event GitHub tried to send
4. Click a failed delivery to see the HTTP response your server sent back
5. The usual causes are:
   - `404`: the webhook URL path is wrong. It has to end with `/webhook/github`
   - `401`: the webhook secret in your k8s secret does not match the one in the GitHub App
   - Connection refused: the ingress is not ready yet. Check with `kubectl get ingress`

### AI Review Comments Are Not Showing Up on PRs

Check the orchestrator logs:
```
kubectl logs -l app=orchestrator -f
```

Check the reviewer logs:
```
kubectl logs -l app=reviewer -f
```

The usual causes are:
- The OpenAI API key is invalid: check platform.openai.com/usage to see whether any calls are getting through
- The OpenAI account is out of credits: top up at platform.openai.com/billing
- The GitHub App is missing Pull requests: Read and Write permission: check Step 8.3
- The private key format is wrong in the secrets: the key has to keep its line breaks

### Terraform Fails with "Error: VPC limit exceeded"

AWS allows only 5 VPCs per region by default.
Either delete an old VPC in the AWS Console (go to the VPC service, then Your VPCs, then delete one you no longer need),
or ask for a limit increase (go to AWS Support, create a case, choose Service limit increase, then VPC).

### kubectl Commands Fail with "error: You must be logged in to the server"

Your kubeconfig token has expired. Run this again:
```
aws eks update-kubeconfig --name ai-code-reviewer --region us-east-1
```

---

## 24. What It Costs

**Fixed monthly AWS costs (running 24/7):**

| Resource | Specification | Monthly Cost |
|---|---|---|
| EKS Cluster control plane | managed | $72 |
| EKS Worker Nodes | 2x t3.medium | $60 |
| RDS PostgreSQL | db.t3.micro, 20 GB | $15 |
| ElastiCache Redis | cache.t3.micro | $12 |
| Application Load Balancer | for ingress and Grafana | $18 |
| ECR storage | 5 repos, around 500 MB each | $1 |
| S3 bucket | minimal storage | $1 |
| **Total AWS per month** | | **around $179** |

**Usage-based costs:**

| Resource | Cost |
|---|---|
| OpenAI GPT-4o-mini | around $0.15 per 100 PRs reviewed |
| LangFuse | free up to 50,000 events per month |

**To pause the system and stop all charges:**
```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\terraform"
terraform destroy -var="cluster_name=ai-code-reviewer" -var="db_password=YourStrongPassword123!"
```
Type `yes` when it asks. This deletes everything on AWS. You can bring it all back later with `terraform apply`.

**To just scale down (lower the cost while keeping your data):**

Scale the EKS nodes down to 1:
- Open `infra\terraform\main.tf`
- Change `desired_size = 2` to `desired_size = 1`
- Run `terraform apply` again

---

## Quick Reference Card

```
Open Grafana:
  Run: kubectl get svc -n monitoring grafana
  Open: http://EXTERNAL-IP in browser
  Login: admin / your-password

Open Prometheus:
  Run: kubectl port-forward -n monitoring svc/prometheus-server 9090:80
  Open: http://localhost:9090 in browser
  (keep CMD window open while using it)

Open LangFuse:
  EU: https://cloud.langfuse.com
  US: https://us.cloud.langfuse.com

Check all pods:
  kubectl get pods

Watch live logs:
  kubectl logs -l app=gateway -f
  kubectl logs -l app=orchestrator -f
  kubectl logs -l app=reviewer -f

Deploy new code:
  git add .
  git commit -m "your message"
  git push origin main

Scale a service:
  kubectl scale deployment orchestrator --replicas=4

Shut down everything (stop AWS charges):
  cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\terraform"
  terraform destroy -var="cluster_name=ai-code-reviewer" -var="db_password=YourStrongPassword123!" -var="environment=production"
```

---

## 25. Teardown: Cleaning Everything Up

Run this whenever you want to shut everything down and stop all AWS charges.

### 25.1 Destroy All the AWS Infrastructure

```
cd "D:\MAJOR PROJECT KRISH SIR\ai-code-reviewer\infra\terraform"
```

```
terraform destroy -var="cluster_name=ai-code-reviewer" -var="db_password=YourStrongPassword123!" -var="environment=production"
```

Terraform lists everything it plans to delete and then asks:
```
Do you really want to destroy all resources? Enter a value:
```

Type `yes` and press Enter. This takes 10 to 15 minutes.

### 25.2 Check for Leftover Resources

Terraform sometimes misses things that were created outside of it (such as load balancers that Kubernetes made). Check for these and delete them by hand:

**Load Balancers:**
```
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerArn" --output text
```
If anything shows up, delete each one:
```
aws elbv2 delete-load-balancer --load-balancer-arn YOUR_ARN
```

**EBS Volumes** (created by the Prometheus and Grafana PVCs):
```
aws ec2 describe-volumes --filters "Name=status,Values=available" --query "Volumes[].VolumeId" --output text
```
If anything shows up, delete each one:
```
aws ec2 delete-volume --volume-id YOUR_VOLUME_ID
```

**CloudWatch Log Groups:**
```
aws logs describe-log-groups --log-group-name-prefix "/aws/eks" --query "logGroups[].logGroupName" --output text
```
Delete each one:
```
aws logs delete-log-group --log-group-name YOUR_LOG_GROUP
```

### 25.3 Confirm Nothing Is Left

```
aws ec2 describe-instances --query "Reservations[].Instances[].InstanceId" --output text
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerArn" --output text
```

Both should come back empty. At that point you are fully cleaned up with zero ongoing charges.

### 25.4 To Redeploy Later

Just follow this guide again from Phase 8 onward: run `terraform apply`, apply the k8s manifests, and trigger the pipelines. Everything comes back up exactly as it was.

---

## 26. Cleanup: The Right Order to Avoid Errors

Terraform only destroys the things it created itself. The AWS Load Balancer Controller creates load balancers and security groups on the fly while the system runs, and Terraform has no record of those. If they are still around when Terraform tries to delete the VPC, it fails with a `DependencyViolation`.

Always follow this exact order:

### Step 1: Delete the Kubernetes Ingress and Services First

```cmd
kubectl delete ingress --all
kubectl delete svc --all
```

Wait 1 to 2 minutes for AWS to release the load balancers on its own.

### Step 2: Delete the Leftover Security Groups Kubernetes Made

Check which security groups are still around:
```cmd
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=YOUR_VPC_ID" --query "SecurityGroups[*].[GroupId,GroupName]" --output table
```

Delete any security group whose name starts with `k8s-`:
```cmd
aws ec2 delete-security-group --group-id YOUR_SG_ID
```

Do not delete the `default` security group. Terraform takes care of that one.

### Step 3: Delete the Leftover Load Balancers

```cmd
aws elbv2 describe-load-balancers --query "LoadBalancers[*].LoadBalancerArn" --output text
```

Delete each one:
```cmd
aws elbv2 delete-load-balancer --load-balancer-arn YOUR_ARN
```

### Step 4: Run terraform destroy

```cmd
cd infra\terraform
terraform destroy -var="cluster_name=ai-code-reviewer" -var="db_password=YourStrongPassword123!"
```

Type `yes` when it asks. This time it will finish cleanly.

### Why ECR Deletion Just Works

Every ECR repository has `force_delete = true` set in Terraform. That means Terraform will delete them without any errors, even if they still hold Docker images.
