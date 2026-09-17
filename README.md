# Nginx ECS CI/CD Deployment

This project demonstrates a complete **CI/CD pipeline** for deploying a containerized Nginx application to **Amazon ECS** using **GitHub Actions**, **Docker**, and **Amazon ECR**.

The pipeline automatically builds a Docker image, pushes it to Amazon ECR, updates the ECS task definition, and deploys the new application version to an ECS service whenever changes are pushed to the `main` branch.

---

## 🚀 CI/CD Architecture

```text
                    Developer
                        |
                        | git push
                        v
                +----------------+
                |    GitHub      |
                |   Repository   |
                +-------+--------+
                        |
                        | Push to main
                        v
              +---------------------+
              |   GitHub Actions    |
              +----------+----------+
                         |
                         | 1. Checkout Code
                         |
                         | 2. Authenticate to AWS
                         |    using OIDC
                         v
              +---------------------+
              |    Docker Build     |
              +----------+----------+
                         |
                         | Docker Image
                         v
              +---------------------+
              |   Amazon ECR        |
              |  Container Registry |
              +----------+----------+
                         |
                         | New Image
                         v
              +---------------------+
              | ECS Task Definition |
              |   New Revision      |
              +----------+----------+
                         |
                         | Deploy
                         v
              +---------------------+
              |    Amazon ECS       |
              |    ECS Service      |
              +----------+----------+
                         |
                         v
                 +---------------+
                 | Nginx Container|
                 +---------------+
```

---

# 🔄 Complete Deployment Flow

The pipeline follows these steps:

### 1. Developer pushes code

A developer makes changes locally and pushes them to the `main` branch:

```bash
git add .
git commit -m "Update application"
git push origin main
```

This push automatically triggers the GitHub Actions workflow.

---

### 2. GitHub Actions starts

The workflow is triggered by:

```yaml
on:
  push:
    branches:
      - main
```

GitHub Actions creates an Ubuntu runner and starts the deployment process.

---

### 3. GitHub Actions authenticates with AWS

The pipeline uses **AWS IAM OIDC** to authenticate without storing long-lived AWS access keys in GitHub.

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v6.2.4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ${{ env.AWS_REGION }}
    audience: sts.amazonaws.com
```

The GitHub repository requires the following secret:

```text
AWS_ROLE_ARN
```

---

### 4. GitHub Actions logs in to Amazon ECR

The workflow authenticates Docker with Amazon ECR:

```yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2
```

This allows the GitHub Actions runner to push the Docker image to the ECR repository.

---

### 5. Docker image is built

The application is packaged into a Docker image:

```yaml
- name: Build Docker Image
  run: |
    docker build \
      -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
      .
```

The image is tagged using the GitHub commit SHA.

For example:

```text
my-repo:a8f3c92...
```

Using the commit SHA makes every image traceable to a specific version of the source code.

---

### 6. Docker image is pushed to Amazon ECR

The newly built image is pushed to ECR:

```yaml
- name: Push Docker Image to ECR
  run: |
    docker push \
      ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
```

The image is stored in:

```text
Amazon ECR
    |
    └── my-repo
          |
          └── <commit-sha>
```

---

### 7. Current ECS task definition is retrieved

The pipeline retrieves the current ECS task definition:

```yaml
- name: Get Current ECS Task Definition
  run: |
    aws ecs describe-task-definition \
      --task-definition ${{ env.ECS_TASK_DEFINITION }} \
      --region ${{ env.AWS_REGION }} \
      --query taskDefinition \
      --output json \
      --no-cli-pager > task-definition.json
```

This gives the deployment process the current ECS configuration.

---

### 8. ECS task definition is updated

The workflow replaces the old Docker image with the newly pushed ECR image:

```yaml
- name: Update Image in Task Definition
  id: task-def
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: task-definition.json
    container-name: ${{ env.CONTAINER_NAME }}
    image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
```

This creates a new ECS task definition revision containing the new image.

---

### 9. New version is deployed to ECS

The updated task definition is deployed to the ECS service:

```yaml
- name: Deploy to ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@v2
  with:
    task-definition: ${{ steps.task-def.outputs.task-definition }}
    service: ${{ env.ECS_SERVICE }}
    cluster: ${{ env.ECS_CLUSTER }}
    wait-for-service-stability: true
```

ECS then starts tasks using the new Docker image.

The workflow waits for the ECS service to become stable before reporting success.

---

# 🏗️ AWS Resources

| Resource            | Value           |
| ------------------- | --------------- |
| AWS Region          | `us-east-1`     |
| ECR Repository      | `my-repo`       |
| ECS Cluster         | `nginx-cluster` |
| ECS Service         | `nginx-service` |
| ECS Task Definition | `nginx-app`     |
| Container Name      | `nginx`         |

---

# 📁 Project Structure

```text
.
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── Dockerfile
├── README.md
└── ...
```

---

# 🐳 Docker

The application is containerized using Docker and Nginx.

Example Dockerfile:

```dockerfile
FROM nginx:latest

COPY . /usr/share/nginx/html

EXPOSE 80
```

The Docker image is built automatically by GitHub Actions.

There is no need to manually build or push the image from the developer's local machine.

---

# 🔐 AWS Security

The GitHub Actions workflow uses **OIDC authentication** to assume an AWS IAM role.

This avoids storing permanent AWS access keys in GitHub Secrets.

The only required GitHub secret is:

```text
AWS_ROLE_ARN
```

The IAM role should have the permissions required to:

* Authenticate with ECR
* Push images to ECR
* Read the ECS task definition
* Register a new ECS task definition revision
* Update the ECS service
* Describe ECS resources

No AWS access keys or secret keys should be committed to the repository.

---

# ⚙️ GitHub Actions Workflow

The complete workflow is stored in:

```text
.github/workflows/deploy.yml
```

The pipeline can be summarized as:

```text
Git Push
   ↓
GitHub Actions
   ↓
AWS OIDC Authentication
   ↓
Checkout Code
   ↓
Docker Build
   ↓
ECR Login
   ↓
Push Image to ECR
   ↓
Retrieve ECS Task Definition
   ↓
Update Container Image
   ↓
Create New Task Definition Revision
   ↓
Deploy to ECS Service
   ↓
Wait for Service Stability
   ↓
Deployment Complete
```

---

# 🚀 Deployment

To deploy a new version:

```bash
git add .
git commit -m "Update application"
git push origin main
```

The deployment then happens automatically.

Monitor the pipeline from:

```text
GitHub
  → Actions
    → Build and Deploy to ECS
```

---

# 🔎 Verify ECS Deployment

Check the ECS service:

```bash
aws ecs describe-services \
  --cluster nginx-cluster \
  --services nginx-service \
  --region us-east-1
```

List the running ECS tasks:

```bash
aws ecs list-tasks \
  --cluster nginx-cluster \
  --service-name nginx-service \
  --region us-east-1
```

Check the task definition revisions:

```bash
aws ecs list-task-definitions \
  --family-prefix nginx-app \
  --region us-east-1
```

---

# 📌 Key CI/CD Concepts Demonstrated

This project demonstrates:

* Git-based deployments
* GitHub Actions CI/CD
* Docker image creation
* Amazon ECR container registry
* AWS IAM OIDC authentication
* ECS task definitions
* ECS service deployments
* Immutable image tagging using Git commit SHA
* Automated deployment verification
* Infrastructure and application integration

---

# 👩‍💻 Author

**Sorelle Meliedje**

Cloud / DevOps Project
