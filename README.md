# ecs-cicd-pipeline

A complete CI/CD pipeline that builds a Docker image on every push to `main`, pushes it to Amazon ECR, and deploys it to Amazon ECS (Fargate) behind an Application Load Balancer — all provisioned with Terraform and authenticated via GitHub OIDC (no long-lived AWS credentials).

## Architecture

Developer pushes to `main`
→ GitHub Actions authenticates to AWS via OIDC (no stored secrets)
→ Builds Docker image
→ Pushes image to Amazon ECR (`my-repo`)
→ Forces a new ECS deployment
→ Amazon ECS (Fargate) pulls the latest image and runs it
→ Traffic flows through an Application Load Balancer to users

## Tech Stack

- **CI/CD**: GitHub Actions
- **Container Registry**: Amazon ECR
- **Compute**: Amazon ECS on AWS Fargate
- **Load Balancing**: Application Load Balancer (ALB)
- **Infrastructure as Code**: Terraform
- **Auth**: GitHub OIDC → AWS IAM (no static AWS keys stored in GitHub)

## Repository Structure

- `.github/workflows/cicd.yml` — GitHub Actions pipeline
- `infra/main.tf` — all AWS infrastructure (VPC, ECS, ALB, IAM)
- `Dockerfile`
- `index.html`
- `README.md`

## Infrastructure (Terraform)

`infra/main.tf` provisions:
- A dedicated VPC with public + private subnets across 2 AZs
- Internet Gateway + NAT Gateway (for private ECS tasks to reach ECR/internet)
- Security groups (ALB accepts public HTTP; ECS tasks only accept traffic from the ALB)
- ECS Cluster (`nginx-cluster`) and Fargate Service (`nginx-service`)
- Application Load Balancer, target group, and listener
- ECS service autoscaling (min 2, max 6 tasks, target 50% CPU)
- IAM execution role for ECS tasks

To provision or update infrastructure:

    cd infra
    terraform init
    terraform plan
    terraform apply

## CI/CD Pipeline

On every push to `main`, `.github/workflows/cicd.yml`:

1. Checks out the code
2. Authenticates to AWS via OIDC (`aws-actions/configure-aws-credentials`) — no long-lived AWS keys stored in GitHub
3. Logs in to Amazon ECR
4. Builds the Docker image
5. Tags and pushes the image to ECR (`my-repo:latest`)
6. Forces a new ECS deployment so the running service picks up the fresh image

### OIDC Trust Policy

The GitHub Actions IAM role (`github-action-role`) trusts GitHub's OIDC provider, scoped to this repository. Note: GitHub Actions may issue OIDC tokens using **immutable subject claims** (embedding numeric account/repo IDs, e.g. `repo:Egide16@289290698/ecs-cicd-pipeline@1372323826:ref:refs/heads/main`) rather than the classic name-based format. The trust policy's `sub` condition is written to match this format.

## Accessing the App

After `terraform apply`, get the public URL:

    terraform output nginx_url

## Local Development

Build and run the container locally:

    docker build -t my-repo .
    docker run -p 8080:80 my-repo

Visit `http://localhost:8080`.