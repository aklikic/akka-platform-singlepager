# Akka Platform — Capabilities Overview

*For enterprise architecture evaluation — mission-critical, regulated workloads*

---

## Architecture: Federation Plane and Application Plane

The platform separates concerns into two planes:

- **Federation Plane** — a global coordination system (managed by Akka) responsible for multi-region orchestration, cross-region failover, key rotation, role assignments, access token management, and application deployment across regions. If the Federation Plane is temporarily unavailable, running applications continue operating without interruption.
- **Application Plane(s)** — one or more regional runtime environments that host applications. Each application plane embeds a Kubernetes cluster, an encrypted persistence store, Akka operators for elasticity/routing/observability, traffic steering proxies, compute and storage, and a regional container registry. Application data does not leave the Application Plane or interact with the Federation Plane.

This separation ensures that application data remains within the customer's regional boundaries while centralized operations (deployment, failover, certificate rotation) are coordinated globally.

## Deployment Models

The platform supports multiple deployment configurations to meet varying infrastructure governance requirements:

| Model | Infrastructure | Managed By | Use Case |
|---|---|---|---|
| **BYOC** (Bring Your Own Cloud) | Akka provisions VPC, K8s, DB, networking in customer's cloud account | Akka | Full operational delegation; customer retains cloud account ownership |
| **BYOK8s** (Bring Your Own Kubernetes) | Customer provisions and owns K8s cluster and database | Akka manages platform within K8s | Organizations with existing K8s investment or strict infra control requirements |
| **Serverless Cloud** | Akka-hosted infrastructure | Akka | Fastest path to production; no infrastructure management |

In BYOC, infrastructure is bootstrapped via a Terragrunt utility (`akka-bootstrap`) that creates the necessary IAM roles, networking, and permissions in the customer's Azure subscription (or AWS account / GCP project). Akka then remotely provisions and manages the region. The customer maintains custodial ownership of all cloud resources.

**Private connectivity** is supported through VNet Peering with NAT Gateways (Azure) and equivalent patterns on AWS/GCP, enabling fully private region-to-region communication without internet-facing traffic. For private Kubernetes API endpoints, Teleport provides a zero-trust identity proxy using encrypted gRPC-over-mTLS tunnels with cryptographic identity verification — no public K8s API exposure required.

## Multi-Region Operations

Multi-region replication is a built-in runtime capability, not an application-level concern.

- **Replicated Reads** — full state replication across regions with automatic primary selection (request-region or pinned-region modes). Entities are accessible from any region; writes are routed to the primary.
- **Replicated Writes** — CRDT-based concurrent writes across regions with automatic conflict detection and resolution using version vectors upon reconnection. No application-level merge logic required. Replication filters allow restricting which data replicates between regions.
- **Failover guarantees** — sub-one-minute RTO and zero-byte RPO for application data. Regional failure triggers automatic traffic redistribution with no operator intervention. CLI-based cross-region failover with data consistency verification on recovery.
- **Data sovereignty** — entity state can be pinned to specific regions for regulatory compliance, while still participating in cross-region routing.

All cross-region communication is authenticated via mutual TLS with per-region root CAs, intermediate CA auto-rotation, and per-service certificate issuance.

## State Management and Persistence

Event-sourced entities provide an immutable, append-only event journal — every state transition is captured and auditable. The platform manages:

- Automatic snapshotting for recovery performance
- Exactly-once event processing semantics
- Persistence backend provisioning and management (PostgreSQL, encrypted at rest, sharded for performance)
- Memory auto-scaling and storage oversight
- Benchmarked to over 1M writes/second with <20ms write latency

Key-value entities are available for simpler state patterns. Both entity types participate in multi-region replication without additional application code.

## Operational Model

The Automated Operations layer eliminates the need to build and maintain distributed systems infrastructure:

- **Auto-elasticity** — cold starts and automatic scaling of compute and persistence in response to traffic load, without manual capacity planning
- **Self-healing** — automatic recovery and failover; built-in consensus and split-brain resolution
- **Self-clustering** — peer-to-peer node discovery and traffic steering without requiring a service mesh
- **Zero-downtime deployments** — rolling updates that transition traffic between versions side-by-side, even during data model changes
- **Runtime patching** — live JVM and infrastructure patches without triggering application redeployment or versioning; when self-managing, any JVM or infrastructure patch requires repackaging and redeploying the application
- **Certificate lifecycle** — fully automated rotation of service certificates, intermediate CAs, and encryption keys
- **Multi-tenancy** — multiple projects and teams share compute and data infrastructure, reported to reduce cloud costs by up to 90% compared to isolated deployments. Dedicated single-tenant regions are available where regulatory isolation is required.

