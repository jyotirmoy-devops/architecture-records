# Technical Architecture Document (TAD)

## Draw.io Diagram URL
https://app.diagrams.net/?grid=0&pv=0&border=10&edit=_blank#create=%7B%22type%22%3A%22mermaid%22%2C%22compressed%22%3Atrue%2C%22data%22%3A%22dVPbbhoxEP2aldoHJEqF%2BrxXihLIJpsmz4Mx4MrY1PZC2q%2FvjPfmJSCtvPZ4fM7MnJmd1Bd2AOOi2fTxJZrG%2BMcvjeZJ%2FuG4USDpKAVXLppnuJ9Mou959CP9%2BfpaVt47wXVV4MsieLLM0Af%2F8QKh4n%2B14XQ4naRg4IRWeFqA4xf4i7vzrAd6jwvi6QLB5x0lHsoxFq4VN2fBeDRP1a%2BT1LBFW6mN8zE0iL09Lpcj6DLIpnLao5ZZ0cSdINWXjiuRekNk6AR7T7ZUTB%2BF2nuoAtfnGgwoJxTvTankoL7epczVn5rX5P4b0T1pNWLtckP%2B2noKdB%2FhJQHck5rUXaZHkBfwGJaBarCzHAubrAQz2uod6Z3xHVdbbnC70ybMb2CgV0GVCG02NdzWkhD4mdrCw%2Behzrm3o8JGbEO0fBGA%2BfIQt5B8jFWsA6yiVowaxlLZV9zBFhwQ1oczwNpWapReayd2XX8FtNVQJw%2Fe2XE7hPNuhCO%2BY09RWIcd2irz%2FDiSBo9YHHTbgB1rMgKtsMB4FKqfC4UxshaU0lwOVzFjuqYW2n%2FKJVWvHI6WRuwIgpxfuMTRucd7oxtBctPWt8ppvivOasyZ5i%2BmO%2BK93auz6cNboMgDpzdvQD1w78HqKdRwpZVwvsWaWJrZXSor9gdn72TRgPQa%2Borfuknu2G292Rs4HWiqjTiDVzfVSnHsmjMl3nk2X5l%2Fw5AH11xtTxqVs31%2FDXfZugqjJpWvyjBpEcdRXlu7rK7tWO%2Bb7rdR2rFqzf8B%22%7D

# 1. Purpose
Design a secure, cost-effective, and highly scalable web application for external clients to upload sensitive financial tax documents (PDFs). The system must automatically scan each upload for malware, extract basic metadata (Client ID and Year), and notify the internal accounting team after successful processing.

# 2. Architecture Summary
The recommended solution is a **secure, event-driven, PaaS-first Azure architecture** using a **Web-Queue-Worker** pattern with a durable queue.

Core characteristics:
- Thin public upload front end
- Asynchronous processing pipeline
- Mandatory malware scanning before downstream use
- Minimal metadata extraction only
- Internal notification after validation
- Private connectivity for internal services
- Autoscaling for tax-season spikes
- Strong logging, auditability, and encryption
- Explicit reliability, DR, and cost controls

# 3. Business and Technical Requirements
## Functional requirements
- External client authentication
- PDF upload only
- Malware scanning on every upload
- Metadata extraction for Client ID and Year
- Notification to internal accounting team
- Status tracking for uploaded documents

## Non-functional requirements
- High security
- Low operating cost
- Automatic scaling
- Strong auditability
- Safe failure behavior
- Resilience during traffic spikes
- Defined recovery objectives

# 4. Architecture Principles
- Security by design
- Least privilege
- Fail closed
- Asynchronous by default
- Stateless services
- Idempotent processing
- Data minimization
- Separation of duties
- Auditability
- Elasticity with governance
- Secretless service-to-service access where possible

