# ET Architecture Review Board

# 2026 ARB Cloud Component Standard

## Component Metadata

* **Jira Ticket:** `[ACBSD-XXXX]`
* **ServiceNow Ticket:** `[RITMXXXX]`
* **Component Standard:** Amazon Managed Streaming for Apache Kafka (MSK)
* **Component Type:** AWS Service
* **Assigned Architect:** `[Name]`
* **Enterprise Architecture Classification:** Core

## Purpose

The purpose of this Cloud Component Standard is to:

* Define approved and restricted **use cases** for Amazon MSK
* Ensure alignment to **Well-Architected Framework principles and enterprise standards**
* Establish **implementation guardrails** for streaming data workloads
* Enable DevOps teams to build **IaC modules** for MSK clusters
* Provide **pre-approved component patterns** for event-driven architectures (subject to guardrails + ARB approval)
* Document the **deployment model requirement** mandating Express Brokers for production
* Serve as authoritative reference for **architecture patterns and solution designs**
* Ensure component approval is published to Architecture Repository and AWS Service Inventory with ARB status

## Version Control

| Version | Date       | Author | Description                                        |
| ------- | ---------- | ------ | -------------------------------------------------- |
| 1.0     | 2026-07-16 | ARB    | Initial component standard — Approved with restrictions |

## Overview

### Drivers and Motivation

Amazon Managed Streaming for Apache Kafka (Amazon MSK) is a fully managed service (IAM namespaces: `kafka` for management plane, `kafka-cluster` for data plane) that provisions and operates Apache Kafka clusters within customer VPCs. MSK handles broker provisioning, configuration, patching, high availability across multiple Availability Zones, and automatic storage management.

MSK is being introduced to support event-driven architectures, real-time data streaming, Change Data Capture (CDC), and analytics pipeline decoupling across the enterprise. Kafka is the industry standard for high-throughput, low-latency distributed streaming and is required for workloads that cannot tolerate the batching latency of traditional ETL approaches.

MSK differentiates from alternatives by providing a fully managed Kafka experience with native AWS integrations (IAM authentication, KMS encryption, CloudWatch monitoring), eliminating the operational burden of self-hosting Kafka on EC2/EKS while retaining full Apache Kafka API compatibility.

### Business Value

* Enables real-time event-driven architectures for microservices decoupling and domain event publishing
* Supports CDC patterns for replicating database changes to analytics, search, and downstream systems
* Provides durable, ordered message delivery for financial transaction processing and audit trails
* Reduces operational overhead — no cluster management, patching, or capacity planning (Express Brokers)
* Supports high-throughput workloads: millions of messages per second with sub-10ms latency
* Native integration with AWS Glue, Lambda, Kinesis Data Analytics, and S3 for stream processing
* MSK Connect provides managed Kafka Connect for source/sink integrations without custom infrastructure

### Technology Reference Model (TRM)

| Component                         | Status |
| --------------------------------- | ------ |
| Amazon MSK (Express Brokers)      | Buy    |
| Amazon MSK (Standard Brokers)     | Hold   |
| Amazon Kinesis Data Streams       | Hold   |
| Self-hosted Kafka on EC2/EKS      | Sell   |
| RabbitMQ / ActiveMQ (self-hosted) | Sell   |

### Recommendation

* This component: **Buy**

**Non-Production:** Approved for immediate use in all non-production environments. Both Express Brokers and Standard Brokers are acceptable. SCPs enforcing TLS-only communication and IAM authentication must still be applied to prevent accidental creation of plaintext or unauthenticated clusters. Non-production clusters should use customer-managed KMS keys to maintain consistency with production configurations. Standard controls (Security Groups, CloudTrail, broker logging) are sufficient.

**Production:** Approved with mandatory controls. Express Brokers are required for all production clusters — they enforce TLS + IAM authentication by default, auto-scale without manual capacity planning, and recover 90% faster than Standard Brokers. The following guardrails must be implemented to protect production data:

