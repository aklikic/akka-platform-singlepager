# Akka Platform — Where the Value Is

*An enterprise architecture perspective on what the platform delivers versus what you'd need to build and sustain internally.*

---

## Akka as a Unified Platform

Akka integrates three capabilities into a single system:

| Capability | What it provides | Who uses it |
|---|---|---|
| **SDK** | Programming model — event-sourced entities, key-value entities, workflows, views, endpoints. Developers write business logic against the SDK. | Application developers |
| **Runtime** | Distributed systems engine — clustering, sharding, replication, traffic steering, split-brain resolution, exactly-once processing. These are built into the runtime and activated by the SDK's component model. No application code required. | Activated automatically |
| **Platform** | Infrastructure-as-Code and operational automation — purpose-built for running Akka applications. Provisions, configures, monitors, patches, scales, and recovers the infrastructure that the Runtime runs on. | Operations / Akka SRE |

The Platform is where the operational value compounds. It takes the Runtime's capabilities and ensures they work in production, across regions, continuously, without dedicated internal teams.

The rest of this document focuses on the Platform layer and what it delivers.

---

## 1. Infrastructure Operations

The Platform supports two deployment models: **BYOK8s** (Bring Your Own Kubernetes), where the Platform runs within your existing Kubernetes clusters and connects to your managed databases, and **BYOC** (Bring Your Own Cloud), where the Platform also provisions and manages the Kubernetes clusters, databases, and networking within your cloud account. In both models, you retain custodial ownership of all cloud resources.

In a BYOK8s deployment — which aligns with organizations that have existing Kubernetes investment and infrastructure governance — you retain full ownership of the underlying clusters, networking, storage, and database instances. The Platform manages the Akka layer on top, absorbing the day-2 operational work:

| Concern | What the Platform handles | Without the Platform |
|---|---|---|
| **Scaling** | Auto-elasticity of Akka application instances within Kubernetes based on real-time load. No pod tuning or scaling runbooks. | You size, monitor, and adjust application scaling yourself. |
| **Patching** | Live JVM and Akka platform patches without triggering application redeployment. Kubernetes major version and database upgrades are coordinated jointly. | Every platform patch means repackaging and redeploying every service. |
| **Certificate management** | Automated rotation of service certs, intermediate CAs, and encryption keys across all regions. | You deploy and operate cert-manager, build rotation automation, handle cross-region CA hierarchies. |
| **Failover coordination** | Cross-region failover with data consistency verification — coordinated by the Federation Plane. | You write and test failover runbooks. An on-call engineer executes them under pressure. |
| **Persistence management** | Akka's encrypted event store managed, sharded, and scaled within your database instances. Benchmarked to >1M writes/sec at <20ms latency. | You manage Akka's storage layer yourself — schema management, sharding, performance tuning. |
| **Deployments** | Zero-downtime rolling updates with side-by-side version transitions, even during data model changes. | You build deployment pipelines with blue-green or canary logic and test them against your data model. |

### Multi-Region Operations

The Akka Runtime provides multi-region replication as a built-in capability — automatic state replication with configurable primary selection (follow-the-writer or pinned), CRDT-based conflict resolution when regions reconnect, sub-one-minute failover with zero data loss (zero-byte RPO), and regional data pinning for sovereignty requirements. The application code is identical whether it runs in one region or four.

The Platform adds the operational layer that makes multiple regions work as a coherent system. It configures cross-region mTLS with per-region root CAs and automated certificate rotation. It deploys traffic steering proxies and Akka operators for elasticity, routing, and service management. It coordinates failover with data consistency verification. It handles the cross-region backup replication that makes zero-byte RPO achievable — whether within your existing Kubernetes infrastructure (BYOK8s) or in Platform-provisioned clusters (BYOC).

Without the Platform, provisioning, connecting, securing, and operating this infrastructure across regions falls on internal teams. That's where the multi-year engineering programme lives.

The RACI is explicit: Akka is responsible and accountable for the platform layer within Kubernetes — application scaling, platform maintenance, availability monitoring, Akka platform backups, and disaster recovery of Akka services. In BYOK8s, the customer is responsible for the underlying infrastructure (Kubernetes clusters, database instances, networking); in BYOC, the Platform provisions and manages that infrastructure as well. In both models, the customer owns application code, application-level patching, and IAM/compliance policies. Kubernetes major versions and database major upgrades are coordinated between both parties.

The net effect is that the Akka workload does not require a dedicated SRE team.

## 2. Security

The Runtime provides application-level security (mTLS between services, endpoint ACLs, JWT integration). The Platform complements this with infrastructure-level security:

- **Zero-trust remote access**: A zero-trust proxy deployed in the customer's cluster provides cryptographic machine identity for all cross-region management. Private Kubernetes API endpoints remain private — no public exposure, no IP whitelisting. Encrypted gRPC-over-mTLS reverse tunnels between Federation Plane and every region. Akka SRE access is scoped via RBAC integrated with customer SSO.
- **Certificate lifecycle**: Per-region root CAs, automated intermediate CA rotation, per-service certificate issuance — all managed by the Platform without operator intervention.
- **Encryption at rest**: All persisted data encrypted within the Akka persistence layer. Host-level encryption remains under customer control at the infrastructure layer.
- **Least-privilege IAM**: Scoped roles (Deployment, Editor, Viewer, SRE) with auditable permissions. Integrates with Entra ID and enterprise identity systems.
- **Continuous vulnerability scanning**: Akka InfoSec scans source repositories and running environments. Reports shared via trust center.
- **Private connectivity**: VNet Peering with NAT Gateways enables fully private region-to-region communication without internet-facing traffic.

