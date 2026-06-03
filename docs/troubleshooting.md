# Troubleshooting Notes

This document records real issues encountered while building the ECS Fargate CI/CD project, how they were diagnosed, and how they were fixed.

## Issue 1: ECR login failed with 400 Bad Request

### Problem

Docker login to Amazon ECR failed with an error similar to:

```text
login attempt to https://816625019234.dkr.ecr.eu-west-2.amazonaws.com/v2/ failed with status: 400 Bad Request
```

### Investigation

The ECR repository existed, and AWS CLI access was working, but Docker was not successfully authenticating to the ECR registry.

I verified the repository using:

```powershell
aws ecr describe-repositories --repository-names micah-docker-app --region eu-west-2
```

### Root Cause

The issue was related to Docker/ECR authentication from PowerShell.

The standard pipe-based login command was not working reliably.

### Fix

I logged out of the registry, retrieved the ECR login password, stored it in a variable, and then logged Docker into ECR manually.

```powershell
docker logout 816625019234.dkr.ecr.eu-west-2.amazonaws.com

$ECR_PASSWORD = aws ecr get-login-password --region eu-west-2

docker login --username AWS --password $ECR_PASSWORD 816625019234.dkr.ecr.eu-west-2.amazonaws.com
```

### Lesson Learned

When ECR login fails, verify AWS CLI identity, confirm the repository exists, and isolate whether the issue is AWS credentials or Docker registry authentication.

---

## Issue 2: Docker push failed because the tag did not exist

### Problem

Docker returned an error similar to:

```text
tag does not exist
```

when attempting to push the image to ECR.

### Investigation

I checked the local Docker images using:

```powershell
docker images
```

The local image existed as:

```text
micah-docker-app:latest
```

but the ECR-tagged image did not exist locally.

### Root Cause

The image had not been tagged with the full ECR repository URI before pushing.

Docker requires the image tag to match the remote repository destination.

### Fix

I tagged the image correctly:

```powershell
docker tag micah-docker-app:latest 816625019234.dkr.ecr.eu-west-2.amazonaws.com/micah-docker-app:latest
```

Then pushed it:

```powershell
docker push 816625019234.dkr.ecr.eu-west-2.amazonaws.com/micah-docker-app:latest
```

### Lesson Learned

Before pushing to ECR, the local image must be tagged with the full ECR repository URI.

---

## Issue 3: Docker push failed with “no basic auth credentials”

### Problem

Docker returned:

```text
no basic auth credentials
```

when trying to push the image to ECR.

### Investigation

The image had been tagged correctly, but Docker was not authenticated with ECR.

### Root Cause

Docker was either not logged in to ECR or the previous login session had failed or expired.

### Fix

I re-authenticated Docker with ECR and then pushed the image again.

```powershell
aws ecr get-login-password --region eu-west-2 | docker login --username AWS --password-stdin 816625019234.dkr.ecr.eu-west-2.amazonaws.com
```

After successful login, the image push worked.

### Lesson Learned

ECR push failures are often caused by authentication issues. Confirm the repository exists, confirm AWS CLI credentials, and re-run Docker login before pushing.

---

## Issue 4: ECS cluster creation failed because of service-linked role issue

### Problem

Creating the ECS cluster initially failed with an error about ECS being unable to assume the service-linked role.

The error mentioned:

```text
Unable to assume the service linked role
```

### Investigation

ECS requires an AWS service-linked role so it can manage resources on behalf of the account.

I attempted to create the service-linked role using:

```powershell
aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com
```

### Root Cause

The ECS service-linked role already existed, but ECS was not initially able to use it at the time the cluster was created.

### Fix

After confirming the role existed, I retried creating the ECS cluster and the cluster was created successfully.

### Lesson Learned

Some AWS services require service-linked roles. If a service-linked role error appears, verify whether the role exists and retry the operation after AWS has registered the role properly.

---

## Issue 5: ECS task definition resource settings were unclear

### Problem

While creating the ECS task definition, the console showed container-level CPU and memory fields, which caused confusion about whether those values were required.

### Investigation

The task definition wizard separates task-level resources from container-level limits.

For a simple Nginx container, container-level CPU and memory overrides were not necessary.

### Root Cause

The ECS console UI presents multiple resource configuration areas, which can make it unclear whether values should be set at the task level or container level.

### Fix

I kept the container-level resource limits simple and focused on defining the essential task settings:

- Container image
- Container port 80
- Fargate launch type
- Basic CPU and memory allocation

### Lesson Learned

For simple ECS workloads, start with minimal task configuration. Container-level tuning can be added later when resource requirements are better understood.

---

## Issue 6: GitHub Actions needed AWS secrets

### Problem

The GitHub Actions workflow required AWS credentials to build the image, push it to ECR, and trigger an ECS deployment.

### Investigation

The workflow referenced GitHub secrets such as:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_ACCOUNT_ID
```

### Root Cause

GitHub Actions runs outside AWS, so it needs a secure method to authenticate with AWS.

Hardcoding credentials into the workflow would be insecure.

### Fix

I added the required values as GitHub repository secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_ACCOUNT_ID
```

The workflow then accessed them securely using the GitHub Actions secrets context.

### Lesson Learned

CI/CD workflows should use secrets or identity federation for cloud credentials. Credentials should never be committed directly into source code.

---

## Issue 7: GitHub Actions deployment succeeded after push

### Problem

After creating the GitHub Actions workflow, I needed to confirm whether the pipeline actually deployed the application.

### Investigation

I pushed the workflow and watched the GitHub Actions run.

The workflow completed successfully.

### Root Cause

There was no failure at this stage. The purpose was validation.

### Fix

The successful workflow confirmed that the CI/CD pipeline could:

- Checkout the repository
- Configure AWS credentials
- Authenticate with ECR
- Build the Docker image
- Push the image to ECR
- Force a new ECS deployment

### Lesson Learned

A deployment pipeline should always be validated end-to-end. A green GitHub Actions run proves the automation path works.

---

## Summary

The main troubleshooting themes in this project were:

- Authenticating Docker with Amazon ECR
- Correctly tagging Docker images before pushing
- Understanding ECS service-linked roles
- Configuring ECS task definitions
- Managing AWS credentials securely in GitHub Actions
- Validating CI/CD deployment success

These issues helped turn the project from a simple container deployment into a realistic CI/CD troubleshooting exercise.