## Security and Access Control

- **mTLS everywhere** — inter-service and cross-region communication secured by default with rotating certificates
- **Zero-trust remote access** — Teleport-based infrastructure access with cryptographic machine identity; supports private K8s API endpoints without public exposure
- **Secret management** — native project-level secrets injected as environment variables; supports key rotation without config changes. Integration with Azure KeyVault, AWS Secrets Manager, GCP Secret Manager, and HashiCorp Vault.
- **Encryption at rest** — all persisted data encrypted; host-level encryption enforced (e.g. Azure EncryptionAtHost)
- **Endpoint ACLs** — declarative access control at the API layer (service-to-service, internet, JWT-based principal matching)
- **IAM integration** — platform integrates with enterprise identity systems; supports Entra ID (Azure), service principals, and federated credentials. Scoped roles (Deployment, Editor, Viewer, SRE) with least-privilege permissions.
- **Vulnerability management** — Akka InfoSec performs automated scanning of source repositories and running environments; vulnerability reports shared via trust center
- **Custom domain TLS** — auto-provisioned via Let's Encrypt or customer-provided certificates with client certificate validation

## Observability

The observability stack combines Prometheus Operator (Akka runtime metrics) and Groundcover (eBPF-based logging, network flows, traces) without requiring application instrumentation:

| Signal | Detail |
|---|---|
| Metrics | Request rates, latency percentiles (p50/p95/p99), replication lag/throughput/errors, consumer lag, data ops |
| Logging | Aggregated with trace ID correlation; customer SDK service logs remain within the customer's environment and are never sent to the Federation Plane |
| Tracing | OpenTelemetry-compatible, configurable sampling, spans across entities/endpoints/consumers |

Platform telemetry from all regions is shipped to a centralized Federation Plane dashboard (60-day retention). Export destinations include OTLP (gRPC and HTTP), Prometheus Remote Write, Splunk HEC, Azure Monitor, and Google Cloud Monitoring.

## Disaster Recovery and Backups

| Component | Backup Method | RPO | RTO |
|---|---|---|---|
| Application data | Continuous point-in-time recovery; snapshots replicated cross-region | < 5 minutes | Sub-1 minute (multi-region failover) |
| Infrastructure (K8s state) | Velero backups to cross-region cloud object storage | Continuous | ~4 hours (full region rebuild) |
| Federation Plane | PIT database recovery, multi-region storage | < 5 minutes | ~6 hours |

Recovery from regional failure: new region provisioned, database restored from latest snapshot, Velero restores K8s state, Federation Plane re-projects resources. Snapshot retention is 7 days (configurable).

## Shared Responsibility Model

Responsibilities are clearly delineated between Akka and the customer:

| Domain | Akka | Customer |
|---|---|---|
| Infrastructure management & scaling | Provisions, manages, patches | Provides cloud account; reviews costs |
| Platform maintenance & security patches | Performs updates including vulnerability remediation | Informed of changes; manages application patches |
| Availability & performance monitoring | Monitors platform; notifies on incidents | Monitors application logs; configures export to their observability tools |
| Data backups | Manages infrastructure and persistence backups | Manages application code backups |
| Disaster recovery | Executes regional failover and rebuild | Configures DNS; informed of recovery status |
| IAM & access control | Integrates with enterprise IAM; manages platform roles | Provides identity, auth, and authorization policies |
| Incident response | 24/7 support; joint accountability | Joint accountability; additional contacts via Slack/email |
| Compliance & audit | Quarterly access-control reports; SIEM monitoring; db-level audit logging | Owns security audit logs and GDPR data management |

## Deployment and CI/CD

- Scoped service tokens for CI/CD pipelines (deploy, route, secret, and registry permissions)
- Regional or external container registries (supports customer Artifactory as mirror)
- Declarative service and route descriptors
- CLI-driven operations for all deployment and management tasks
- Bootstrap via Terragrunt for automated cloud account preparation

## Regulatory and Production Track Record

The platform is regulator-approved and deployed in production at financial institutions including Westpac and Visma. The underlying runtime has been proven over 18 years across 100,000+ production deployments. Contractual SLAs target 99.9999% availability — backed by the platform's self-healing, replication, and failover architecture.

---

**What the platform absorbs** — multi-region replication and failover orchestration, persistence provisioning and management, certificate lifecycle and key rotation, elastic scaling and capacity planning, observability pipeline, split-brain resolution, zero-downtime deployments, runtime patching, cross-region mTLS, zero-trust access management, disaster recovery procedures, and vulnerability management. These are operational capabilities that would otherwise require dedicated infrastructure, SRE, and security teams to design, build, certify, and continuously maintain.
