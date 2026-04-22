CloudEagle DevOps Assignment

Overview

This assignment demonstrates the design of a CI/CD pipeline and GCP-based infrastructure for a Spring Boot service (`sync-service`) that connects to MongoDB and is deployed across multiple environments (QA, Staging, Production).

Part 1 – CI/CD Design

1. Branching Strategy

A GitFlow-based branching model is used to map code changes to environments:

| Branch    | Environment | Purpose                              |
| --------- | ----------- | ------------------------------------ |
| feature/* | QA          | Feature development & testing        |
| develop   | Staging     | Integration & pre-production testing |
| main      | Production  | Live application                     |
| hotfix/*  | Production  | Emergency fixes                      |

Safety Measures

* Pull Request (PR) approval required before merging
* Protected `main` branch in GitHub
* Manual approval stage in Jenkins before production deployment


2. Jenkins Pipeline Design

The CI/CD pipeline automates build, test, and deployment.

Pipeline Stages


Checkout → Build → Test → Docker Build → Push Image → Deploy → Smoke Test


PR vs Merge Behavior

| Action           | Pipeline Behavior                               |
| ---------------- | ----------------------------------------------- |
| Pull Request     | Runs build & test only                          |
| Merge to develop | Deploy to Staging                               |
| Merge to main    | Deploy to Production (manual approval required) |


Rollback Strategy

If deployment fails:

* Previous Docker images are retained using version tags
* Rollback is performed by redeploying the last stable image

Example:


docker pull sync-service:<previous_version>
docker run sync-service:<previous_version>

This ensures quick recovery with minimal downtime.

3. Configuration Management

Environment-specific configurations are managed using separate config files:

application-qa.yml
application-staging.yml
application-prod.yml

This ensures isolation between environments and avoids misconfiguration.

Secrets Handling

Sensitive data such as MongoDB credentials and API keys are securely managed using:

Google Secret Manager

Best Practices:

* No secrets stored in code
* Inject secrets as environment variables during runtime

4. Deployment Strategy

 Selected Strategy: Rolling Deployment

Why Rolling Deployment?

* Zero or minimal downtime
* Gradual replacement of instances
* Cost-efficient (no duplicate environment needed)

Alternatives Considered:

* Recreate → Causes downtime 
* Blue-Green → High cost 

Part 2 – Infrastructure Design

1. Compute Choice

Selected: Google Cloud Run

Justification:

* Fully managed (no server maintenance)
* Auto-scaling (including scale-to-zero)
* Pay-per-use model (cost efficient for startups)


2. MongoDB Hosting

Selected: MongoDB Atlas

Benefits:

* Managed backups
* Auto-scaling
* Built-in security features

3. Networking

The application follows a secure and simple network architecture:

* Virtual Private Cloud (VPC) for isolation
* Cloud Run exposed via HTTPS endpoint
* MongoDB access restricted using IP whitelisting

4. Secrets & IAM

Secrets Management:

* Google Secret Manager used to store credentials securely

IAM (Identity & Access Management):

* Service accounts with least privilege access
* Role-based access control to limit permissions


5. Logging & Monitoring

Tools Used:

* Google Cloud Logging → Centralized logs
* Google Cloud Monitoring → Metrics & alerts

Benefits:

* Real-time monitoring
* Faster troubleshooting
* Alerting for failures

Architecture Overview

System Flow

User → Cloud Run (sync-service) → MongoDB Atlas
                ↓
        Cloud Logging & Monitoring


Key Design Highlights

* Automated CI/CD pipeline using Jenkins
* Secure secrets management
* Cost-optimized serverless deployment
* Scalable architecture using Cloud Run
* Zero/minimal downtime deployment strategy
* Robust monitoring and logging setup

Repository Contents

.
├── README.md
├── Jenkinsfile
└── architecture.png



Conclusion

This design ensures:

* Reliable and automated deployments
* Secure handling of sensitive data
* Scalable and cost-effective infrastructure
* High availability with minimal downtime


