# MyECS-CI-CD-pipeline

This project uses GitHub Actions to build a Docker image, push it
to Amazon ECR, and deploy it to Amazon ECS Fargate.

### Triggers

The workflow runs:
- Automatically when commits are pushed to `main`.
- Manually through GitHub Actions using **Run workflow**.

### Deployment flow

```text
Push to main / Manual trigger
              |
              v
        GitHub Actions
              |
              v
    Build Docker image once
              |
              v
      Tag with commit SHA
              |
              v
     Push image to Amazon ECR
              |
              v
  Update ECS task definition
     with the same image URI
              |
              v
  Register and deploy new revision
              |
              v
    Wait for ECS service stability
              |
              v
   Application accessible through ALB
```

### Jobs

1. **build**
   - Checks out the repository.
   - Authenticates to AWS using GitHub OIDC.
   - Builds the Docker image from the root Dockerfile.
   - Tags the image with the Git commit SHA.
   - Pushes the image to ECR.
   - Passes the image URI to the deployment job.

2. **deploy**
   - Runs only after the build job succeeds.
   - Downloads the service's current ECS task definition.
   - Updates the `nginx` container image URI.
   - Registers a new task definition revision and updates the service.
   - Waits for service stability and writes a deployment summary.

### Required GitHub configuration

Under Settings → Secrets and variables → Actions, configure:

| Type | Name | Description |
|------|------|-------------|
| Secret | AWS_ROLE_ARN | IAM role assumed through GitHub OIDC |
| Variable | AWS_REGION | AWS region, such as us-east-1 |
| Variable | ECR_REPOSITORY | Existing ECR repository name |

### Prerequisites

- A Dockerfile and index.html in the repository root.
- An existing ECR repository.
- ECS cluster: `nginx-demo-cluster`.
- ECS service: `nginx-demo-service`.
- Container name in the task definition: `nginx`.
- An ALB configured to route traffic to the service.
- An IAM role with a GitHub OIDC trust policy restricted to the
  intended repository and deployment branch.
- IAM permissions to push images to ECR, describe ECS services and
  task definitions, register task definitions, update the service,
  and pass the task roles.

### Verify a deployment

1. Check that both GitHub Actions jobs completed successfully.
2. Review the deployment summary for the deployed image URI.
3. Open the ALB DNS name using HTTP.
4. Confirm that the expected website changes are visible.

### Infrastructure ownership

Terraform manages the infrastructure and autoscaling configuration.
GitHub Actions manages application task definition deployments.
The ECS service lifecycle ignores `desired_count` and `task_definition`
so Terraform does not reset autoscaling or revert application releases.