1. **Customer-managed KMS key** with annual rotation enabled
2. **Topic-level IAM authorization** — each producer/consumer scoped to specific topic ARNs; no wildcard access
3. **Data classification per topic** — topics carrying PII/NPI/PHI require DPIA and OneTrust registration
4. **Application-layer DLP** — producers must tokenize/mask sensitive data before publishing (MSK has no native DLP)
5. **Broker logs → Splunk** — authentication failures, configuration changes, and operational events forwarded for SIEM monitoring
6. **Consumer lag alerting** — CloudWatch alarms on sustained consumer group lag to detect processing delays
7. **Security Hub MSK.1–MSK.6 compliant** — all controls must report compliant status before go-live
8. **Multi-VPC PrivateLink prohibited** without ISO/DPO/ARB exception approval
9. **Quarterly Sailpoint entitlement review** of all IAM roles with `kafka-cluster:*` permissions
10. **Change management** — cluster configuration changes, version upgrades, and topic creation via IaC with approval

### GenAI Enablement

* Does this enable GenAI use cases? **No** (data streaming infrastructure only)
* MSK may transport events that trigger GenAI model invocations (e.g., streaming data to a Bedrock-backed consumer), but does not itself provide GenAI capabilities
* If topics carry data consumed by GenAI services, the target service's GenAI governance applies

## Use Cases

### ✅ In Scope (Approved)

* **Event-driven architecture** — Domain event publishing and consumption between microservices
* **Change Data Capture (CDC)** — Database change replication via Debezium/DMS source connectors
* **Real-time analytics pipelines** — Streaming data to S3, Redshift, OpenSearch via MSK Connect sink connectors
* **Transaction event streaming** — Financial transaction events for fraud detection, audit, and reconciliation
* **Application integration** — Asynchronous communication between bounded contexts
* **IoT event ingestion** — High-volume device telemetry collection and processing
* **Log aggregation** — Centralized application log streaming for operational analytics
* **Data lake ingestion** — Real-time data landing into S3-based data lake via Firehose or Connect

### ❌ Out of Scope (Restricted)

* Public-facing Kafka endpoints (incompatible with architecture — no IGWs)
* Direct internet ingestion without the Hosting Service pattern
* Multi-VPC Private Connectivity (PrivateLink Endpoint Services) without ISO/DPO/ARB exception
* Cross-account cluster policies without explicit architecture review
* MSK Serverless for production workloads without separate ARB evaluation
* Unauthenticated or plaintext clusters in any environment
* Topics carrying PII/NPI/PHI without DPIA and data classification documentation
* MSK Connect connectors reaching SaaS endpoints without Browsing Service firewall approval

## Component Capability Model

### Capability Map

| Type      | Capability                        | Description                                                                              | Priority (MoSCoW) | Lifecycle |
| --------- | --------------------------------- | ---------------------------------------------------------------------------------------- | ----------------- | --------- |
| Technical | High-Throughput Streaming         | Millions of messages/second with sub-10ms latency, durable ordered delivery              | Must Have         | Introduce |
| Technical | IAM Authentication & Authorization| Topic-level access control via IAM policies on `kafka-cluster` actions                   | Must Have         | Introduce |
| Technical | KMS Encryption at Rest            | Always-on AES-256 encryption with CMK support                                           | Must Have         | Introduce |
| Technical | TLS Encryption in Transit         | TLS 1.2 for all client-to-broker and broker-to-broker communication                     | Must Have         | Introduce |
| Technical | Multi-AZ High Availability        | Automatic replication across 3 AZs with 99.9% SLA                                       | Must Have         | Introduce |
| Technical | Auto-Scaling (Express Brokers)    | Automatic throughput and storage scaling without manual capacity planning                 | Must Have         | Introduce |
| Technical | MSK Connect                       | Managed Kafka Connect for source/sink connectors (S3, DynamoDB, RDS, OpenSearch)         | Should Have       | Introduce |
| Technical | Tiered Storage                    | Offload cold partition data to S3 for cost-effective long-term retention                 | Should Have       | Introduce |
| Technical | Schema Registry (Glue)            | Schema validation and evolution for Avro/JSON/Protobuf messages                          | Should Have       | Introduce |
| Business  | Event-Driven Architecture         | Decouple microservices via asynchronous domain event publishing                          | Must Have         | Introduce |
| Business  | Real-Time Analytics               | Stream processing for operational dashboards, fraud detection, and alerting              | Must Have         | Introduce |
| Technical | MSK Replicator                    | Cross-region cluster replication for DR                                                   | Could Have        | Introduce |
| Technical | MSK Serverless                    | Fully automatic Kafka without cluster management                                         | Could Have        | Future    |

### Capability Evolution

* **Introduce:** Express Brokers, IAM auth, KMS, TLS, Multi-AZ, MSK Connect, Tiered Storage, Schema Registry
* **Enhance:** (future) MSK Serverless evaluation for non-critical workloads; MSK Replicator for DR
* **Retire:** Self-hosted Kafka on EC2/EKS (migrate to MSK)