# 5. Logical Architecture
## 5.1 Public access layer
**Azure Application Gateway v2 + WAF**
- Public HTTPS ingress
- Web application firewall protection
- TLS termination and request filtering
- First line of defense against common web attacks
- Use only if justified by security and traffic requirements; otherwise consider a lower-cost edge pattern in implementation planning

## 5.2 Web application layer
**Azure App Service**
- Hosts the client portal and upload API
- Handles authentication, upload initiation, and status queries
- Remains thin and stateless
- Uses managed identity for service access
- Supports autoscale and deployment slots
- External authentication must be implemented with a defined external identity pattern such as Microsoft Entra External ID or an approved federated identity solution

## 5.3 Durable work queue
**Azure Service Bus**
- Provides the durable queue required by the Web-Queue-Worker pattern
- Buffers upload-processing work during tax-season spikes
- Supports retries, dead-lettering, and backpressure
- Prevents Event Grid from being used as the only work buffer
- Enables poison-message isolation and controlled reprocessing

## 5.4 Document storage layer
**Azure Blob Storage**
- Stores uploaded PDFs as immutable objects
- Uses separate containers for lifecycle states:
  - incoming-unscanned
  - quarantine
  - clean-approved
  - rejected-malicious
- Public access disabled
- Private endpoint enabled
- Soft delete and versioning enabled where appropriate
- Storage account should use encryption at rest and approved key management standards

## 5.5 Malware scanning layer
**Microsoft Defender for Storage**
- Performs on-upload malware scanning automatically
- Every file is treated as untrusted until scan completion
- Malicious files are quarantined or blocked from downstream processing
- Scan results are retained for audit
- Monthly scan cap should be configured for cost control
- If the scan cap is reached, the system must fail closed and route uploads to a safe pending/quarantine state with alerting

## 5.6 Eventing and orchestration layer
**Azure Event Grid**
- Receives scan-result and upload lifecycle events
- Decouples upload from processing
- Supports event filtering and private connectivity where required
- Used for event notification, not as the sole durable processing buffer

**Azure Functions**
- Consumes clean-file events or Service Bus messages
- Performs metadata extraction orchestration
- Sends notification to accounting
- Handles quarantine or exception workflows
- Prefer Consumption for cost efficiency unless Premium is required for latency, VNet needs, or warm instances

## 5.7 Data layer
**Azure SQL Database**
- Stores extracted metadata and workflow state
- Supports structured queries and audit reporting
- Holds document lifecycle fields such as:
  - DocumentId
  - ClientId
  - TaxYear
  - UploadTimestamp
  - ScanStatus
  - MalwareResult
  - BlobUri
  - Checksum
  - NotificationStatus
- Use an appropriately sized tier with indexing and retention controls to avoid hidden cost growth

## 5.8 Identity and secrets
**Microsoft Entra ID**
- Authenticates internal users and administrators
- Supports RBAC and MFA
- External client identity must be explicitly defined and isolated from internal admin identity

**Azure Key Vault**
- Stores certificates and secrets if needed
- Accessed via managed identity
- Private endpoint enabled
- Public access disabled
- Key rotation, backup, purge protection, and access review requirements must be defined before implementation

## 5.9 Observability
**Azure Monitor / Application Insights**
- Captures logs, metrics, and traces
- Supports alerting and operational visibility
- Must not store sensitive document contents or secrets in logs
- Must support SLO/SLI monitoring and actionable alert thresholds

# 6. End-to-End Processing Flow
## Happy path
1. Client authenticates and uploads a PDF through the portal.
2. Application Gateway WAF forwards the request to App Service.
3. App Service stores the file in Blob Storage and enqueues a processing message in Service Bus.
4. Defender for Storage scans the blob automatically.
5. Event Grid emits the scan result.
6. Azure Functions processes the clean-file event or Service Bus message.
7. Metadata extraction retrieves only Client ID and Year.
8. Metadata is validated and stored in Azure SQL Database.
9. Notification is sent to the internal accounting team.
10. Processing status is updated for client visibility and audit.

