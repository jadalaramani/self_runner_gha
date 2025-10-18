
#  GITHUB ACTIONS — END TO END DOCUMENTATION

---

##  1. **Introduction**

GitHub Actions is a CI/CD platform that lets you automate your build, test, and deployment workflows directly from your GitHub repository.

 **Use Cases:**

* Automate build and test on every commit
* Deploy applications to environments (Dev, Stage, Prod)
* Trigger workflows on push, PR, schedule, or manually

---

##  2. **Basic Components of GitHub Actions**

| Component    | Description                                                        |
| ------------ | ------------------------------------------------------------------ |
| **Workflow** | A YAML file that defines the automation process                    |
| **Event**    | Trigger for the workflow (push, pull request, schedule, etc.)      |
| **Job**      | A set of steps run on the same runner                              |
| **Step**     | Individual tasks within a job                                      |
| **Runner**   | The server/machine that runs the job                               |
| **Action**   | Predefined reusable units (e.g., checkout code, set up node, etc.) |

---

##  3. **Folder Structure**

```
📂 your-repo
 ├── .github
 │    └── workflows
 │         └── ci-cd.yml
 ├── src/
 ├── Dockerfile
 ├── k8s_deployment.yaml
 └── ...
```

> All workflows must be inside `.github/workflows/` directory.

---

##  4. **Common Triggers**

| Trigger             | Usage                              |
| ------------------- | ---------------------------------- |
| `push`              | Run pipeline when code is pushed   |
| `pull_request`      | Run pipeline on PR creation/update |
| `workflow_dispatch` | Manual trigger                     |
| `schedule`          | Run on cron schedule               |

**Example:**

```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
```
##  How GitHub Actions Works Internally
1. Event Trigger — A code push, PR, schedule, or manual dispatch triggers the workflow.
2. GitHub Actions Runner — A virtual machine or container executes the jobs.
3. Checkout & Build — The runner pulls your repo, installs dependencies, and executes steps.
4. Artifacts & Caching — Optional steps to store build output or cache dependencies.
5. Deployment — The runner interacts with cloud services (AWS, GCP, Azure, etc.) to deploy.
6. Logs & Notifications — Every step’s output is stored and viewable in the GitHub UI.


## Types of GitHub Runners
#### 1. GitHub-Hosted Runners (Default)

Ready-to-use VM (Ubuntu, Windows, macOS)

Auto-provisioned and destroyed after each run

Good for most use cases
```
runs-on: ubuntu-latest
```
#### 2. Self-Hosted Runners

You host your own machine (on-prem or cloud)

Can have custom software installed

Useful for secure networks or large builds
```
runs-on: self-hosted
```

Note: You can label self-hosted runners and target them in jobs:

```
runs-on: [self-hosted, linux, x64]
```
---

##  5. **Basic CI/CD Workflow Example**

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Dependencies
        run: npm install

      - name: Run Tests
        run: npm test

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Build Docker Image
        run: docker build -t my-app:${{ github.run_number }} .

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Push to ECR
        run: |
          docker tag my-app:${{ github.run_number }} ${{ secrets.AWS_ECR_URI }}:latest
          docker push ${{ secrets.AWS_ECR_URI }}:latest

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name my-cluster --region ap-south-1
          kubectl apply -f k8s_deployment.yaml
```

---

##  6. **Important GitHub Actions Used**

| Action                                  | Purpose                      |
| --------------------------------------- | ---------------------------- |
| `actions/checkout`                      | Checkout source code         |
| `actions/setup-node`                    | Set up Node.js               |
| `aws-actions/configure-aws-credentials` | Authenticate with AWS        |
| `aws-actions/amazon-ecr-login`          | Login to ECR                 |
| `docker/build-push-action`              | Build and push Docker images |
| `kubectl` (shell)                       | Deploy to Kubernetes         |

---

## 🧑‍💻 7. **Using Secrets & Variables**

* Store sensitive values in **Repository → Settings → Secrets and variables → Actions**
* Example:

  * `AWS_ACCESS_KEY_ID`
  * `AWS_SECRET_ACCESS_KEY`
  * `AWS_ECR_URI`
  * `K8S_CLUSTER_NAME`

**Usage in YAML:**

```yaml
with:
  aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
  aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

##  8. **Job Dependencies**

You can control order of jobs using `needs:`

```yaml
jobs:
  build:
    runs-on: ubuntu-latest

  deploy:
    runs-on: ubuntu-latest
    needs: build
```

This ensures `deploy` runs **only after** `build` succeeds.

---

##  9. **Matrix Build Example**

For testing multiple versions/platforms:

```yaml
strategy:
  matrix:
    node: [14, 16, 18]
steps:
  - name: Setup Node
    uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node }}
```

---

##  10. **Schedule Triggers (Cron)**

```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # Every day at 2 AM UTC
```

---

##  11. **Manual Trigger**

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Select environment"
        required: true
        default: "dev"
```

You can run it from **Actions tab → Run workflow**.

---

## 🧭 12. **Best Practices**

* Use `needs` to manage job dependencies
* Use reusable workflows for DRY pipelines
* Use secrets for credentials (never hardcode)
* Use tagging/versioning for Docker images
* Create environment protection rules for production
* Use separate pipelines for build and deploy

---

##  13. **Reusable Workflow Example**

You can create a reusable workflow to call from other repos:

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build
on:
  workflow_call:
    inputs:
      image_tag:
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Build using ${{ inputs.image_tag }}"
```

Calling it:

```yaml
jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      image_tag: "v1.0"
```

---

##  14. **Real-Time CI/CD Flow Example**

 **Scenario:** Deploy a Node.js application to EKS when code is pushed to `main`.

1. Developer pushes code to `main`
2. GitHub Action triggers build
3. Application builds & tests run
4. Docker image built & pushed to ECR
5. Kubernetes manifests updated with new image tag
6. Deployment applied to EKS cluster
7. Pipeline sends status to GitHub

---

##  15. **Sample Block Diagram**

```
+-------------+        +---------------+       +-------------------+       +-----------------+
| Developer   |  Push  | GitHub Repo   |  Run  | GitHub Actions    |  Deploy | AWS ECR + EKS  |
| Commit Code | -----> | main branch   |-----> | Build/Test/Deploy |------->| App Deployment |
+-------------+        +---------------+       +-------------------+       +-----------------+
```

---

##  16. **Deployment Rollback**

In case of failure:

* Use previous Docker image tag
* Revert commit or deployment
* Manual workflow trigger for rollback

```bash
kubectl rollout undo deployment/my-app
```

---

##  17. **Monitoring & Logs**

* Check logs in **Actions tab**
* Job & step level logs available
* Use `::warning::` or `::error::` to highlight errors

---

##  18. **Artifacts & Caching**

* Cache dependencies for faster builds:

```yaml
- name: Cache npm
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

* Upload build artifacts:

```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v4
  with:
    name: build
    path: dist/
```

---

##  19. **Environment Protection**

* Configure Environments in GitHub → Settings → Environments
* Add rules like required reviewers or secrets per environment (Dev/Stage/Prod)

---

##  20. **Security Tips**

* Use GitHub Secrets, not plain text
* Use least privilege AWS IAM roles
* Rotate secrets periodically
* Protect main branch