## Component Architecture

### Architecture Diagram

```
+-----------------------------------------------------------------------+
|                    APPLICATION ACCOUNT (VPC)                            |
+-----------------------------------------------------------------------+
|                                                                         |
|  +-----------------------------------+                                  |
|  |  MSK Cluster (Express Brokers)    |                                  |
|  |  - Broker 1 (AZ-a, Private Subnet)|                                  |
|  |  - Broker 2 (AZ-b, Private Subnet)|                                  |
|  |  - Broker 3 (AZ-c, Private Subnet)|                                  |
|  |  - EBS/Managed Storage (encrypted)|                                  |
|  |  - IAM Auth + TLS 1.2 enforced    |                                  |
|  +-----------------------------------+                                  |
|           |              |              |                                |
|     Port 9098      Port 9098      Port 9098                             |
|     (IAM TLS)      (IAM TLS)      (IAM TLS)                            |
|           |              |              |                                |
|  +------------------+  +------------------+  +------------------+       |
|  | Producer App     |  | Consumer App     |  | MSK Connect      |       |
|  | (ECS/Lambda/EC2) |  | (ECS/Lambda/EC2) |  | (Sink to S3/     |       |
|  | Own IAM Role     |  | Own IAM Role     |  |  Redshift/OS)    |       |
|  +------------------+  +------------------+  +------------------+       |
|                                                                         |
|  Security Group: Allow 9098 inbound from authorized client SGs only     |
|  S3 Gateway Endpoint: Tiered storage + Connect S3 sink                  |
|  DynamoDB Gateway Endpoint: Already provisioned                         |
|                                                                         |
+-----------------------------------------------------------------------+
            |
            | (TGW routing for cross-VPC traffic)
            |
+-----------------------------------------------------------------------+
|              TRANSAMERICA NETWORK CONTROL BOUNDARY                      |
+-----------------------------------------------------------------------+
|                                                                         |
|  Cross-VPC Consumer Access (same region, same environment):             |
|  Consumer VPC --> TGW --> Inspection VPC (Palo Alto) --> TGW -->        |
|  MSK VPC (Port 9098)                                                    |
|                                                                         |
|  Cross-Region Replication (MSK Replicator / DR):                        |
|  Source VPC --> TGW --> Inspection VPC --> TGW --> Edge VPC              |
|  (Aviatrix Gateway) --> SECURE TUNNEL --> Dest Edge VPC                 |
|  (Aviatrix Gateway) --> TGW --> Dest Inspection VPC --> TGW -->         |
|  Dest MSK VPC                                                           |
|                                                                         |
|  Cross-Environment Access (Prod ↔ Non-Prod):                            |
|  Same path as cross-region (Aviatrix encrypted tunnel between envs)     |
|                                                                         |
|  AWS API Access (kafka:* management plane):                             |
|  Admin workstation --> TGW --> Browsing Service --> AWS Public API       |
|                                                                         |
+-----------------------------------------------------------------------+
```

### Design Considerations

* **Network Classification**: Category 2 — Customer-Managed VPC Resource. MSK broker ENIs reside in customer-managed private subnets. Subject to Security Groups, NACLs, TGW routing, and Inspection VPC firewalls.
* **Broker Deployment**: Minimum 3 brokers across 3 AZs for production. Express Brokers recommended (pre-hardened security, auto-scaling, faster recovery).
* **Client Connectivity**: Producers/consumers connect to brokers on port 9098 (IAM + TLS). Clients must be in the same VPC or reachable via TGW → Inspection VPC routing.
* **Cross-VPC Access**: Consumer applications in other VPCs access MSK via TGW → Inspection VPC → TGW path. Palo Alto firewall policies must permit port 9098 traffic.
* **No Public Access**: MSK Public Access feature is incompatible with Transamerica architecture (no IGWs, no public subnets). If internet-facing streaming is required, use the Hosting Service pattern with a REST-to-Kafka translation layer.
* **API Access**: Management plane operations (`kafka:*`) accessed via TGW → Browsing Service → AWS Public API. PrivateLink for MSK APIs available but used only when performance requires it.
* **High Availability**: Multi-AZ replication automatic. Express Brokers recover 90% faster than Standard. 99.9% SLA for Provisioned clusters.
* **Storage**: Express Brokers use unlimited pay-as-you-go managed storage. Standard Brokers use customer-specified EBS. Tiered storage offloads to S3 for cost-effective retention.

