# Deployment Process

## Overview
This document describes the deployment process for the Dev and QA environments.

## Branching Base
- **Staging** is the common shared base branch for all developers.
- Each **feature branch is cut from Staging**.

---

# Deployment Process to Dev Environment

## Step 1: Create Feature Branch from Staging
Create or switch to your feature branch from the Staging branch.

```bash
git checkout staging
git pull origin staging
git checkout -b feature/your-feature-name
````

## Step 2: Push Changes to Feature Branch

Commit your changes and push them to the remote feature branch.

```bash
git add .
git commit -m "Your meaningful commit message"
git push origin feature/your-feature-name
```

## Step 3: Raise Pull Request (Feature → Staging)

Create a Pull Request (PR) from:

* **Source:** Feature branch
* **Target:** Staging branch

**Review requirement:** No review required.

## Step 4: Merge Pull Request (Feature → Staging)

Once ready, merge the PR.

Changes are now merged into the **Staging** branch.

## Step 5: Raise Pull Request (Staging → Dev)

Create another Pull Request (PR) from:

* **Source:** Staging branch
* **Target:** Dev branch

**Review requirement:**

* GitHub Copilot manual code review
* No teammate review required

This is because the **Dev environment is used for fast testing and quick feature movement**.

## Step 6: Merge Pull Request (Staging → Dev)

Once approved, merge the PR into the **Dev** branch.

## Step 7: Dev Deployment

As soon as the commit/PR is merged into the **Dev** branch:

1. GitHub Actions builds the Docker image from the **Dev branch** code
2. The image is pushed to **AWS ECR**
3. Using **AWS SSM**, the image is deployed to the **Dev server**

## Where the Code Lives in Dev

* Code for Dev lives in the **Dev branch** , Server only has ECR docker image.

---

# Deployment Process to QA Environment

## Step 1: Raise Pull Request (Dev → QA)

Create a Pull Request (PR) from:

* **Source:** Dev branch
* **Target:** QA branch

**Review requirement:**

* GitHub Copilot automatic code review
* Teammate review required
* Human in loop

## Step 2: Merge Pull Request (Dev → QA)

Once approved, merge the PR into the **QA** branch.

## Step 3: QA Deployment

There is **no automatic CI/CD** for QA.

Deployment is done manually:

1. Code lives in the **QA branch**
2. **Shashank** manually copies the code to the **Indegene QA server**
3. `docker-compose_qa.yaml` rebuilds the image
4. The rebuilt image is deployed on the QA server

## Where the Code Lives in QA

* Code for QA lives in the **QA branch** and on QA server (because docker-compose makes image)

---

# Server Requirement Note
No additional server-side requirement is needed.

Everything is external, for example: **AWS RDS** for database

