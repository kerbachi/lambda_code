# Security Evaluation: AWS IAM Namespace "kafka" (Amazon Managed Streaming for Apache Kafka)

## Service Security Assessment

Assessment Date: June 22, 2026 (Updated: July 16, 2026 — Production Assessment added, regions aligned to Transamerica baseline)

## 1. Service Overview

Amazon Managed Streaming for Apache Kafka (Amazon MSK) is a fully managed service for building and running applications that use Apache Kafka to process streaming data. The service handles broker provisioning, configuration, patching, and high availability across multiple Availability Zones. Brokers are deployed inside customer VPCs.

The IAM namespace is `kafka` (management plane) and `kafka-cluster` (data plane / Kafka API operations).

**Deployment Models:**
- **MSK Provisioned – Standard Brokers**: Customer selects instance types and EBS storage
- **MSK Provisioned – Express Brokers**: 3x throughput, 20x faster scaling, unlimited pay-as-you-go storage, pre-configured security best practices
- **MSK Serverless**: Fully automatic provisioning and scaling

**Network Classification**: Category 2 — Customer-Managed VPC Resource. MSK clusters run inside customer VPCs subject to Security Groups, NACLs, Transit Gateway routing, and Inspection VPC firewalls. All cross-VPC traffic routes through TGW → Inspection VPC → TGW. Cross-region or cross-environment traffic additionally traverses Aviatrix encrypted tunnels via Edge VPCs.

**Regional Availability (Transamerica Intersection):**

MSK is available in all three Transamerica baseline-approved AWS regions:

| Region | Code | Transamerica Approved |
|--------|------|----------------------|
| US East (N. Virginia) | `us-east-1` | ☒ Yes |
| US East (Ohio) | `us-east-2` | ☒ Yes |
| Europe (Ireland) | `eu-west-1` | ☒ Yes |

Source: Transamerica-Organizational-Alignment steering document (authoritative region baseline).

### Non-Production Assessment Opinion

Amazon MSK is appropriate for use in non-production environments. The risks are generally controllable through standard IAM policies, Security Groups, and encryption defaults. Express Brokers (recommended) enforce TLS and IAM authentication by default, eliminating the most common misconfiguration risks. In a non-production network where access to production data is restricted, the residual risk is acceptable provided the SCPs documented in this evaluation are applied to prevent plaintext communication and unauthenticated cluster creation. The service does not require internet connectivity for core operations, does not share data for AI/ML purposes, and runs entirely within the private VPC — all of which align well with non-production use.

### Production Assessment Opinion

Amazon MSK is appropriate for production use with additional controls. As a Category 2 service running inside customer VPCs, MSK benefits from the full depth of Transamerica network security controls (Security Groups, Inspection VPC firewalls, Aviatrix encryption for cross-environment traffic). Production deployment requires the following additional considerations:

1. **Express Brokers mandatory for production** — Enforces TLS + IAM by default, eliminating plaintext/unauthenticated misconfiguration risk. Standard Brokers should not be used in production without an approved architecture exception.
2. **Customer-managed KMS key required** — The default `aws/msk` key is insufficient for production. CMK provides key rotation, access auditing, and key policy control.
3. **Topic-level IAM authorization** — Each producer/consumer application must have its own IAM role scoped to specific topics. No shared roles or wildcard topic access in production.
4. **Data classification per topic** — Topics carrying PII, NPI, or financial data must be inventoried, tagged, and subject to DPIA. Retention policies must align with NYDFS 500 requirements.
5. **Application-layer data protection** — MSK has no native DLP or data masking. Producers must implement tokenization/masking before publishing sensitive data. Consumers must have approved need-to-know for topic content.
6. **MSK Connect connector security** — Production connectors require vulnerability scanning of plugin JARs, least-privilege execution roles, and code review before deployment.
7. **Multi-AZ + DR strategy** — Production clusters must document RTO/RPO. MSK Replicator should be evaluated for cross-region DR between us-east-1 and us-east-2. Cross-region replication traffic flows through Aviatrix encrypted tunnels (Source VPC → TGW → Production Inspection VPC → TGW → Production Edge VPC → Aviatrix Secure Tunnel → Destination Edge VPC → TGW → Destination Inspection VPC → TGW → Destination VPC).
8. **Consumer lag monitoring** — Production topic consumers must have CloudWatch alarms on consumer lag metrics. Sustained lag indicates data processing delays that may impact SLAs.
9. **Security Hub MSK.1–MSK.6 controls** — All Security Hub MSK controls must report compliant status before production go-live.
10. **Splunk integration mandatory** — Broker logs, authentication events, and CloudTrail must flow to Splunk for SIEM monitoring. Alert on authentication failures, unauthorized topic access attempts, and configuration changes.
11. **Change management** — All production cluster configuration changes (scaling, version upgrades, topic creation) must follow GTS Enterprise Change Management process.
12. **Quarterly entitlement review** — IAM roles with `kafka-cluster:*` permissions reviewed via Sailpoint quarterly. Unused or overly-broad permissions must be remediated.