## Data Privacy & Protection

### Data Classification

| Classification           | Regulatory          |
| ------------------------ | ------------------- |
| ☒ Strictly Confidential  | ☒ Financial         |
| ☒ Confidential           | ☒ PII               |
| ☒ Internal               | ☐ PHI               |
| ☐ Public                 | ☐ PCI               |
| ☐ No Data                | ☐ No Personal Data  |

Note: MSK is a data transport platform. Classification depends on topic content. Each topic must be individually classified based on the data it carries. The above reflects the maximum potential classification — individual topics will vary.

### OneTrust Integration

* **Asset Discovery Questionnaire:** `[Link Required — must be completed for production clusters carrying PII/NPI]`

### Requirements & Guardrails

| # | Requirement                                              | Configuration / Guardrail                                                                                      |
| - | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 1 | Data classification documented per topic                 | Kafka topics must be tagged with DataClassification metadata. Naming convention enforces classification prefix. |
| 2 | CMK encryption mandatory                                 | SCP enforces customer-managed KMS key at cluster creation. Default `aws/msk` key prohibited.                   |
| 3 | DPIA required for topics with PII/NPI/PHI                | Data Protection Impact Assessment completed before publishing personal data to any topic                       |
| 4 | Topic retention aligned to data classification           | `retention.ms` configured per topic based on regulatory requirements (NYDFS: 7 years for financial data)       |
| 5 | No sensitive data in topic names                         | Topic naming conventions must not expose PII or business-sensitive terms                                        |
| 6 | Secrets Manager for SCRAM credentials                    | If SASL/SCRAM is used (non-IAM auth), credentials must be in Secrets Manager with rotation                     |

### Additional Considerations

* Messages are the customer's responsibility to classify — MSK has no native data classification engine
* Application-layer DLP/masking must be implemented at the producer before publishing sensitive data
* Consumer applications must demonstrate need-to-know authorization for topic content
* Topic data retention: define per data class. Internal: 7 days default. Financial/regulatory: align with NYDFS 500 retention.
* Data residency: all MSK resources restricted to us-east-1, us-east-2, and eu-west-1 via SCP

## Well-Architected Framework

### Security Pillar

| # | Requirement                                     | Configuration / Guardrail                                                                                   |
| - | ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 1 | IAM authentication enforced                     | `ClientAuthentication.Sasl.Iam = true`; SCP denies cluster creation with `unauthenticatedAccess = true`     |
| 2 | TLS-only client-to-broker communication         | `EncryptionInTransit.ClientBroker = TLS`; SCP denies PLAINTEXT or TLS_PLAINTEXT                             |
| 3 | Customer-managed KMS key                        | CMK specified at cluster creation; SCP denies default `aws/msk` key                                        |
| 4 | Topic-level IAM authorization                   | Each producer/consumer has dedicated IAM role with topic-specific `kafka-cluster:*` resource ARNs           |
| 5 | Security Groups restrict to port 9098           | Inbound: only from authorized client Security Groups on port 9098. No CIDR rules. No 0.0.0.0/0.            |
| 6 | Public access disabled                          | SCP denies `kafka:UpdateConnectivity` with public access. Security Hub MSK.4 validates.                     |
| 7 | CloudTrail logging for management plane         | All `kafka:*` API calls logged via CloudTrail → Splunk                                                      |
| 8 | Broker logs to Splunk                           | Broker logs forwarded via Kinesis Firehose to Splunk. Authentication failures alerting enabled.              |
| 9 | SSO via PingOne                                 | Console access via PingOne SAML/OIDC. Programmatic access via federated IAM roles.                          |
| 10| MSK Connect plugin scanning                    | Custom connector JARs vulnerability-scanned before upload to S3. No unscanned plugins in production.         |

**Key Areas Addressed:**
* Identity & Access: IAM authentication with topic-level authorization; PingOne SSO for console
* Authentication: IAM (primary), mTLS (optional), SASL/SCRAM via Secrets Manager (optional)
* Logging & Auditing: CloudTrail (management), broker logs (data plane), VPC Flow Logs (network) → all to Splunk
* Network Security: Security Groups, TGW routing, Inspection VPC firewalls, no public endpoints

### Cost Optimization Pillar