Assembling this from individual components (cert-manager, Vault, zero-trust proxies, policy engines, network policies, identity federation) is achievable. Keeping it operational, patched, auditable, and regulator-ready across multiple regions is where the cost compounds. The Platform delivers this as a maintained, integrated system.

## 3. Observability

Generic observability tooling doesn't understand Akka's internals. The Platform ships an OpenTelemetry-based stack that does:

- **Runtime-aware metrics** — replication lag, replication throughput, replication errors, consumer lag, entity data ops. These are metrics that generic Kubernetes monitoring cannot provide.
- **eBPF-based signal collection** for logs, network flows, and traces — collected at the kernel level without application instrumentation.
- **Centralized Federation Plane dashboard** aggregating platform telemetry from all regions with 60-day retention.

The observability pipeline is built on OpenTelemetry, meaning it integrates natively with OTel-compatible backends. Customer application logs stay within the customer's environment — never shipped to the Federation Plane. Platform telemetry can be exported to the customer's existing observability tools via OTLP, Splunk HEC, Azure Monitor, or Google Cloud Monitoring.

The value is that this pipeline is pre-integrated with the Runtime's internals and maintained as part of the operational contract — not a separate system to build and operate.

## 4. Disaster Recovery

The Runtime provides the replication and consistency mechanisms. The Platform wraps them in a complete DR strategy:

- **Application data**: Continuous point-in-time recovery with cross-region snapshot replication (RPO < 5 minutes). Multi-region Runtime replication provides sub-one-minute failover for active regions.
- **Akka platform state**: Automated backups of Kubernetes resources to customer's cloud object storage. Akka restores platform services; in BYOK8s the customer restores the database from point-in-time recovery, in BYOC this is managed by the Platform.
- **Federation Plane**: PIT database recovery, multi-region storage (RPO < 5 minutes, RTO ~6 hours).

Recovery from regional failure is a coordinated procedure: Akka restores platform services from backup, Federation Plane re-projects resources. In BYOK8s, the customer restores the database from PIT backup; in BYOC, the Platform handles this. The split is clear and rehearsed, with 24/7 incident response from Akka's SRE team.

The alternative is designing this yourself: choosing backup tools, building cross-region storage replication, writing and testing runbooks, training on-call teams, and conducting regular DR exercises. Each of those is a project. Together they're a programme.

## 5. Regulatory Readiness

The Platform is already deployed and regulator-approved at financial institutions. This means:

- The operational model has passed regulatory scrutiny for financial services
- Audit trails are architectural (immutable event journals from the Runtime, trace ID correlation, access-control reporting from the Platform)
- Data sovereignty controls (regional pinning) are a Runtime configuration option, operationalized by the Platform's regional infrastructure
- Quarterly access-control reports and SIEM monitoring are part of the operational contract
- The shared responsibility model is documented and auditable

For a regulated organization, the compliance burden of a platform that's already been through this process is materially different from one that hasn't.

## 6. Time to Production

The bootstrap path for a new Azure region:

**BYOK8s** — customer provisions infrastructure, Akka manages the platform layer:

1. Capacity planning with Akka solutions architect
2. Customer provisions Kubernetes cluster, database, networking, and DNS subdomain zones
3. Customer seeds database credentials into Akka namespaces and configures certificate issuer
4. Customer installs zero-trust proxy (provided by Akka) for secure Federation Plane connectivity
5. Akka configures the cluster as an Akka application plane region, sets up internal observability
6. Smoke tests; region is handed over and live

**BYOC** — Platform provisions and manages infrastructure within the customer's cloud account:

1. Customer runs bootstrap utility (provided by Akka) to set up IAM roles and permissions
2. Share credentials with Akka; remote provisioning begins
3. Customer configures DNS records
4. Smoke tests; region is live

Adding a second region for multi-region HA follows the same process. The application code doesn't change — the Runtime handles replication, the Platform handles the operational layer.

## 7. Production Readiness

A common cost in enterprise projects is the gap between "application works in dev" and "application is production-ready." Day-2 concerns — scaling, monitoring, alerting, patching, backup, failover, certificate rotation, security hardening — are typically addressed late in the project and require significant engineering and operations effort.

With the Platform, these concerns are resolved from day one. The infrastructure is already configured for production: observability is collecting metrics before the first deployment, certificates are rotating, backups are running, scaling policies are active, and failover procedures are in place. There is no separate production-readiness workstream because the Platform's operational baseline *is* the production configuration.

This compresses the timeline from development to production and eliminates the risk of day-2 gaps discovered post-launch.

---

**The core proposition**: the SDK gives developers a productive programming model. The Runtime gives that model distributed systems capabilities — replication, clustering, failover, consistency. The Platform ensures those capabilities work in production by provisioning, operating, securing, monitoring, patching, and recovering the infrastructure underneath. The infrastructure layer is where the real cost, risk, and staffing burden lives — and that's precisely what the Platform absorbs.