With these controls in place, MSK is suitable for production streaming workloads including event-driven architectures, real-time data pipelines, and CDC (Change Data Capture) patterns.

## 2. Security Evaluation Against Criteria

### 2.1 Does the service itself store data?

**Yes.** MSK persists message data (topics, partitions, consumer offsets) on EBS volumes within the customer VPC. Express Brokers use AWS-managed pay-as-you-go storage. Tiered storage can offload cold data to S3.

Data remains in the AWS Region where the cluster is created. It does not leave approved Transamerica regions when clusters are provisioned in us-east-1, us-east-2, or eu-west-1. Cross-region replication only occurs if MSK Replicator is explicitly configured, and cross-region traffic would traverse Aviatrix encrypted tunnels through Edge VPCs.

**Risk Level: Medium**

### 2.2 If it stores data, does it encrypt it?

**Yes — encryption at rest is always enabled and cannot be disabled.**

- Default: AWS-managed key (`aws/msk`)
- Customer-managed KMS key (CMK) supported at cluster creation
- Tiered storage inherits the same KMS configuration

Encryption in transit:
- Broker-to-broker: TLS 1.2 by default
- Client-to-broker: configurable (TLS, TLS_PLAINTEXT, or PLAINTEXT)
- Express Brokers: TLS enforced; cannot be weakened

Can resources be created without encryption at rest? **No.** Without encryption in transit? **Yes** — Standard clusters allow `ClientBroker = PLAINTEXT`.

**Risk Level: Low**

### 2.3 What security measures protect the data?

- **Authentication**: IAM access control (recommended), mutual TLS, SASL/SCRAM via Secrets Manager
- **Authorization**: IAM policies with topic-level granularity; Kafka ACLs also supported
- **Network Segmentation**: Private subnets only; Security Groups restrict port access; NACLs managed by ET
- **VPC Isolation**: Dedicated broker instances per cluster (Provisioned)
- **MFA**: Not native to Kafka protocol; inherited from IAM principal chain
- **Data Masking / Anonymization**: Not provided natively — application-layer responsibility
- **Secrets Management**: SCRAM credentials in Secrets Manager with rotation

**Risk Level: Medium**

### 2.4 Does the service process data (NYDFS / GDPR)?

**Yes.** MSK is a data transport and persistence layer. It processes whatever producers publish, which may include PII, financial records, or PHI.

Amazon MSK is in scope for FedRAMP Moderate, SOC 1/2/3, PCI DSS, HIPAA eligibility, ISO 27001. Within our architecture, inter-region traffic traverses Aviatrix encrypted tunnels, satisfying NYDFS 500.15(a) requirements.

**Risk Level: Medium**

### 2.5 Does the service have access to data in other services?

**Conditionally.** The cluster itself does not reach into DynamoDB, RDS, or S3. However:
- MSK Connect source/sink connectors can read/write to S3, DynamoDB, RDS, OpenSearch
- MSK Replicator replicates data between MSK clusters
- Consumer applications downstream may access any data store

MSK Connect workers run in the customer VPC and follow standard TGW/Inspection VPC routing for cross-VPC access.

**Risk Level: Medium**

### 2.6 Is transmission secured through minimum TLS 1.3/HTTPS?

**Partially.** Kafka wire protocol uses TLS 1.2. AWS management APIs support TLS 1.3.

| Path | Minimum TLS |
|------|-------------|
| Broker-to-broker | TLS 1.2 |
| Client-to-broker | TLS 1.2 (configurable) |
| MSK Connect worker-to-broker | TLS 1.2 |
| AWS Management API | TLS 1.2 (1.3 supported) |
| Console/browser | TLS 1.3 via Browsing Service |

Per Transamerica security requirements, browser traffic on the network should use TLS 1.3 at a minimum. The Kafka wire protocol (broker-to-broker, client-to-broker) uses TLS 1.2 — this is an accepted limitation of the Apache Kafka protocol. Within our environment, all broker traffic stays in private VPCs. Cross-region or cross-environment flows are additionally encrypted by Aviatrix tunnels through Edge VPCs.

**Risk Level: Medium**

### 2.7 Is minimum TLS 1.3 enabled by default? Can resources be created without TLS?

**No**, TLS 1.3 is not available for Kafka protocol traffic. TLS 1.2 is the default.

- Express Brokers: TLS + IAM enforced by default; cannot be disabled
- Standard Brokers: can be created with `ClientBroker = PLAINTEXT`

**Risk Level: Medium-High**

### 2.8 Can it transmit data outside our AWS network?

**Yes.** Cross-account mechanisms:
- Multi-VPC Private Connectivity (PrivateLink) — requires cluster policy + exception approval
- MSK Connect sink connectors — can write to other accounts if IAM permits
- MSK Replicator — same-account only; cross-account requires MirrorMaker 2 via Connect