| # | Requirement                                | Configuration / Guardrail                                                                            |
| - | ------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| 1 | Right-size broker instance type            | Express Brokers auto-scale; Standard Brokers sized to workload with monitoring for over-provisioning |
| 2 | Tiered storage for cost-effective retention| Enable tiered storage for topics with >7 day retention to offload cold data to S3                    |
| 3 | AWS Budgets configured                     | Monthly budget alerts at 80% forecasted and 100% actual for MSK service costs                        |
| 4 | Cost allocation via tagging                | All MSK resources tagged with CostCenter, Application, Environment, DataClassification               |
| 5 | Partition count optimization               | Size partitions to actual throughput needs; avoid over-partitioning which increases broker load       |

**Pricing Model**: 
* Express Brokers: per broker-hour + per throughput unit + per storage GB
* Standard Brokers: per broker-hour + per EBS GB provisioned
* MSK Connect: per worker-hour
* No upfront commitment required

**Ownership**: Application team owns MSK cluster costs. ET owns VPC infrastructure and TGW costs.

### Operational Excellence Pillar

| # | Requirement                                     | Configuration / Guardrail                                                                         |
| - | ----------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | Infrastructure as Code (IaC)                    | All MSK clusters, topics, and IAM policies managed via Terraform                                  |
| 2 | Kafka version pinning                           | Pin specific Kafka version (e.g., `3.7.x.kraft`); upgrades follow change management               |
| 3 | Automated health monitoring                     | CloudWatch alarms on `ActiveControllerCount`, `UnderReplicatedPartitions`, `OfflinePartitionsCount`|
| 4 | Consumer lag monitoring                         | CloudWatch alarms on consumer group lag; sustained lag triggers investigation                      |
| 5 | MSK Connect connector monitoring                | CloudWatch alarms on connector health status, task failures, and restart counts                    |
| 6 | Runbook for MSK-specific incidents              | Document broker failure, under-replicated partitions, consumer lag spikes, connector failures      |
| 7 | Schema Registry enforcement                     | Glue Schema Registry for Avro/JSON schema validation. Breaking schema changes require review.      |

**Key Considerations:**
* Automation: IaC for all infrastructure; topic creation via pipeline or approved Terraform
* Monitoring: CloudWatch enhanced monitoring + Prometheus-compatible metrics for broker health
* Incident Management: Alert on controller election, partition offline, authentication failures
* CI/CD Alignment: MSK Connect plugin JARs deployed via CI/CD pipeline with vulnerability scanning

### Reliability Pillar

| # | Requirement                                     | Configuration / Guardrail                                                                         |
| - | ----------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | Multi-AZ deployment (3 AZs)                     | Minimum 3 brokers across 3 AZs. Replication factor ≥ 3 for production topics.                    |
| 2 | In-sync replica minimum                         | `min.insync.replicas = 2` for production topics to prevent data loss                              |
| 3 | Express Brokers for production                  | 90% faster recovery; automatic scaling; eliminates capacity planning failures                     |
| 4 | Idempotent producers                            | `enable.idempotence = true` for exactly-once semantics on critical topics                         |
| 5 | Consumer group resilience                       | Consumers designed for rebalancing; dead-letter topics for processing failures                    |

### Disaster Recovery Strategy

* **RTO**: < 1 hour (Express Brokers); < 4 hours (Standard Brokers)
* **RPO**: Near-zero within region (synchronous multi-AZ replication). Cross-region RPO depends on MSK Replicator lag.
* **Cross-Region DR**: Evaluate MSK Replicator between us-east-1 and us-east-2. Replication traffic traverses Aviatrix encrypted tunnels (Source VPC → TGW → Inspection VPC → TGW → Edge VPC → Aviatrix Tunnel → Destination Edge VPC → TGW → Inspection VPC → TGW → Destination VPC).
* **Testing**: Quarterly failover testing for production clusters with documented results.

### Performance Efficiency Pillar

| # | Requirement                                     | Configuration / Guardrail                                                                         |
| - | ----------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | Express Brokers for auto-scaling                | Eliminates manual capacity planning; automatic throughput scaling based on load                   |
| 2 | Partition strategy                              | Size partitions for parallel consumer processing; align with consumer group capacity              |
| 3 | Compression at producer                         | Enable `lz4` or `zstd` compression to reduce network and storage overhead                        |
| 4 | Batch size tuning                               | Configure `batch.size` and `linger.ms` for throughput vs. latency tradeoff per use case          |

### Sustainability Pillar