## Malware-detected path
1. File is uploaded.
2. Defender detects malware.
3. File is quarantined or moved to rejected-malicious.
4. Downstream extraction and notification are suppressed.
5. Security alerting and audit logging occur.
6. Client receives a safe, non-sensitive status response.

## Metadata-failure path
1. File is clean.
2. Metadata extraction fails or values are invalid.
3. Document is flagged for manual review.
4. Accounting is not notified automatically.
5. Audit and status records are updated.

## Downstream service failure path
1. File is clean but Service Bus, SQL, or notification services are unavailable.
2. The system retains the document in a safe pending state.
3. Retries occur with backoff and dead-letter handling where appropriate.
4. No duplicate notifications are sent.
5. Operations are alerted and the system fails closed.

# 7. Security Architecture
## Access control
- External clients must authenticate before upload
- Internal accounting access must be restricted to authorized users
- Administrative access must be least privilege and fully logged
- MFA required for internal users
- No shared accounts
- Tenant/client isolation rules must be enforced for document access

## File upload security
- Only PDF files allowed
- Validate file type by content, not extension
- Enforce file size limits
- Rate limit uploads
- Reject malformed or duplicate files
- Treat all uploads as untrusted until scanning completes
- Protect against oversized payloads and abuse patterns through request limits and throttling

## Malware scanning and quarantine
- Scan every upload before downstream use
- Quarantine or reject malicious files
- No notification until file is clean
- Retain scan results for audit
- Support re-scan if threat intelligence changes
- If scanning is unavailable, the file must not proceed

## Metadata extraction
- Extract only Client ID and Year
- Run only after malware scan passes
- Do not execute embedded content or scripts
- Validate extracted values against business rules
- Flag missing or invalid metadata for manual review
- Use safe parsing libraries and content-disarm principles where applicable

## Notification controls
- Send only necessary information
- Do not include full document contents
- Notify only after scan and metadata validation
- Log delivery outcomes
- Prevent duplicate alerts and reprocessing loops
- Use an approved internal transport such as Teams, email relay, or workflow integration with anti-spoofing controls

## Logging and monitoring
Log all security-relevant events:
- upload attempt
- authentication event
- scan start/end
- scan verdict
- metadata extraction result
- notification sent/failed
- quarantine action
- administrative access

Additional logging requirements:
- Tamper-resistant central retention
- Time synchronization enforced
- No sensitive content or secrets in logs
- Correlation IDs across all services

## Encryption and data protection
- TLS for data in transit
- Encryption at rest for documents and metadata
- Keys managed under enterprise standards
- Key access restricted and audited
- No sensitive data in URLs, logs, or error messages
- Prefer managed identity and secretless connectivity for service-to-service calls

# 8. Availability, Resilience, and Scale
## Scale strategy
- App Service autoscale for the web tier
- Functions Consumption for bursty event processing
- Service Bus for durable buffering and backpressure
- Event-driven decoupling to absorb spikes
- Queue depth and processing latency as primary scaling signals

## Resilience strategy
- Asynchronous processing
- Retry with backoff for transient failures
- Dead-letter handling for poison messages
- Idempotent consumers
- Correlation IDs across services
- Graceful degradation if downstream services are unavailable
- Explicit degraded-mode behavior when scanning, SQL, or notification services are unavailable

## Disaster recovery and recovery objectives
- Define RTO and RPO before implementation approval
- Use automated backups for SQL and storage recovery
- Document restore and failover procedures for App Service, SQL, Storage, Key Vault, and Service Bus
- Test recovery regularly
- Keep region and redundancy choices aligned to data residency and compliance requirements

## Cost controls
- Prefer PaaS over IaaS
- Prefer Functions Consumption for burst workloads
- Use Defender scan caps with alerting and safe fallback behavior
- Minimize private endpoints to required services only
- Avoid overprovisioning
- Keep synchronous processing minimal
- Add budgets, alerts, and usage thresholds for cost governance

