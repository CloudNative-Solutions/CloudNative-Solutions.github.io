---
title: "AWS OpsWorks to AWS Fargate Migration"
meta_title: "Case Studies"
image: /images/case-studies/opsworks-to-fargate.webp
description: "Learn how Cloud Native Solutions migrated a Django-based infrastructure from AWS OpsWorks to AWS Fargate, improving scalability, deployment, and management."
---
## Introduction

This case study highlights how Cloud Native Solutions successfully migrated for our customer its existing Django-based infrastructure managed by AWS Opsworks (Chef) to a containerized infrastructure on AWS Fargate.

By leveraging Gitlab-CI, Terraform, and a custom Django base Docker image, the company improved its application deployment, management, and scalability while adhering to its established standards for encryption, logging, monitoring, and security.

## Background

Our customer had been using AWS OpsWorks to manage its infrastructure, which consisted of Django applications deployed on several instances in an OpsWorks stack. As the company grew, it faced challenges in scalability and infrastructure management. To overcome these obstacles and modernize their infrastructure, our customer decided to hire Cloud Native Solutions to modernize and manage their infrastructure.

## Project Goals

After an initial evaluation of our customer’s infrastructure and their needs, we established the following project goals:

1. Migrate existing Django applications from AWS OpsWorks (Chef) to a containerized infrastructure on AWS Fargate.
2. Utilize Gitlab-CI for continuous integration and deployment (CI/CD) to streamline application deployment and management.
3. Implement Terraform for infrastructure as code (IAC) to maintain consistent and secure infrastructure provisioning.

## Migration Process

### Leverage Gitlab-CI for CI/CD Pipeline

- Utilize the self-hosted Gitlab instance already in place. The customer was already using Gitlab for source control.
- Configure Gitlab-CI to automatically build, test, and deploy Django applications to AWS Fargate.

### Create a Base Django Docker Image

- Develop a base Docker image with all the necessary packages and dependencies required to run Django applications according to company policies.
- This base image serves as the starting point for building application-specific images.
- Create a CI/CD pipeline for building / scanning / testing / publishing the image to ECR

### Develop a Terraform Module for Django Applications

- Create a reusable Terraform module that provisions the required resources for running a Django application on AWS Fargate.
- Ensure that the module adheres to company standards for encryption, logging, monitoring, scalability, and security.

### Application Containerization

For each Django application, in its own repository:

- Create a Dockerfile
- Start from the base Django Docker image previously created
- Add application code
- Install project-specific Python packages.
- Include a Terraform folder
- Define environment variables and secrets
- Load the Terraform module previously created with the required variables, secrets, and Docker image.

### Implement the CI/CD Pipeline

With all the pieces in place, we can use Gitlab-CI to automate the process of building, testing, and deploying Django applications as containers to AWS Fargate.
Each application repository would contain a Gitlab-CI file with the following stages:
Build Docker image
Scan image for vulnerabilities
Run tests
Publish the Docker image to ECR
Validate and Terraform plan
Terraform apply (manual for production, automated for all other environments)
This allowed us to enable continuous integration and deployment, reducing the need for manual intervention and human error.

## Results

By migrating to a containerized infrastructure on AWS Fargate, our customer achieved the following benefits:

- **Improved scalability:** Containerization allows the company to scale individual applications independently, optimizing resource utilization and performance.
- **Streamlined deployment process:** Gitlab-CI enables continuous integration and deployment, reducing manual intervention and human error.
- **Simplified infrastructure management:** Terraform facilitates easy management and tracking of infrastructure changes, resulting in a more maintainable and consistent infrastructure.
- **Enhanced security and compliance:** The custom Terraform module ensures that all Django applications comply with company standards for encryption, logging, monitoring, scaling, and security.

## Conclusion

Our customer’s migration to AWS Fargate, Gitlab-CI, and Terraform has proven to be a successful transformation. The new infrastructure delivers improved scalability, streamlined deployment, and simplified management while adhering to the company’s security and compliance requirements.