| # | Requirement                                     | Configuration / Guardrail                                                                         |
| - | ----------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | Region selection: approved regions only         | us-east-1, us-east-2, eu-west-1 only. Avoid unnecessary cross-region data movement.              |
| 2 | Tiered storage for efficient resource usage     | Offload cold data to S3; reduces EBS provisioning and broker memory pressure                      |
| 3 | Right-size partitions                           | Avoid over-partitioning. Fewer partitions = less metadata overhead, faster recovery.              |
| 4 | Topic retention optimization                    | Set retention to minimum required by business/regulatory needs. Don't retain indefinitely.        |
| 5 | Express Brokers auto-scaling                    | Scales down during low-traffic periods; avoids always-on over-provisioned capacity.               |

## Compliance & Risk

| # | IT Control                                      | Configuration / Guardrail                                                                         |
| - | ----------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | ID 129 - Logging & Monitoring                   | CloudTrail for API + broker logs via Firehose → Splunk. Authentication failure alerting.          |
| 2 | ID 115 - Encryption Solution                    | AES-256 at rest via CMK (always on); TLS 1.2 in transit for all broker communication             |
| 3 | ID 79 - Administrative Access                   | Cluster admin (`kafka:*`) separated from data plane (`kafka-cluster:*`). Least-privilege IAM.    |
| 4 | ID 98 - Change Management                       | Cluster config changes, version upgrades, topic creation via IaC with change approval             |
| 5 | ID 74 - Access Entitlement Review               | Quarterly Sailpoint review of IAM roles with `kafka-cluster:*` permissions                       |
| 6 | ID 103 - Data Privacy                           | Topic-level data classification. DPIA for PII/NPI/PHI topics. OneTrust registration.             |
| 7 | ID 147 - Regulatory Requirements                | TLS 1.2 enforced; AES-256; NYDFS 500 compliant. HIPAA eligible with BAA.                        |
| 8 | ID 102 - Data Loss Prevention                   | Application-layer masking/tokenization before publishing sensitive data. No native DLP.           |
| 9 | ID 137 - Network Security Controls              | Security Groups on broker ENIs. TGW routing. Inspection VPC firewalls. No public endpoints.      |
| 10| ID 123 - High Availability                      | Multi-AZ (3 AZs). 99.9% SLA. Express Brokers for faster recovery. MSK Replicator for DR.        |
| 11| ID 172 - Monitor Noncompliance                  | Security Hub MSK.1–MSK.6 controls. AWS Config rules. Non-compliance alerts → Splunk.             |
| 12| ID 173 - Generative AI                          | Not applicable — MSK is not an AI/ML service. No AI data sharing.                                |

### Additional Considerations

* MSK is SOC 1/2/3, ISO 27001, PCI DSS, HIPAA eligible, FedRAMP Moderate
* Inter-region replication traffic encrypted by Aviatrix tunnels satisfying NYDFS 500.15(a)
* Audit trail: all cluster operations via CloudTrail; all data plane activity via broker logs
* No AI/ML opt-out required — MSK does not share customer data for service improvement

## Additional Guardrails

### SCP: Block Public Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyMSKPublicAccess",
      "Effect": "Deny",
      "Action": "kafka:UpdateConnectivity",
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringEquals": {
          "kafka:publicAccess": "SERVICE_PROVIDED_EIPS"
        }
      }
    }
  ]
}
```

### SCP: Enforce TLS-Only Communication

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPlaintext",
      "Effect": "Deny",
      "Action": ["kafka:CreateCluster", "kafka:CreateClusterV2", "kafka:UpdateSecurity"],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringEquals": {
          "kafka:ClientBroker": ["PLAINTEXT", "TLS_PLAINTEXT"]
        }
      }
    }
  ]
}
```

### SCP: Enforce Authentication

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnauthenticated",
      "Effect": "Deny",
      "Action": ["kafka:CreateCluster", "kafka:CreateClusterV2"],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "kafka:unauthenticatedAccess": "true"
        }
      }
    }
  ]
}
```

### SCP: Restrict to Approved Transamerica Regions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyMSKOutsideApprovedRegions",
      "Effect": "Deny",
      "Action": "kafka:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-east-2", "eu-west-1"]
        }
      }
    }
  ]
}
```

