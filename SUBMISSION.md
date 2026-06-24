# Submission

## What I changed and why

### App

I reviewed the Django application settings and addressed several issues that would prevent safe operation outside of a local development environment.

Changes made:

* Moved the Django SECRET_KEY to environment-variable based configuration with a development fallback.
* Changed DEBUG to be controlled through environment variables and default to False.
* Replaced the wildcard ALLOWED_HOSTS = ["*"] configuration with an environment-driven allow list.
* Updated database configuration to support PostgreSQL through environment variables while preserving SQLite as a local fallback.
* Moved session storage from filesystem-based sessions to database-backed sessions to improve reliability in containerized environments.
* Added basic production security settings:
    * Secure session cookies
    * Secure CSRF cookies
    * Browser XSS protection
    * MIME type sniffing protection

These changes improve security, portability, and production readiness while keeping local development simple.

### Docker

I containerized the application using Docker and Docker Compose.

Changes made:

* Added a Dockerfile for building the application image.
* Added a Docker Compose configuration that provisions:
    * Django application container
    * PostgreSQL database container
* Configured health checks to ensure the application waits for PostgreSQL before startup.
* Added persistent database storage through Docker volumes.
* Automated database migrations and demo data creation during container startup.
* Built frontend assets and collected Django static files during image creation.

The result is a reproducible local environment that can be started with a single command and does not require Python, Node.js, or PostgreSQL to be installed on the host system.

### CI

I replaced the existing workflow with a more production-oriented GitHub Actions pipeline.

Changes made:

* Run on pull requests and pushes to the main branch.
* Install Python and Node.js dependencies.
* Build frontend assets.
* Execute automated tests.
* Build a Docker image.
* Push the image to GitHub Container Registry (GHCR) on successful merges to main.
* Tag images using both:
    * latest
    * Git commit SHA

Using commit SHA tags allows deterministic rollback to a known-good image version.

## Tradeoffs

I intentionally focused on operational and deployment concerns rather than application refactoring.

Examples:

* I kept SQLite support for local development instead of forcing PostgreSQL in every environment.
* I used environment-variable based secret management rather than integrating a dedicated secret manager because the assignment specifically targets local execution and GitHub-based workflows.
* I kept the existing application architecture and business logic unchanged unless required for operational improvements.

## What I’d do with another day

Given additional time, I would focus on:

* Adding a dedicated non-root application user inside the container.
* Introducing container vulnerability scanning in CI.
* Expanding automated test coverage.
* Adding health check endpoints and readiness/liveness probes.
* Adding a staging environment deployment workflow.
* Improving secrets management documentation.
* Reviewing authentication and authorization flows for additional security hardening.

## How to run

Recommended local run path:

```bash
docker compose up --build
```
This starts the full local stack:

* Django application container
* PostgreSQL database container
* Persistent Docker volume for database data

Docker Desktop / Docker daemon must be running on the local machine before using Docker Compose.
Application:
```text
http://localhost:8000
```

Demo credentials:
```text
Username: demo
Password: demo
```

Development-only alternative:

```bash
python manage.py runserver
```

## Deployment plan

### Platform

For a production deployment, I would keep the current Django application architecture and deploy it on AWS as a containerized application.

My preferred setup would be:

* ECS Fargate for the application containers
* RDS PostgreSQL for the database
* Application Load Balancer for HTTPS traffic
* Route 53 for DNS
* ACM for TLS certificates
* ECR for container image storage
* CloudWatch for monitoring and alerting
* Terraform for infrastructure provisioning

For an application of this size, I would start with a single AWS region and focus on simplicity, reliability, and operational visibility.

### Environments

I would create three environments:

* Development
* Staging
* Production

The CI/CD pipeline would build the Docker image once and tag it with the Git commit SHA.

The same image would be promoted from Development to Staging and then to Production after validation and testing. This provides consistency across environments and avoids surprises in Production because every environment runs the exact same application image.

Infrastructure differences between environments would be managed through Terraform variables and separate Terraform state files rather than building different application images.

### Network Layout

I would place the Application Load Balancer in public subnets and run ECS Fargate tasks in private subnets.

The PostgreSQL database would also run in private subnets and would not be publicly accessible.

Traffic flow:
```text
Internet
  -> Route 53
  -> HTTPS / ACM
  -> Application Load Balancer
  -> ECS Fargate
  -> RDS PostgreSQL
```

### Scalability

The ECS service would use ECS Service Auto Scaling based on CloudWatch metrics such as CPU and memory utilization.

As traffic increases, additional Fargate tasks would be launched automatically behind the Application Load Balancer. As traffic decreases, unused tasks would be removed to optimize cost.

### Secrets Management

Sensitive values would be stored in AWS Secrets Manager and injected into ECS tasks at runtime.

Examples:

* Django SECRET_KEY
* Database credentials
* API keys

For less sensitive configuration values, I would use AWS Systems Manager Parameter Store.

Examples:

* Environment names
* Application configuration values
* Allowed hostnames

Secrets should never be stored in source control, Docker images, or Terraform code.

### CI/CD and Rollout Strategy

GitHub Actions would validate every change.

Deployment flow:

1. Pull request validation
2. Build Docker image
3. Push image to ECR using a Git SHA tag
4. Deploy to Development
5. Promote to Staging after testing
6. Promote to Production after approval

For normal application releases, I would use rolling deployments behind the Application Load Balancer so new tasks are validated before traffic is shifted from the previous version.

### Rollback Strategy

Each image is tagged with the Git commit SHA.

If a deployment causes issues:

1. Identify the last known-good image
2. Redeploy that image
3. Verify application health
4. Investigate the failed release

Rollback becomes a deployment action rather than a rebuild process.

### Database Migration Approach

For a migration from an existing environment to AWS, I would use a planned maintenance window.

The process would be:

1. Put the application into maintenance mode
2. Take a final database snapshot
3. Restore the snapshot into RDS
4. Validate application functionality
5. Switch DNS to the new environment
6. Monitor the application after cutover

For an application of this size, a controlled cutover with a short maintenance window is simpler and lower risk than implementing a more complex live migration solution.

### Logs, Metrics, and Alerts

Application and infrastructure logs would be sent to CloudWatch.

I would create dashboards and alarms for:

* Application errors
* Failed deployments
* ECS CPU and memory utilization
* ECS task failures
* Database connectivity issues
* RDS resource utilization
* ALB health check failures

Alerts would be sent through SNS and routed to Slack.

### Before Real Users

Before exposing the application to production traffic, I would want:

* HTTPS enabled
* Secure secret management
* Private database networking
* Automated backups
* CloudWatch monitoring and alarms
* CI/CD pipeline with rollback capability
* Infrastructure as Code
* Documented deployment procedure
* Documented rollback procedure
* Basic load testing
* Security review

The goal is to make deployments repeatable, rollback straightforward, and production issues visible before they impact users.
