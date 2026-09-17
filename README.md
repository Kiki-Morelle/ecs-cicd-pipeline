# ecs-cicd-pipeline

# Nginx ECS Deployment with GitHub Actions

This project demonstrates how to build, containerize, and deploy an Nginx application to **Amazon ECS** using **Amazon ECR** and **GitHub Actions**.

The deployment process is fully automated using GitHub Actions and AWS IAM OIDC authentication.

## Architecture

```text
Developer
    |
    | git push
    v
GitHub Repository
    |
    v
GitHub Actions
    |
    | Build Docker Image
    v
Docker Image
    |
    | Push
    v
Amazon ECR
    |
    | New Image
    v
Amazon ECS
    |
    v
ECS Service
    |
    v
Nginx Container
```

## Technologies Used

* Docker
* Nginx
* GitHub Actions
* Amazon ECR
* Amazon ECS
* AWS IAM
* AWS OIDC

## Project Structure

```text
.
├── .github/
│   └── workflows/
│       └── deploy.yml
├── Dockerfile
├── README.md
└── ...
```

## Dockerfile

The application is packaged into a Docker image using Nginx.

Example:

```dockerfile
FROM nginx:latest

COPY . /usr/share/nginx/html

EXPOSE 80
```

## AWS Resources

The deployment uses the following AWS resources:

| Resource            | Name            |
| ------------------- | --------------- |
| AWS Region          | `us-east-1`     |
| ECR Repository      | `my-repo`       |
| ECS Cluster         | `nginx-cluster` |
| ECS Service         | `nginx-service` |
| ECS Task Definition | `nginx-app`     |
| Container           | `nginx`         |

## CI/CD Pipeline

The GitHub Actions workflow is triggered whenever code is pushed to the `main` branch.

```yaml
on:
  push:
    branches:
      - main
```

The pipeline performs the following steps:

1. Checkout the source code.
2. Authenticate to AWS using GitHub OIDC.
3. Login to Amazon ECR.
4. Build the Docker image.
5. Tag the image with the GitHub commit SHA.
6. Push the image to Amazon ECR.
7. Retrieve the current ECS task definition.
8. Replace the container image with the newly built image.
9. Deploy the updated task definition to ECS.
10. Wait for the ECS service to become stable.

## Image Tagging

Each Docker image is tagged using the GitHub commit SHA:

```text
<aws-account>.dkr.ecr.us-east-1.amazonaws.com/my-repo:<commit-sha>
```

This makes each deployment traceable to a specific Git commit.

## AWS Authentication

The workflow uses **GitHub Actions OIDC** instead of storing long-lived AWS access keys in GitHub.

The workflow requires the following GitHub secret:

```text
AWS_ROLE_ARN
```

The secret contains the ARN of the IAM role that GitHub Actions is allowed to assume.

Example:

```text
arn:aws:iam::<ACCOUNT_ID>:role/GitHubActions-ECS-Role
```

Do not put AWS access keys, secret keys, passwords, or other sensitive credentials in this repository.

## GitHub Actions Workflow

The workflow is located at:

```text
.github/workflows/deploy.yml
```

The workflow builds and pushes the image to ECR:

```yaml
- name: Build Docker Image
  run: |
    docker build \
      -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
      .
```

Then it pushes the image:

```yaml
- name: Push Docker Image to ECR
  run: |
    docker push \
      ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
```

Finally, it deploys the new image to ECS:

```yaml
- name: Deploy to ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@v2
```

## Deployment

To deploy a new version:

```bash
git add .
git commit -m "Update application"
git push origin main
```

Pushing to `main` automatically starts the GitHub Actions workflow.

You can monitor the deployment from:

```text
GitHub → Actions → Build and Deploy to ECS
```

## Verify the Deployment

After the workflow completes successfully, verify the ECS service:

```bash
aws ecs describe-services \
  --cluster nginx-cluster \
  --services nginx-service \
  --region us-east-1
```

You can also check the running tasks:

```bash
aws ecs list-tasks \
  --cluster nginx-cluster \
  --service-name nginx-service \
  --region us-east-1
```

## Deployment Flow

```text
git push
    ↓
GitHub Actions
    ↓
AWS OIDC Authentication
    ↓
Docker Build
    ↓
Amazon ECR
    ↓
ECS Task Definition Revision
    ↓
ECS Service Update
    ↓
New Nginx Container
```

## Benefits

This setup provides:

* Automated deployments
* No manual Docker image uploads
* No long-lived AWS credentials in GitHub
* Versioned Docker images
* Repeatable ECS deployments
* Automatic ECS service stability checks
* Clear traceability between Git commits and deployments

## Author

**Sorelle Meliedje**

Cloud / DevOps Project