In our environment: PrivateLink Endpoint Services are restricted (ISO/DPO/ARB approval required). Internal cross-account traffic flows through TGW → Inspection VPC → TGW, subject to Palo Alto firewall policy. Third-party access routes through the Third Party Inspection Service. Cross-region replication traffic additionally traverses Aviatrix encrypted tunnels via Edge VPCs.

**Risk Level: Medium-High**

### 2.9 Is it designed to link external entities?

**Not by design, but conditionally capable.**
- MSK Connect can reach SaaS endpoints via Browsing Service (requires firewall rule approval)
- Multi-VPC connectivity creates PrivateLink connections (requires exception)
- Public Access feature exists but is incompatible with our architecture (no Internet Gateways)

**Risk Level: Medium-High**

### 2.10 Does it create a public interface?

**Optionally.** MSK has a "Public Access" feature (disabled by default, requires IGW which we don't have). Security Hub control MSK.4 verifies it is disabled.

In our environment this is **fully incompatible** — no IGWs, no public subnets. If internet-facing Kafka access is needed, the Hosting Service pattern applies (Imperva WAF → DDoS protection → AWS ALB → Palo Alto Next-Generation Firewalls → TGW → application VPC with a REST-to-Kafka translation layer). This would require onboarding through the Cloud Hosting Service process with appropriate approvals.

**Risk Level: Medium-High**

### 2.11 Is it a cloud management tool?

**No.** MSK is a data streaming platform. Governable via Config, Security Hub, SCPs, Terraform, CloudWatch, Control Tower.

**Risk Level: Low**

### 2.12 Does it require a contract agreement?

**No.** Pay-as-you-go pricing across all deployment models. No contracts or upfront commitments required.

**Risk Level: Low**

### 2.13 Does it require Marketplace items?

**No.** Native AWS service. MSK Connect plugins are uploaded as JAR files from S3 — no Marketplace procurement.

**Risk Level: Low**

### 2.14 Does it send data for service improvement?

**No.** MSK is not an AI/ML service. Not listed in the AWS AI services opt-out policy. Customer message content is never used for service improvement.

**Risk Level: Low**

### 2.15 Risk Classification

**Overall: Medium.** Moderate risk mitigated by VPC deployment, always-on encryption at rest, and network controls. Primary risks are misconfiguration (plaintext, unauthenticated access) on Standard brokers, and connector-based data exfiltration — both addressable via SCPs and IAM.

### 2.16 Non-Production Use

For non-production environments where access to production data is restricted:
- The core risk surface (network isolation, encryption at rest) is acceptable by default
- Express Brokers eliminate misconfiguration risks that affect Standard brokers
- SCPs should still enforce TLS and authentication to prevent accidental plaintext clusters
- Topic-level IAM authorization prevents lateral data access even within non-production
- No internet connectivity required for core streaming operations

**Risk Level: Low** — Acceptable for non-production use with standard controls applied.

## 3. Summary Risk Profile

| # | Criteria | Finding | Risk Level |
|---|----------|---------|------------|
| 1 | Data Storage | Yes — EBS/managed in VPC; tiered to S3; US-only | Medium |
| 2 | Encryption | Always on at rest (CMK); TLS 1.2 in transit | Low |
| 3 | Security Measures | IAM/mTLS/SCRAM; VPC segmentation; no native masking | Medium |
| 4 | Data Processing | Processes streaming data; FedRAMP/SOC/PCI/HIPAA | Medium |
| 5 | Data Sources | MSK Connect can access external data stores | Medium |
| 6 | TLS/HTTPS | TLS 1.2 Kafka protocol; TLS 1.3 management API | Medium |
| 7 | TLS Default | TLS 1.2 default; plaintext possible on Standard brokers | Medium-High |
| 8 | Cross-Account | Multi-VPC PrivateLink, Connect, cluster policies | Medium-High |
| 9 | External Linking | Connect SaaS egress, public access, PrivateLink | Medium-High |
| 10 | Public Interface | Feature exists but incompatible with architecture | Medium-High |
| 11 | Management Tool | No — data platform | Low |
| 12 | Contract | No — pay-as-you-go | Low |
| 13 | Marketplace | No — native service | Low |
| 14 | Data Sharing | No — not AI/ML | Low |
| 15 | Risk Classification | Moderate; controllable via SCPs and IAM | Medium |
| 16 | Non-Production | Acceptable with standard controls | Low |

## 4. Security Best Practices & Recommendations

1. **Use Customer-Managed KMS Keys** — Specify a CMK at cluster creation. Enable annual automatic rotation. Do not rely on the default `aws/msk` key.
2. **Enforce TLS-Only Communication** — Set `ClientBroker = TLS`. Block PLAINTEXT via SCP. Prefer Express Brokers.
3. **Enable IAM Access Control** — Use `kafka-cluster:*` actions with topic-level resource ARNs for fine-grained authorization.
4. **Disable Unauthenticated Access** — SCP to deny cluster creation with `unauthenticatedAccess = true`.
5. **Block Public Access via SCP** — Deny `kafka:UpdateConnectivity` for public access. Defense-in-depth since no IGWs exist.
6. **Restrict Cross-Account Access** — No cluster resource policies without ISO/DPO/ARB approval. Monitor `PutClusterPolicy` via CloudTrail.
7. **Restrict MSK Connect Roles** — Least-privilege IAM for connector execution roles. Deny cross-account resource access by default.
8. **AI/ML Data Opt-Out** — Not applicable; MSK is not an AI service.
9. **Disable Anonymous Access** — Enforce authentication. Security Hub MSK.6 validates this.
10. **Enable CloudTrail Logging** — Already enforced by ET. Covers all `kafka:*` management operations.
11. **Enable SIEM Logging (Splunk)** — Route broker logs via Firehose to Splunk. Include auth failures and config changes.
12. **Enable Application Logging (Neptune)** — Route producer/consumer application logs to Neptune for operational analysis.
13. **Apply SCPs** — Block: public access, plaintext, unauthenticated, default KMS, unrestricted multi-VPC connectivity.
14. **Use VPC Endpoints Only When Needed** — PrivateLink available for MSK APIs (Oct 2024) and Connect APIs (Jan 2025). Use only when needed for adequate performance or when a security need specifically requires the endpoint versus using the AWS Public network to access the API through the Browsing Service. S3 and DynamoDB Gateway endpoints are already provisioned in every VPC.
15. **SSO via PingOne** — Console access uses PingOne SAML/OIDC. Microsoft EntraId acceptable only with an approved exception.
16. **Identity Provisioning via Sailpoint/SCIM** — Automated IAM role lifecycle for MSK access aligned with entitlement reviews. Microsoft EntraId may be acceptable for provisioning only with an approved exception.
17. **Prefer Express Brokers** — Enforces TLS + IAM by default; eliminates Standard broker misconfiguration risks.
18. **Application-Layer Data Classification** — Implement masking/tokenization at producer/consumer layer. Use Glue Schema Registry for enforcement.
19. **Least-Privilege Security Groups** — Allow only port 9098 (IAM TLS) from authorized client SGs. No CIDR rules.
20. **Security Hub Monitoring** — Enable MSK.1–MSK.6 controls.

## 5. Control Framework Risks Assessment

The following table maps Amazon MSK against the organizational Risk Catalog, identifying applicable risks, their ITCF policy relation, and the current control posture for this service.

| ID | Risk Name | ITCF Policy | Review Type | Applicability to MSK | Risk Mitigation Status |
|----|-----------|-------------|-------------|---------------------|----------------------|
| 74 | Access Entitlement Review | P02.02.09 | Core Controls | IAM roles/policies for `kafka-cluster` actions require quarterly review to prevent privilege creep | Requires Implementation — Sailpoint integration for automated review |
| 76 | Active Directory Integration | P02.01.01 | Core Controls | IAM authentication is federated via AD through PingOne SSO; no direct AD-to-Kafka integration | Mitigated — IAM federation handles identity |
| 79 | Administrative Access | P02.03.01 | Core Controls | Cluster management (`kafka:*`) restricted to admin IAM roles mapped to AD groups via least-privilege | Controlled — SCP + IAM boundary policies |
| 82 | Application Scanning & Remediation | P03.02.01 | Core Controls | MSK Connect custom plugins (JAR files) must be vulnerability-scanned before upload to S3 | Requires Implementation — Include in CI/CD pipeline |
| 84 | Architecture Review Board | D02.05.01 | Core Controls | MSK cluster deployments require ARB approval per standard change process | Process Control |
| 86 | Authenticate and Login | P02.04.03 | Core Controls (Internal) | IAM authentication for Kafka API; console access via PingOne SSO | Controlled |
| 88 | Authentication Standards | P02.04.01 | Core Controls (Internal) | IAM + PingOne SSO for internal users; no direct password auth to brokers when IAM is enforced | Controlled |
| 90 | Backup and Recovery Framework | T04.02.01 | Core Controls | Multi-AZ replication provides broker-level redundancy; tiered storage to S3 for durability; MSK Replicator for cross-region DR | Partially Controlled — DR plan required per workload |
| 94 | Business Impact Analysis | T04.01.03 | Core Controls | Required for MSK clusters supporting critical data streams; RTO/RPO depends on broker type | Process Control |
| 98 | Change Management Procedures | D03.01.01 | Core Controls | Cluster configuration via Terraform/IaC with change management approval | Controlled |
| 100 | Configuration Hardening | P01.09.01 | Core Controls | SCPs enforce TLS, authentication, CMK; Security Hub MSK.1–MSK.6 monitors compliance; Express Brokers pre-hardened | Controlled |
| 101 | Data Deletion | P04.01.06 | Core Controls | Topic retention policies control lifecycle; permanent deletion via topic deletion API | Requires Configuration — Define retention per data class |
| 102 | Data Loss Prevention | P04.03.02 | Core Controls | No native DLP in MSK; must implement at application layer or via Connect transformations | Requires Implementation |
| 103 | Data Privacy | P04.01.01 | Core Controls | Data classification depends on topic content; no native classification engine | Requires Implementation — Tag topics with data class |
| 106 | Data Privacy Inventory | P04.01.04 | Core Controls | Topics containing PII/PHI/NPI must be inventoried in the data privacy register | Requires Implementation |
| 108 | Data Retention | P04.01.05 | Core Controls | Configurable via `retention.ms` and `retention.bytes` per topic; tiered storage extends retention | Configurable — Must align with policy |
| 111 | Business Continuity | T04.01.04 | Core Controls | Multi-AZ by default; Express Brokers recover 90% faster; MSK Replicator for DR | Partially Controlled |
| 112 | Document Technical Specification | D02.05.02 | Core Controls | MSK cluster architecture must be documented in Solution Design | Process Control |
| 113 | Encryption Solution (Secrets) | P04.02.04 | Core Controls | SCRAM credentials in Secrets Manager with rotation; KMS CMK for data at rest | Controlled |
| 115 | Encryption Solution | P04.02.04 | Core Controls | Always-on AES-256 at rest; TLS 1.2 in transit; cannot create unencrypted storage | Controlled |
| 117 | Establish Baseline Configuration | T02.02.01 | Core Controls | Express Brokers provide a pre-hardened baseline; Standard brokers require IaC-managed configuration | Controlled — IaC enforces baseline |
| 118 | Firewall | P05.01.01 | Core Controls | Security Groups at broker level; Palo Alto firewalls in Inspection VPC for cross-VPC traffic | Controlled |
| 122 | Hardening Guide | P05.01.02 | Core Controls | Express Brokers pre-hardened; document Standard broker hardening requirements in runbook | Controlled |
| 123 | High Availability | T03.01.01 | Core Controls | Multi-AZ by default; 99.9% SLA (Provisioned); Express provides faster recovery | Controlled |
| 124 | Information Classification Schema | P04.01.02 | Core Controls | Topics must be tagged with data classification; enforce via naming convention or resource tags | Requires Implementation |
| 125 | Information Handling Requirements | P04.01.04 | Core Controls | Data stays in selected US region; cross-border only with explicit Replicator config | Controlled |
| 126 | IP Allowlisting | P05.01.01 | Core Controls | Security Groups restrict access to specific client SGs; no CIDR-based allowlisting needed in VPC model | Controlled |
| 127 | Key Management | P04.02.03 | Core Controls | CMK with annual rotation; key policy restricts to authorized principals | Controlled |
| 129 | Logging & Monitoring | P01.10.01 | Core Controls | CloudTrail (management plane); broker logs to Splunk via Firehose; CloudWatch metrics | Controlled |
| 131 | Logging and Monitoring Framework | O02.01.01 | Core Controls | CloudTrail + broker logs + VPC Flow Logs + Security Hub findings | Controlled |
| 132 | Maintain Inventory Accuracy | M03.02.02 | Core Controls | MSK clusters discoverable via AWS Config; tagged with application metadata | Controlled |
| 134 | Perform Vendor Due Diligence | M02.03.02 | Core Controls | AWS is the vendor; SOC2/ISO certifications provide assurance | Controlled — AWS certifications |
| 136 | Network Edge Protection | P05.01.10 | Core Controls | All cross-VPC traffic inspected by Palo Alto firewalls; no direct internet exposure | Controlled |
| 137 | Network Security Controls | P05.01.04 | Core Controls | TGW routing + Inspection VPC + Security Groups + NACLs | Controlled |
| 141 | Penetration Testing | P01.09.05 | Core Controls | Kafka auth/authz should be included in annual pen testing scope | Process Control |
| 143 | Performance Monitoring | O02.01.01 | Core Controls | CloudWatch enhanced monitoring; Prometheus-compatible metrics available | Controlled |
| 145 | Privileged Access | P02.03.02 | Core Controls | Cluster admin actions restricted to approved admin roles; separated from data plane access | Controlled |
| 147 | Regulatory Requirements | P04.02.01 | Core Controls | TLS 1.2 enforced; AES-256 at rest; meets NYDFS/GDPR encryption requirements | Controlled |
| 150 | Secure Open Source Updates | P01.09.01 | Core Controls | AWS manages broker patching; customer responsible for Connect plugin JAR updates | Shared Responsibility |
| 151 | Segregation of Duties | P02.02.06 | Core Controls | Separate IAM roles for cluster admin vs. topic producer/consumer | Controlled |
| 154 | Service and Support Account Management | P02.01.04 | Core Controls | Service accounts for MSK Connect managed via IAM roles (not shared credentials) | Controlled |
| 155 | Service Offering & SLAs | P01.06.01 | Core Controls | 99.9% SLA for Provisioned clusters; document in service catalog | Process Control |
| 157 | Shared Access Accounts | P02.01.03 | Core Controls | Not applicable when IAM auth is used — each application gets its own IAM role | Controlled |
| 160 | User Access | P02.01.01 | Core Controls | IAM-based access with topic-level granularity | Controlled |
| 161 | User Access Management Procedures | P02.02.01 | Core Controls | Provisioning via Sailpoint/SCIM; quarterly entitlement review | Requires Implementation — Sailpoint integration |
| 163 | Vulnerability Management | P01.09.03 | Core Controls | AWS patches brokers; customer patches Connect plugin JARs; Security Hub monitors | Shared Responsibility |
| 165 | Vulnerability Management (CSP) | P01.09.01 | Core Controls | AWS applies security patches to managed infrastructure | Controlled — AWS responsibility |
| 167 | Separation of Duties (PreProd) | D04.03.01 | Core Controls | Separate MSK clusters per environment; separate AWS accounts for dev/staging/prod | Controlled |
| 168 | Segregation of Environments | D04.03.03 | Core Controls | Production and non-production clusters in separate accounts and VPCs | Controlled |
| 169 | Job Scheduling | O01.02.01 | Core Controls | Not directly applicable — MSK is event-driven streaming, not batch scheduling | Not Applicable |
| 170 | Jobs Monitoring | O01.02.02 | Core Controls | MSK Connect connector health monitoring via CloudWatch; consumer lag monitoring | Controlled |
| 172 | Monitor Noncompliance | G03.03.03 | Core Controls | Security Hub dashboard tracks MSK control compliance; non-compliance alerts to Splunk | Controlled |
| 173 | Generative AI | Core Controls | Not applicable — MSK is not an AI/ML service; no AI data sharing | Not Applicable |

## 6. Sources

- [Amazon MSK Encryption](https://docs.aws.amazon.com/msk/latest/developerguide/msk-encryption.html)
- [Amazon MSK Data Protection](https://docs.aws.amazon.com/msk/latest/developerguide/data-protection.html)
- [Amazon MSK IAM Access Control](https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html)
- [Amazon MSK Infrastructure Security](https://docs.aws.amazon.com/msk/latest/developerguide/infrastructure-security.html)
- [Amazon MSK Public Access](https://docs.aws.amazon.com/msk/latest/developerguide/public-access.html)
- [Amazon MSK Multi-VPC Private Connectivity](https://docs.aws.amazon.com/msk/latest/developerguide/aws-access-mult-vpc.html)
- [Amazon MSK APIs PrivateLink (October 2024)](https://aws.amazon.com/about-aws/whats-new/2024/10/amazon-msk-apis-aws-privatelink/)
- [Amazon MSK Connect APIs PrivateLink (January 2025)](https://aws.amazon.com/about-aws/whats-new/2025/01/amazon-msk-connect-apis-aws-privatelink/)
- [Amazon MSK Security Hub Controls](https://docs.aws.amazon.com/securityhub/latest/userguide/msk-controls.html)
- [Amazon MSK Express Brokers](https://docs.aws.amazon.com/msk/latest/developerguide/msk-broker-types-express.html)
- [Amazon MSK Replicator](https://docs.aws.amazon.com/msk/latest/developerguide/msk-replicator.html)
- [Amazon MSK Update Security](https://docs.aws.amazon.com/msk/latest/developerguide/msk-update-security.html)
- [Amazon MSK FAQs](https://aws.amazon.com/msk/faqs/)
- [Amazon MSK Pricing](https://aws.amazon.com/msk/pricing/)
- [AWS AI Services Opt-Out Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out.html)
- [AWS FedRAMP Services in Scope](https://aws.amazon.com/compliance/services-in-scope/FedRAMP/)
- [AWS NYDFS Encryption Guidance (August 2024)](https://aws.amazon.com/blogs/security/encryption-in-transit-over-external-networks-aws-guidance-for-nydfs-and-beyond/)
- [Express Brokers GA (November 2024)](https://aws.amazon.com/blogs/aws/introducing-express-brokers-for-amazon-msk-to-deliver-high-throughput-and-faster-scaling-for-your-kafka-clusters/)
- [MSK Cross-Account Connect (June 2025)](https://aws.amazon.com/blogs/big-data/secure-access-to-a-cross-account-amazon-msk-cluster-from-amazon-msk-connect-using-iam-authentication/)
- Internal: Transamerica-Organizational-Alignment steering document (authoritative region baseline and network architecture context)

## 7. Appendix A: Security Guardrails and Controls Examples

### A.1 SCP: Block Public Access

```hcl
resource "aws_organizations_policy" "msk_block_public_access" {
  name = "msk-block-public-access"
  type = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "DenyMSKPublicAccess"
      Effect    = "Deny"
      Action    = "kafka:UpdateConnectivity"
      Resource  = "*"
      Condition = {
        "ForAnyValue:StringEquals" = {
          "kafka:publicAccess" = "SERVICE_PROVIDED_EIPS"
        }
      }
    }]
  })
}
```

**Detection**: Security Hub MSK.4; Config rule `msk-cluster-public-access-check`. Alert to Splunk via EventBridge.

---

### A.2 SCP: Enforce TLS

```hcl
resource "aws_organizations_policy" "msk_enforce_tls" {
  name = "msk-enforce-tls"
  type = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "DenyPlaintext"
      Effect = "Deny"
      Action = ["kafka:CreateCluster", "kafka:CreateClusterV2", "kafka:UpdateSecurity"]
      Resource = "*"
      Condition = {
        "ForAnyValue:StringEquals" = {
          "kafka:ClientBroker" = ["PLAINTEXT", "TLS_PLAINTEXT"]
        }
      }
    }]
  })
}
```

**Detection**: Security Hub MSK.1; Config rule `msk-in-cluster-node-require-tls`. Target: 100% compliance.

---

### A.3 SCP: Enforce Authentication

```hcl
resource "aws_organizations_policy" "msk_enforce_auth" {
  name = "msk-enforce-auth"
  type = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "DenyUnauthenticated"
      Effect = "Deny"
      Action = ["kafka:CreateCluster", "kafka:CreateClusterV2"]
      Resource = "*"
      Condition = {
        "Bool" = { "kafka:unauthenticatedAccess" = "true" }
      }
    }]
  })
}
```

**Detection**: Security Hub MSK.6. Target: zero unauthenticated clusters.

---

### A.4 Secure Cluster (Express Brokers)

```hcl
resource "aws_msk_cluster" "main" {
  cluster_name           = "events-nonprod"
  kafka_version          = "3.7.x.kraft"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.m7g.large"
    client_subnets  = var.private_subnet_ids
    security_groups = [aws_security_group.msk.id]
    storage_info { ebs_storage_info { volume_size = 100 } }
  }

  encryption_info {
    encryption_at_rest_kms_key_arn = aws_kms_key.msk.arn
    encryption_in_transit { client_broker = "TLS"; in_cluster = true }
  }

  client_authentication {
    sasl { iam = true }
    unauthenticated = false
  }

  logging_info {
    broker_logs {
      cloudwatch_logs { enabled = true; log_group = aws_cloudwatch_log_group.msk.name }
      firehose { enabled = true; delivery_stream = aws_kinesis_firehose_delivery_stream.splunk.name }
    }
  }

  tags = { Environment = "nonprod"; DataClassification = "internal" }
}

resource "aws_kms_key" "msk" {
  description         = "CMK for MSK encryption"
  enable_key_rotation = true
}
```

**Detection**: Config rule checking KMS key ARN is customer-managed. Quarterly key audit per P04.02.03.

---

### A.5 Least-Privilege Security Group

```hcl
resource "aws_security_group" "msk" {
  name_prefix = "msk-broker-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 9098
    to_port         = 9098
    protocol        = "tcp"
    security_groups = [var.client_sg_id]
    description     = "IAM auth TLS from authorized clients"
  }

  ingress {
    from_port = 9094; to_port = 9094; protocol = "tcp"; self = true
    description = "Inter-broker TLS"
  }

  egress {
    from_port = 443; to_port = 443; protocol = "tcp"
    cidr_blocks = var.aws_svc_cidrs
    description = "KMS, CloudWatch, S3"
  }

  egress {
    from_port = 9094; to_port = 9098; protocol = "tcp"; self = true
    description = "Inter-broker"
  }
}
```

**Detection**: VPC Flow Logs + Splunk alerting on rejected connections to broker ports.

---

### A.6 Least-Privilege IAM (Consumer)

```hcl
resource "aws_iam_policy" "consumer" {
  name = "msk-consumer-orders"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      { Effect = "Allow"; Action = ["kafka-cluster:Connect"]
        Resource = "arn:aws:kafka:${var.region}:${var.acct}:cluster/${var.cluster}/*" },
      { Effect = "Allow"; Action = ["kafka-cluster:ReadData","kafka-cluster:DescribeTopic"]
        Resource = "arn:aws:kafka:${var.region}:${var.acct}:topic/${var.cluster}/*/orders.*" },
      { Effect = "Allow"; Action = ["kafka-cluster:AlterGroup","kafka-cluster:DescribeGroup"]
        Resource = "arn:aws:kafka:${var.region}:${var.acct}:group/${var.cluster}/*/${var.group}" }
    ]
  })
}
```

**Detection**: IAM Access Analyzer flags wildcard `kafka-cluster:*` grants. Quarterly Sailpoint review (P02.02.09).

---

### A.7 Splunk SIEM Integration

```hcl
resource "aws_kinesis_firehose_delivery_stream" "splunk" {
  name        = "msk-to-splunk"
  destination = "splunk"
  splunk_configuration {
    hec_endpoint               = var.splunk_hec_url
    hec_token                  = var.splunk_hec_token
    hec_acknowledgment_timeout = 300
    retry_duration             = 3600
    s3_backup_mode             = "FailedEventsOnly"
  }
  s3_configuration {
    role_arn   = aws_iam_role.firehose.arn
    bucket_arn = var.backup_bucket_arn
    prefix     = "msk/failed/"
  }
}
```

**Detection**: Monitor `DeliveryToSplunk.Success` > 99.9%. Alert on delivery gaps > 5 minutes.

---

### A.8 Restrict Multi-VPC Connectivity

```hcl
resource "aws_organizations_policy" "msk_restrict_privatelink" {
  name = "msk-restrict-multi-vpc"
  type = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "DenyMultiVPC"
      Effect = "Deny"
      Action = ["kafka:UpdateConnectivity", "kafka:PutClusterPolicy"]
      Resource = "*"
      Condition = {
        "StringNotEquals" = { "aws:PrincipalOrgPaths" = var.admin_ou_paths }
      }
    }]
  })
}
```

**Detection**: CloudTrail alerts on `PutClusterPolicy`. Requires documented architecture exception.

---

### A.9 Enforce CMK

```hcl
resource "aws_organizations_policy" "msk_enforce_cmk" {
  name = "msk-enforce-cmk"
  type = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "DenyDefaultKey"
      Effect = "Deny"
      Action = ["kafka:CreateCluster", "kafka:CreateClusterV2"]
      Resource = "*"
      Condition = {
        "StringEquals" = { "kafka:EncryptionAtRestKmsKeyId" = "" }
      }
    }]
  })
}
```

**Detection**: Custom Config rule verifying CMK ARN. Quarterly key management review (P04.02.03).

---

### A.10 Authentication Failure Alerting

```hcl
resource "aws_cloudwatch_log_metric_filter" "auth_fail" {
  name           = "msk-auth-failures"
  pattern        = "\"AuthenticationException\""
  log_group_name = aws_cloudwatch_log_group.msk.name
  metric_transformation {
    name = "MSKAuthFailures"; namespace = "Custom/MSK"; value = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "auth_alarm" {
  alarm_name          = "msk-auth-spike"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "MSKAuthFailures"
  namespace           = "Custom/MSK"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_actions       = [var.sns_security_arn]
}
```

**Detection**: Route alarm to Splunk via SNS → Lambda → HEC. Dashboard for auth success/failure ratio.

---

### A.11 SCP: Restrict MSK to Approved Transamerica Regions

```hcl
resource "aws_organizations_policy" "msk_restrict_regions" {
  name = "msk-restrict-to-approved-regions"
  type = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "DenyMSKOutsideApprovedRegions"
      Effect    = "Deny"
      Action    = "kafka:*"
      Resource  = "*"
      Condition = {
        StringNotEquals = {
          "aws:RequestedRegion" = ["us-east-1", "us-east-2", "eu-west-1"]
        }
      }
    }]
  })
}
```

**Detection**: CloudTrail monitoring for `kafka:CreateCluster` or `kafka:CreateClusterV2` events with `awsRegion` outside approved Transamerica regions. Alert to Splunk.

---

### A.12 Production Readiness Checklist

Before granting production go-live for MSK clusters, confirm all items:

| # | Control | Owner | Status |
|---|---------|-------|--------|
| 1 | Express Brokers selected (not Standard) | Application Team | ☐ |
| 2 | Customer-managed KMS key configured with rotation | Cloud Security | ☐ |
| 3 | IAM authentication enforced (no unauthenticated access) | Application Team | ☐ |
| 4 | TLS-only client-to-broker communication | Application Team | ☐ |
| 5 | Topic-level IAM policies for each producer/consumer | Application Team | ☐ |
| 6 | Data classification documented per topic | Data Governance / DPO | ☐ |
| 7 | DPIA completed for topics containing PII/NPI/PHI | DPO | ☐ |
| 8 | Security Groups restrict to port 9098 from authorized client SGs only | Cloud Networking (ET) | ☐ |
| 9 | Broker logs forwarded to Splunk via Firehose | Security Operations | ☐ |
| 10 | CloudTrail logging verified for kafka:* actions | Security Operations | ☐ |
| 11 | Security Hub MSK.1–MSK.6 controls reporting compliant | Cloud Security | ☐ |
| 12 | AWS Budgets configured with appropriate thresholds | Finance / Application | ☐ |
| 13 | DR strategy documented (RTO/RPO, MSK Replicator evaluation) | Application Team | ☐ |
| 14 | Consumer lag CloudWatch alarms configured | Application Team | ☐ |
| 15 | Authentication failure alerting configured | Security Operations | ☐ |
| 16 | Sailpoint quarterly review scheduled for kafka-cluster roles | Identity Team | ☐ |
| 17 | MSK Connect plugin JARs vulnerability-scanned | DevOps Team | ☐ |
| 18 | Architecture Review Board approval obtained | ARB | ☐ |
| 19 | Change management process documented for production changes | Application Team | ☐ |
| 20 | Application-layer DLP/masking implemented for sensitive topics | Application Team | ☐ |

---