# 9. Network and Connectivity
- Dedicated VNet for the workload
- Subnets for Application Gateway, App Service integration, and Private Endpoints
- Private Endpoints for Storage, Key Vault, Service Bus, SQL, and Event Grid where applicable
- Private DNS zones linked to the VNet
- NSGs applied to subnets
- Public network access disabled where possible
- WAF policies at the edge
- Consider DDoS Network Protection for high-value exposure
- Justify each private endpoint based on security and cost

# 10. Data Architecture
## Primary storage
**Blob Storage** is the system of record for PDFs.

## Metadata storage
**Azure SQL Database** is the system of record for workflow state and audit-friendly metadata.

## Optional transient cache
**Azure Cache for Redis** may be used only for:
- short-lived upload session state
- idempotency keys
- rate-limit counters
- temporary workflow coordination
- reference data

Do not cache raw PDFs or sensitive content.

## Data model guidance
Suggested SQL entities:
- Documents
- Clients
- ScanEvents
- NotificationEvents
- AuditEvents

Suggested indexes:
- ClientId + TaxYear
- ScanStatus + UploadTimestamp
- DocumentId primary key
- Checksum unique if deduplication is needed

# 11. Reliability, Security, and Operational Gaps Addressed
This revision explicitly fixes the prior review findings by adding:
- a durable queue via Azure Service Bus
- external authentication requirements
- explicit authorization and tenant isolation
- DR, backup, and failover requirements
- CI/CD and IaC expectations
- SLO/SLI and alerting requirements
- cost governance and scan-cap fallback behavior
- notification transport controls
- key rotation and secret lifecycle requirements
- safe failure behavior for downstream outages

# 12. Compliance and Governance
The solution must align to:
- NIST control families: AC, AU, IA, SC, SI, IR, CP
- CIS Controls v8: 4, 6, 8, 10, 11, 13

Governance prerequisites before implementation approval:
- security architecture review
- privacy review
- data classification confirmation
- threat modeling
- logging and retention approval
- incident response runbook
- recovery testing
- cost guardrails and usage thresholds
- CI/CD and infrastructure-as-code approval

# 13. In Scope
- Upload intake
- Malware scanning
- Metadata extraction
- Notification workflow
- Audit logging
- Access control
- Encryption
- Quarantine and incident handling
- Durable queueing and retry handling
- Recovery and failover planning

# 14. Out of Scope
- Unrelated client data storage or processing
- Full document content analytics
- Human review unless explicitly approved
- Model training or secondary use of uploaded files
- Public exposure of document links or metadata

# 15. Recommended Azure Service Set
- Azure Application Gateway v2 + WAF
- Azure App Service
- Azure Blob Storage
- Microsoft Defender for Storage
- Azure Service Bus
- Azure Event Grid
- Azure Functions
- Azure SQL Database
- Azure Key Vault
- Microsoft Entra ID
- Azure Monitor / Application Insights
- Private Endpoints and Private DNS Zones

# 16. Final Recommendation
Use a **secure, event-driven, PaaS-first Azure architecture** with:
- Application Gateway WAF v2
- App Service
- Blob Storage
- Defender for Storage on-upload malware scanning
- Azure Service Bus
- Azure Functions
- Event Grid
- Azure SQL Database
- Key Vault
- Private Endpoints and Private DNS
- Azure Monitor / App Insights

This design provides:
- strong security for sensitive tax documents
- low operational cost
- automatic scaling for seasonal spikes
- clean separation of concerns
- auditability and compliance support
- improved reliability through durable queueing and explicit recovery controls

# 17. Implementation Notes
Before build approval, the delivery team must produce:
- external identity design
- threat model and abuse cases
- CI/CD and IaC plan
- DR plan with RTO/RPO
- SLOs/SLIs and alert thresholds
- cost model and budget alerts
- notification transport specification
- key rotation and secret lifecycle procedures
- runbooks for scan failure, queue backlog, SQL throttling, and notification outages