### SCP: Enforce Customer-Managed KMS Key

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDefaultMSKKey",
      "Effect": "Deny",
      "Action": ["kafka:CreateCluster", "kafka:CreateClusterV2"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kafka:EncryptionAtRestKmsKeyId": ""
        }
      }
    }
  ]
}
```

### Additional Preventative Controls

* Prevent public access: SCP + Security Hub MSK.4 (defense-in-depth with no-IGW architecture)
* Enforce TLS: SCP denies plaintext; Security Hub MSK.1 validates
* Enforce authentication: SCP denies unauthenticated; Security Hub MSK.6 validates
* Restrict Multi-VPC connectivity: SCP restricts `kafka:PutClusterPolicy` and `kafka:UpdateConnectivity` to admin OU paths
* Restrict regions: SCP limits to us-east-1, us-east-2, eu-west-1
* Restrict deployment roles: Cluster create/update/delete limited to approved admin roles
* Enable automated compliance: Security Hub MSK.1–MSK.6 + AWS Config + Splunk alerting

## Waivers and Exceptions

### Risk Acceptances

| # | Title                                       | Description                                                                                        | Link |
| - | ------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---- |
| 1 | TLS 1.2 (not 1.3) for Kafka protocol       | Apache Kafka wire protocol supports TLS 1.2 maximum. TLS 1.3 not available for broker communication. Accepted as protocol limitation. | |
| 2 | No native DLP in MSK                        | MSK has no built-in data masking or tokenization. Accepted: implemented at application layer (producer-side). | |
| 3 | AWS-managed metadata storage                | Cluster metadata (ZooKeeper/KRaft) managed by AWS. No direct customer access or backup control. Accepted: AWS SOC2 covers. | |

### Risk Remediations

| # | Title                                       | Description                                                                                        | Link |
| - | ------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---- |
| 1 | Sailpoint integration for kafka-cluster roles| Quarterly entitlement review of `kafka-cluster:*` IAM permissions via Sailpoint. Target: Q4 2026. | |
| 2 | Application-layer DLP for production topics | Implement producer-side tokenization/masking for topics carrying PII/NPI. Target: before production go-live. | |
| 3 | Glue Schema Registry enforcement            | Mandatory schema validation for production topics to prevent schema drift and data quality issues. Target: Q1 2027. | |

**Note:** Risk acceptance/remediation must be **approved prior to ARB submission**.

## Approved Regions

### AWS

* ☒ AWS N. Virginia (`us-east-1`)
* ☒ AWS Ohio (`us-east-2`)
* ☒ AWS Ireland (`eu-west-1`)

Source: Transamerica-Organizational-Alignment steering document. MSK is available in all three Transamerica baseline regions.

## IAM / RBAC Configuration

### IAM Namespaces: `kafka` (management) + `kafka-cluster` (data plane)

### Role Taxonomy

| Role Name Pattern              | Purpose                                                | Environment       |
| ------------------------------ | ------------------------------------------------------ | ----------------- |
| `msk-admin-*`                  | Create/manage MSK clusters, update configs, topics     | All               |
| `msk-producer-*`               | Publish messages to specific topics                    | All               |
| `msk-consumer-*`               | Consume messages from specific topics                  | All               |
| `msk-connect-execution-*`      | MSK Connect connector execution role                   | All               |
| `msk-monitor-*`                | Read-only cluster monitoring and metrics               | All               |

### Key IAM Actions

| Action Category         | Actions                                                                  | Permitted Roles              |
| ----------------------- | ------------------------------------------------------------------------ | ---------------------------- |
| Cluster Management      | `kafka:CreateCluster`, `UpdateClusterConfiguration`, `DeleteCluster`     | `msk-admin-*`                |
| Topic Management        | `kafka-cluster:CreateTopic`, `DeleteTopic`, `AlterTopic`                 | `msk-admin-*`                |
| Produce Messages        | `kafka-cluster:Connect`, `kafka-cluster:WriteData`, `kafka-cluster:DescribeTopic` | `msk-producer-*`   |
| Consume Messages        | `kafka-cluster:Connect`, `kafka-cluster:ReadData`, `kafka-cluster:DescribeTopic`, `kafka-cluster:AlterGroup`, `kafka-cluster:DescribeGroup` | `msk-consumer-*` |
| Monitoring              | `kafka:DescribeCluster`, `kafka:ListClusters`, `kafka:GetBootstrapBrokers` | `msk-monitor-*`, `msk-admin-*` |
| Connect Management      | `kafkaconnect:CreateConnector`, `UpdateConnector`, `DeleteConnector`     | `msk-admin-*`                |

### Topic-Level Authorization Example

```
Resource ARN: arn:aws:kafka:{region}:{account}:topic/{cluster-name}/{cluster-uuid}/{topic-name}
```

Each producer/consumer application receives an IAM role scoped to specific topic name patterns (e.g., `orders.*`, `payments.*`). Wildcard topic access (`*`) is prohibited in production.

### Key SCP Policies (Summary)

1. **Block Public Access** — Deny `kafka:UpdateConnectivity` with public access
2. **Enforce TLS** — Deny PLAINTEXT or TLS_PLAINTEXT client broker communication
3. **Enforce Authentication** — Deny unauthenticated cluster creation
4. **Restrict Regions** — us-east-1, us-east-2, eu-west-1 only
5. **Enforce CMK** — Deny default KMS key
6. **Restrict Multi-VPC Connectivity** — Limit `PutClusterPolicy` to admin OU paths

## ARB Approval Status

* ☒ Approved with Restrictions

**Restrictions:**
1. **Express Brokers mandatory for production** — Standard Brokers acceptable for non-production/development only. Production use of Standard Brokers requires an architecture exception with documented justification.
2. **TLS-only communication enforced** — Plaintext and TLS_PLAINTEXT modes prohibited in all environments via SCP. No exceptions.
3. **IAM authentication mandatory** — Unauthenticated cluster access prohibited in all environments via SCP.
4. **Customer-managed KMS key required** — Default `aws/msk` key prohibited. CMK with rotation enabled.
5. **No public access** — MSK Public Access feature prohibited (incompatible with architecture). No exceptions without Hosting Service pattern.
6. **Data classification required** — Each production topic must have documented data classification. Topics carrying PII/NPI/PHI require DPIA before production.
7. **Topic-level IAM authorization** — No wildcard topic access (`*`) in production. Each application scoped to its specific topics.
8. **Multi-VPC PrivateLink requires exception** — Multi-VPC Private Connectivity (PrivateLink Endpoint Services) requires ISO/DPO/ARB approval before activation.

## Final Notes

* Component must comply with **all defined guardrails**
* Each solution must still undergo **solution-level validation**
* Exceptions must follow **formal risk treatment process**
* MSK is a data transport — the content flowing through topics determines the regulatory obligations. The cluster infrastructure is hardened by default; the data governance responsibility lies with producers and consumers.
* Security Hub MSK.1–MSK.6 controls must report compliant status before production go-live
* Cross-VPC consumer access requires Palo Alto firewall policy permitting port 9098 between source and destination VPCs — submit network request to Cloud Networking (ET)
* Cross-region replication (MSK Replicator) traffic traverses Aviatrix encrypted tunnels — coordinate with Cloud Networking for routing setup

---

## References

* [Amazon MSK Encryption](https://docs.aws.amazon.com/msk/latest/developerguide/msk-encryption.html)
* [Amazon MSK Data Protection](https://docs.aws.amazon.com/msk/latest/developerguide/data-protection.html)
* [Amazon MSK IAM Access Control](https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html)
* [Amazon MSK Infrastructure Security](https://docs.aws.amazon.com/msk/latest/developerguide/infrastructure-security.html)
* [Amazon MSK Express Brokers](https://docs.aws.amazon.com/msk/latest/developerguide/msk-broker-types-express.html)
* [Amazon MSK Public Access](https://docs.aws.amazon.com/msk/latest/developerguide/public-access.html)
* [Amazon MSK Multi-VPC Private Connectivity](https://docs.aws.amazon.com/msk/latest/developerguide/aws-access-mult-vpc.html)
* [Amazon MSK Security Hub Controls](https://docs.aws.amazon.com/securityhub/latest/userguide/msk-controls.html)
* [Amazon MSK Replicator](https://docs.aws.amazon.com/msk/latest/developerguide/msk-replicator.html)
* [Amazon MSK Pricing](https://aws.amazon.com/msk/pricing/)
* [Best Practices for Least Privilege in MSK](https://aws.amazon.com/blogs/big-data/best-practices-for-least-privilege-configuration-in-amazon-mwaa)
* [Express Brokers GA (November 2024)](https://aws.amazon.com/blogs/aws/introducing-express-brokers-for-amazon-msk-to-deliver-high-throughput-and-faster-scaling-for-your-kafka-clusters/)
* Internal: Amazon_MSK_Security_Evaluation.md (Namespace Analysis)
* Internal: Transamerica-Organizational-Alignment steering document
