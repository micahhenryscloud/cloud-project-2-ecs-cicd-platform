# Design Decisions

This document explains the key design choices made in the ECS Fargate CI/CD project and the reasoning behind them.

## 1. Docker for Application Packaging

Docker was used to package the application into a portable container image.

This means the application can run consistently across different environments because the application files, web server and runtime environment are defined inside the image.

The basic flow is:

```text
Application Files
   ↓
Dockerfile
   ↓
Docker Image
   ↓
Docker Container
```

This avoids relying on manually configuring servers one by one.

## 2. Nginx as the Container Web Server

Nginx was used as the base image because it is lightweight and commonly used for serving static web content.

The Dockerfile copies the application page into the Nginx web root:

```text
index.html
   ↓
/usr/share/nginx/html/index.html
```

This created a simple, clear containerised web application suitable for learning ECS deployment.

## 3. Amazon ECR for Container Image Storage

Amazon ECR was used as the private container registry.

This allows Docker images to be stored inside AWS and then pulled by AWS container services such as ECS.

The flow is:

```text
Local Docker Image
   ↓
Amazon ECR
   ↓
ECS Fargate
```

Using ECR keeps the image close to the AWS deployment environment and avoids relying on a public registry.

## 4. ECS Fargate Instead of EC2-Based Containers

ECS Fargate was used so that the container could run without manually managing EC2 instances.

With EC2, I would need to manage:

- Server provisioning
- Operating system updates
- Capacity management
- Instance scaling
- Docker runtime installation

With Fargate, AWS manages the underlying compute layer.

This allows the focus to remain on:

- Task definitions
- Services
- Containers
- Networking
- Deployment

## 5. ECS Task Definition for Container Configuration

The ECS task definition was used to describe how the container should run.

It defines:

- The container image
- Container port
- CPU and memory settings
- Runtime configuration

This is similar to a blueprint for the container workload.

## 6. ECS Service for Long-Running Application Management

An ECS service was used to keep the container running.

The service is responsible for maintaining the desired number of running tasks.

This means if a task stops or fails, ECS can launch a replacement.

The model is:

```text
Desired Task Count
   ↓
ECS Service
   ↓
Running Fargate Task
```

## 7. Public Fargate Task for Initial Testing

For the first ECS deployment, the task was placed in public subnets with a public IP.

This made it easier to validate that the container was running and accessible through a browser.

For a more production-style setup, the next improvement would be to place tasks in private subnets behind an Application Load Balancer.

## 8. GitHub Actions for CI/CD

GitHub Actions was used to automate the deployment process.

Instead of manually running Docker build, Docker push and ECS deployment commands, the workflow performs those steps after code is pushed.

The CI/CD flow is:

```text
Git Push
   ↓
GitHub Actions
   ↓
Docker Build
   ↓
Push Image to ECR
   ↓
Force ECS Deployment
```

This demonstrates automated application delivery.

## 9. GitHub Secrets for AWS Credentials

AWS credentials were stored as GitHub repository secrets instead of being hardcoded into the workflow file.

This keeps sensitive values out of the repository.

The workflow reads secrets such as:

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_ACCOUNT_ID

This is a safer approach than committing credentials directly into code.

## 10. Force New ECS Deployment

The GitHub Actions workflow forces a new ECS deployment after pushing the updated image to ECR.

This tells ECS to replace the running task with a new task based on the latest image.

This demonstrates the deployment automation pattern:

```text
New Image
   ↓
ECR
   ↓
ECS Service Update
   ↓
New Running Task
```

## Summary

The project was designed around four main principles:

- Package the application consistently using Docker
- Store container images securely in Amazon ECR
- Run containers without managing servers using ECS Fargate
- Automate deployments using GitHub Actions

This project demonstrates a simple but realistic cloud-native deployment workflow.