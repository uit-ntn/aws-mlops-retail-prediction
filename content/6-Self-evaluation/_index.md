---
title: "Self-Assessment"
weight: 6
chapter: false
pre: " <b> 6. </b> "
---

During the **Retail MLOps on AWS** workshop, I had the opportunity to apply Cloud/DevOps/MLOps knowledge to a practical end-to-end pipeline: from S3-based data organization, model training and evaluation on SageMaker, model versioning with SageMaker Model Registry, containerizing a FastAPI inference service, deploying to EKS, monitoring with CloudWatch, implementing CI/CD automation, and optimizing cost with teardown procedures after demos.

Through this workshop, I improved the following skills:

- Designing a production-like MLOps architecture (networking, security, scalability)
- Building data-to-model pipelines with clear quality metrics and verification steps
- Working with AWS services: S3, SageMaker, ECR, EKS, IAM/IRSA, CloudWatch, VPC endpoints
- Containerizing inference services (multi-stage builds, non-root, healthcheck, image scanning)
- Deploying and operating Kubernetes workloads (manifests, Service/Ingress, HPA, debugging)
- Cost optimization and operational hygiene (Spot, lifecycle policies, log retention, scheduled start/stop, teardown scripts)

To reflect objectively on my progress after completing the workshop, I self-assess using the criteria below:

| No. | Criteria                            | Description                                                                                | Good | Fair | Average |
| --- | ----------------------------------- | ------------------------------------------------------------------------------------------ | ---- | ---- | ------- |
| 1   | **Professional knowledge & skills** | Understanding MLOps concepts, implementing training/deployment pipeline, output quality    | ✅   | ☐    | ☐       |
| 2   | **Ability to learn**                | Learning and applying AWS/EKS/SageMaker/ECR/CloudWatch quickly                             | ✅   | ☐    | ☐       |
| 3   | **Proactiveness**                   | Taking initiative in debugging (pods, image pull, IRSA), improving configs and cleanup     | ✅   | ☐    | ☐       |
| 4   | **Sense of responsibility**         | Delivering based on task checklist, meeting model metrics and verification requirements    | ✅   | ☐    | ☐       |
| 5   | **Discipline**                      | Following naming conventions, region consistency, and teardown steps to avoid cost leakage | ☐    | ✅   | ☐       |
| 6   | **Progressive mindset**             | Accepting feedback and iterating (Spot, lifecycle, log retention, scheduling)              | ✅   | ☐    | ☐       |
| 7   | **Communication**                   | Explaining architecture choices, reporting progress, documenting verification steps        | ☐    | ✅   | ☐       |
| 8   | **Teamwork (if applicable)**        | Coordinating modules (data/training/deploy/monitor/cost) and integrating deliverables      | ☐    | ✅   | ☐       |
| 9   | **Professional conduct**            | Working systematically, respecting conventions, emphasizing least privilege and security   | ✅   | ☐    | ☐       |
| 10  | **Problem-solving skills**          | Identifying root causes and proposing fixes (VPC endpoints, IRSA/RBAC, LB/SG, logs)        | ☐    | ✅   | ☐       |
| 11  | **Contribution to project/team**    | Producing a reusable end-to-end pipeline plus operational/teardown scripts                 | ✅   | ☐    | ☐       |
| 12  | **Overall**                         | Overall completion level and ability to reproduce the system independently                 | ✅   | ☐    | ☐       |

## Needs Improvement

- **Stronger documentation quality:** clearer architecture diagrams, prerequisites, and a structured troubleshooting checklist.
- **More robust CI/CD gates:** add stronger quality checks (tests, policy checks, clear rollback strategy) and separate dev/stage/prod environments.
- **Better observability maturity:** dashboards/alerts aligned with real SLOs (latency, error rate), log sampling, and optional tracing.
- **Deeper cost controls:** automated schedules aligned with demo windows, stricter log ingestion control, and periodic ECR cleanup.
