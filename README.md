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

**Part-2 Infrastructure Design**

**Overview :**

This repository contains the infrastructure design for deploying a Spring Boot service (`sync-service`) on Google Cloud Platform (GCP). The design focuses on scalability, security, cost efficiency, and operational simplicity, making it suitable for a startup environment.


**Request Flow (Step-by-Step) :**

1. A user sends a request over HTTPS from a web or mobile application.

2. The request first passes through Cloud Armor, which acts as a security layer to protect against malicious traffic and common web attacks.

3. The request is then received by the HTTPS Load Balancer, which routes it to the appropriate backend service.

4. The request is forwarded to Cloud Run, where the Spring Boot application is running.

5. The application retrieves required secrets such as database credentials from Secret Manager using secure access.

6. Based on the request, the application interacts with MongoDB Atlas through a private endpoint:

   * Write operations go to the primary node
   * Read operations can be served from secondary nodes

7. If the request involves a long-running or heavy task, the application publishes a message to Cloud Pub/Sub instead of processing it immediately.

8. Cloud Run Jobs (background workers) consume messages from Pub/Sub and process them asynchronously.

9. Once processing is complete, the application sends the response back through the Load Balancer to the user.

10. Throughout this process, logs, metrics, and traces are collected using Cloud Logging, Cloud Monitoring, Cloud Trace, and Error Reporting.


Architecture Diagram :

Add the architecture diagram file in this repository as:

`**Infra_Design.jpg**`

**Compute Choice :**

The service is deployed using Cloud Run.

Cloud Run was selected because it is a serverless platform that automatically scales based on incoming traffic. It eliminates the need to manage infrastructure and allows cost savings by charging only for actual usage. This makes it well suited for startup workloads where traffic may be unpredictable.

Other options such as GKE and Compute Engine were considered but not chosen. GKE provides more control but introduces operational complexity and higher costs. Compute Engine requires manual scaling and maintenance, which increases operational overhead.


**Database Design :**


MongoDB Atlas is used as the database solution.

This is a managed database service that provides built-in replication, high availability, and automated backups. It reduces the need for manual database management and integrates securely with GCP through private endpoints.

The database is not exposed to the public internet, ensuring secure access from the application.


**Networking :**

The architecture uses a Virtual Private Cloud (VPC) to separate public and private components.

External traffic enters through an HTTPS Load Balancer. Cloud Armor is used to protect against common web attacks and distributed denial-of-service (DDoS) attacks.

The application runs on Cloud Run and communicates with private resources such as the database through secure networking mechanisms.


**Secrets and Access Control :**

Sensitive information such as database credentials and API keys are stored in Secret Manager.

Access to resources is controlled using Identity and Access Management (IAM), following the principle of least privilege. Each service is granted only the permissions it needs to function.

This ensures that no secrets are hardcoded in the application and reduces security risks.


**Asynchronous Processing :**

For tasks that do not need immediate completion, the system uses an asynchronous processing model.

Cloud Pub/Sub is used as a messaging system where tasks are queued. Cloud Run Jobs act as background workers that consume these tasks and process them independently.

This approach improves system performance and ensures that user requests are not delayed by long-running operations.

**CI/CD Pipeline :**

The deployment process is automated using a CI/CD pipeline.

The typical flow is:

* Code is pushed to the source repository
* Cloud Build compiles the application and runs tests
* A container image is created and stored in Artifact Registry
* The application is deployed to Cloud Run

This ensures consistent and reliable deployments with minimal manual intervention.

Logging and Monitoring :

The system uses GCP’s observability tools to track performance and issues.

* Cloud Logging is used for collecting application logs
* Cloud Monitoring is used for metrics and alerting
* Cloud Trace is used for request tracing
* Error Reporting is used to capture and analyze application errors

This provides complete visibility into system behavior and helps in troubleshooting.

**Cost Considerations :**

The design prioritizes cost efficiency.

Cloud Run automatically scales down to zero when there is no traffic, eliminating idle costs. Managed services reduce operational overhead. Logging retention and resource sizing can be adjusted based on usage to control costs further.

**Scalability :**

The system is designed to scale automatically.

Cloud Run scales based on incoming requests. Pub/Sub handles message scaling independently, and MongoDB Atlas supports horizontal scaling if required.

**Security Considerations :** 

The architecture includes several security best practices:

* All traffic is served over HTTPS
* Cloud Armor provides protection against web-based attacks
* The database is not publicly accessible
* Secrets are stored securely in Secret Manager
* IAM enforces strict access control

**Trade-offs :**

There are some trade-offs in this design.

Cloud Run provides ease of use but offers less control compared to Kubernetes. MongoDB Atlas introduces an external dependency outside GCP. Serverless platforms may also introduce minor cold start latency.

**Conclusion :**

This design provides a secure, scalable, and cost-effective infrastructure for running a modern backend service on GCP. It balances simplicity and functionality, making it suitable for both startup environments and production workloads.

