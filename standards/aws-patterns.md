# AWS Reference Patterns

Reusable, client-anonymized building blocks I reach for on AWS. Grown over time by
/retro and manual additions. Each pattern: when to use, the shape of it, gotchas learned.

## How to add a pattern
Use this structure so /diagram can consume them consistently:

### <Pattern name>
- **Use when**: <situation / requirements profile>
- **Shape**: <components and how they connect — prose or a small Mermaid sketch>
- **Gotchas**: <what bit us before: quotas, latency, cost cliffs, compliance>
- **Won/lost evidence**: <anonymized: "well received in 2 pitches", "cost pushback once">

## Patterns

(empty — will grow with use. Seed it by asking Claude to write up the 3-5 AWS
architectures you draw most often.)

## Seeded reference patterns (official AWS docs)

Baselines distilled from AWS Well-Architected / Architecture Center / Prescriptive
Guidance docs (fetched 2026-07-10). No engagement evidence yet — these carry a **Source**
field instead; add won/lost evidence as we use them. Treat them as the "defensible
default" to deviate from consciously, not gospel.

### AWS Well-Architected design-review baseline (6 pillars — not a topology, a checklist)
- **Use when**: Run this checklist before any design review or proposal sign-off. Every
  architecture decision should be defensible against all six pillars, with explicit notes
  where one pillar was traded against another.
  - **Operational excellence** — can you run, monitor, and continuously improve the workload (runbooks, observability, small reversible changes)?
  - **Security** — are data, systems, and assets protected (identity, detective controls, infra + data protection, incident response)?
  - **Reliability** — does the workload perform its intended function correctly and consistently, and recover from failure (Multi-AZ, quotas, DR, change management)?
  - **Performance efficiency** — are compute/storage/database/network choices right-sized and re-evaluated as services evolve (serverless first, experiment often)?
  - **Cost optimization** — is business value delivered at the lowest price point (right-sizing, pricing models, decommission idle resources, cost attribution)?
  - **Sustainability** — are energy consumption and resource utilization minimized (region choice, utilization targets, managed/serverless services, data lifecycle)?
- **Gotchas**:
  - **Pillars trade against each other by design** — the framework explicitly expects trade-offs (e.g., reliability vs cost in dev/test); document each trade-off, don't hide it.
  - **Security and operational excellence are generally NOT traded** against the other pillars — reviewers will gate on any design that sacrifices either for cost or speed.
  - **Sustainability is the sixth pillar** (added Dec 2021) and is still routinely missing from checklists built on the older five-pillar framework — stale templates fail current reviews.
  - **Reviews done only at the end are remediation exercises** — review early and throughout; a pre-build review is cheap, a post-launch one is a re-architecture.
  - **The generic framework is not enough for specialized workloads** — AWS maintains domain lenses (Serverless, SaaS, Generative AI, Analytics); a reviewer using the matching lens will find gaps the base framework misses.
  - **The free AWS Well-Architected Tool** produces a measurable risk register (HRIs/MRIs) — use it so pillar findings are tracked, not just discussed.
- **Source**: docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html

### Baseline highly-available secure web application (containerized three-tier)
- **Use when**: Production web application for a small-to-mid workload needing HA across
  AZs, elastic scale-out, and managed security controls without a platform team. This is
  AWS's current canonical baseline (the old "Web Application Hosting in the AWS Cloud"
  whitepaper is archived; the Solutions Library "Guidance for Building a Containerized and
  Scalable Web Application on AWS" supersedes it). Swap DynamoDB for Multi-AZ RDS/Aurora
  when the data model is relational — the rest of the shape holds.
- **Shape**: Static content from the edge; dynamic traffic through API Gateway to an
  internet-facing ALB fronting ECS-on-Fargate tasks in private subnets across AZs.
  ```mermaid
  flowchart LR
    U[Client] --> R53[Route 53]
    R53 --> CF[CloudFront]
    CF --> S3[(S3 static)]
    CF --> AG[API Gateway]
    COG[Cognito] -.auth.- AG
    AG --> ALB[ALB multi-AZ]
    ALB --> ECS[ECS on Fargate]
    ECS --> DB[(DynamoDB / RDS Multi-AZ)]
    ECR[ECR] -.images.-> ECS
  ```
  Identity: Cognito for user auth; task execution and task roles are separate IAM roles
  (execution role pulls from ECR and reads Secrets Manager; task role holds only app
  permissions). Networking: ALB in public subnets, Fargate tasks and database in private
  subnets; AWS WAF on CloudFront and/or the ALB, Shield via CloudFront for DDoS. Database
  credentials in Secrets Manager injected as ECS secrets, never in task definitions. Ops:
  CloudWatch for metrics/logs, ECS service auto scaling on ALB request-count-per-target,
  Route 53 health checks for failover.
- **Gotchas**:
  - **ALB idle timeout defaults to 60 seconds** — long-polling/streaming responses get severed silently; keep app keep-alive shorter than the ALB timeout or connections reset under load.
  - **Every Fargate task consumes an ENI and a private IP** (awsvpc mode) — undersized private subnet CIDRs cap scale-out long before service quotas do; plan /24 or larger per AZ.
  - **Image pulls from ECR traverse the NAT gateway by default** from private subnets — a real cost cliff at scale; add ECR + S3 VPC endpoints to keep pulls private and cheap.
  - **Fargate scale-out is not instant** — task provisioning + image pull + ALB health-check grace means minutes, not seconds; set min capacity for known traffic floors instead of scaling from zero.
  - **DynamoDB hot partitions throttle regardless of provisioned capacity** — a low-cardinality partition key defeats "it just scales"; if you substitute RDS, Multi-AZ is failover HA, not read scaling (add read replicas separately).
  - **CloudFront caches only what you tell it to** — default cache policies on dynamic paths cause stale responses or a 0% hit ratio with full origin load; split behaviors for `/static/*` vs API paths explicitly.
- **Source**: docs.aws.amazon.com/solutions/building-a-containerized-and-scalable-web-application-on-aws/

### Baseline generative-AI chat / RAG on Amazon Bedrock
- **Use when**: Enterprise chat assistant or Q&A over proprietary documents where you want
  managed ingestion→embedding→retrieval instead of a hand-rolled pipeline, with citations
  for auditability and content safety controls. Note the docs now distinguish a fully
  **Managed Knowledge Base** (AWS-managed embedding, storage auto-scaling, agentic
  multi-hop retrieval, ACL-based document-level permission filtering) from a
  **customer-managed** knowledge base where you own the vector store — AWS now recommends
  Managed for most cases.
- **Shape**: Documents sync from data sources into a vector store via Knowledge Bases; the
  app calls `RetrieveAndGenerate` (single-call RAG with session memory) or `Retrieve`
  (bring-your-own generation), wrapped in Bedrock Guardrails.
  ```mermaid
  flowchart LR
    U[User] --> APP[Chat app / Agent]
    APP --> GR[Bedrock Guardrails]
    GR --> KB[Bedrock Knowledge Base]
    KB --> VS[(Vector store: OpenSearch Serverless / Aurora pgvector / Neptune Analytics)]
    KB --> FM[Bedrock FM via RetrieveAndGenerate]
    S3[(S3 / SharePoint / Confluence docs)] -->|managed ingest + embed| KB
    SM[Secrets Manager] -.connector creds.- KB
  ```
  Identity: the Knowledge Base uses a dedicated IAM service role to read sources and write
  the vector store; connector credentials (SharePoint/Confluence/Salesforce OAuth) live in
  Secrets Manager; Managed KB supports document-level permission filtering at retrieval
  time. Networking: keep all calls private with PrivateLink interface endpoints for
  `bedrock-runtime` and `bedrock-agent-runtime`, and OpenSearch Serverless VPC endpoints.
  Ops: enable Bedrock model invocation logging to CloudWatch/S3, attach Guardrails
  (content filters, denied topics, PII redaction, contextual-grounding checks) to both the
  KB and the generation call, monitor sync-job status per data source in CloudWatch.
- **Gotchas**:
  - **The auto-created OpenSearch Serverless vector store bills in OCUs with a capacity floor even when idle** — the "quick create" path is a standing monthly cost, not pay-per-query; for small corpora consider Aurora PostgreSQL (pgvector) you already run.
  - **Ingestion is batch sync, not streaming** — documents are only queryable after a sync job completes; "why doesn't it know the doc I just uploaded" is a sync-schedule issue, budget for it in UX.
  - **Chunking strategy is fixed per data source at creation** — changing chunk size/strategy later means re-ingesting the corpus; test chunking on a sample before the full sync.
  - **`RetrieveAndGenerate` couples you to Bedrock's prompt orchestration** — if you need custom re-ranking, query rewriting, or a non-Bedrock generator, design around `Retrieve` from day one; migrating later touches the whole app.
  - **Guardrails are a separate resource with separate per-unit pricing and their own quotas** — reviewers gate on guardrails being attached in *both* directions (input and output) and on contextual grounding being enabled for RAG, not just profanity filters.
  - **Model and vector-store availability is per-Region** — check the supported models/Regions page before committing a data-residency story; cross-Region inference profiles change the compliance answer.
- **Source**: docs.aws.amazon.com/prescriptive-guidance/latest/retrieval-augmented-generation-options/rag-fully-managed-bedrock.html

### Data lakehouse / modern data architecture (S3 + Glue + Lake Formation + Redshift/Athena)
- **Use when**: Consolidating enterprise data (operational DBs, SaaS, streams, files,
  third-party) into one governed platform serving BI, ad-hoc SQL, big-data processing, and
  ML from a single copy of data. This is AWS's "Modern Data Analytics Reference
  Architecture" (May 2022, still the canonical diagram); note AWS's newer direction layers
  **Amazon SageMaker Lakehouse** (unified S3 + Redshift access, S3 Tables with native
  Apache Iceberg) on top of the same foundation — new builds should evaluate Iceberg/S3
  Tables from the start.
- **Shape**: Purpose-built ingestion services land data in an S3 data lake governed by
  Lake Formation with the Glue Data Catalog as the single metadata store; multiple engines
  consume in place.
  ```mermaid
  flowchart LR
    SRC[Sources: DBs, SaaS, streams] --> ING[DMS / Kinesis / MSK / AppFlow / Transfer Family]
    ING --> S3[(S3 data lake)]
    LF[Lake Formation governance] -.permissions.- S3
    S3 --> GLUE[Glue ETL + Data Catalog]
    GLUE --> ATH[Athena]
    GLUE --> RS[Redshift + Spectrum]
    GLUE --> EMR[EMR / SageMaker AI]
    ATH --> BI[QuickSight / Quick Suite]
    RS --> BI
  ```
  Identity and governance: Lake Formation centralizes table/column/row-level grants,
  cross-account sharing, and audit trails on top of the Glue Data Catalog — consumers
  (Athena, Redshift Spectrum, EMR, Glue) all enforce the same grants. Networking: keep
  engines private with VPC endpoints for S3/Glue/Athena; Aurora zero-ETL integration feeds
  Redshift without pipeline code. Ops: Glue bills per DPU-hour, Athena per TB scanned, so
  file format (Parquet/Iceberg), compression, and partitioning are the primary cost and
  performance levers; Managed Service for Apache Flink handles the streaming-transform
  path, QuickSight on top for BI.
- **Gotchas**:
  - **Lake Formation and IAM are two overlapping permission systems** — until you flip tables to LF-managed and revoke the `IAMAllowedPrincipals` super-permission, LF grants are silently bypassed; hybrid mode is the #1 governance audit finding.
  - **Athena charges ~$5 per TB scanned with a 10 MB per-query minimum** — unpartitioned raw JSON/CSV makes every dashboard refresh a full-table scan; converting to partitioned Parquet routinely cuts scan cost by >90%.
  - **The small-files problem kills both Glue and Athena** — millions of KB-sized objects from streaming ingest inflate job times and S3 request costs; compact to 128 MB+ files (or adopt Iceberg with automatic compaction via S3 Tables).
  - **Redshift Spectrum bills separately per TB scanned on top of cluster cost** — pushing hot, frequently-joined data into Redshift local storage vs leaving it in S3 is a deliberate cost/latency decision, not a default.
  - **Glue Data Catalog is Regional** — a "global catalog" story requires explicit cross-Region replication or federation design; reviewers gate on where the catalog of record lives.
  - **Zero-ETL (Aurora→Redshift) replaces pipelines only for supported engines/versions** — check the compatibility matrix before promising "no ETL" in a proposal.
- **Source**: docs.aws.amazon.com/architecture-diagrams/latest/modern-data-analytics-on-aws/modern-data-analytics-on-aws.html

### Multi-VPC hub-and-spoke networking (AWS Transit Gateway)
- **Use when**: More than a handful of VPCs (multi-account AWS Organization, shared
  services, hybrid connectivity) where full-mesh VPC peering is unmanageable. Transit
  Gateway is AWS's managed hub-and-spoke: one Regional TGW connects thousands of VPCs plus
  VPN/Direct Connect, with TGW route tables providing segmentation (prod vs non-prod vs
  shared-services domains). TGWs are highly available by design — one per Region suffices;
  use multiple only to limit misconfiguration blast radius.
- **Shape**: Spoke VPCs attach to a TGW living in a dedicated Network Services account and
  shared to workload accounts via AWS RAM; egress, inspection, and hybrid connectivity are
  centralized.
  ```mermaid
  flowchart LR
    A[Spoke VPC prod] --> TGW{Transit Gateway}
    B[Spoke VPC non-prod] --> TGW
    C[Shared services VPC] --> TGW
    TGW --> INS[Inspection VPC · GWLB + firewall]
    TGW --> EGR[Egress VPC · NAT GW]
    TGW --> DXG[DX Gateway / Site-to-Site VPN]
    DXG --> ONP[On-premises]
    RAM[AWS RAM share] -.multi-account.- TGW
  ```
  Ownership: TGW lives in the Network Services account, shared via AWS Resource Access
  Manager across the Organization — network engineers keep the route tables, workload
  teams only create attachments. Segmentation: associate each attachment with one TGW
  route table and control propagation to build isolated routing domains; steer east-west
  and egress traffic through a centralized inspection VPC using Gateway Load Balancer,
  with **appliance mode** on the inspection attachment so flows stay symmetric across AZs.
  Hybrid: terminate Direct Connect (DX Gateway, transit VIF) and Site-to-Site VPN on the
  TGW; ECMP across dynamic-routing VPN tunnels for aggregate bandwidth; TGW Connect
  (GRE + BGP, up to 4 peers × 5 Gbps per Connect attachment) for SD-WAN appliances. Ops:
  TGW Flow Logs plus AWS Network Manager for topology/monitoring.
- **Gotchas**:
  - **TGW has a per-attachment hourly charge plus per-GB data processing** — VPC peering carries no data-processing fee, so for a few chatty VPCs peering is materially cheaper; hybrid designs (TGW for reach, direct peering for high-volume pairs) are legitimate.
  - **Key default quotas**: 5,000 attachments and 20 route tables per TGW, **10,000 total routes across all its route tables** (raise via SA/TAM, not self-service), 5 TGWs per account, 50 peering attachments — the 10,000-route ceiling bites large orgs first; summarize aggressively.
  - **Bandwidth is per-AZ, per-attachment**: up to 100 Gbps each direction per VPC attachment per AZ, but each individual VPN tunnel caps at ~1.25 Gbps — design ECMP across multiple tunnels for real hybrid throughput, and ECMP requires dynamic (BGP) routing, not static.
  - **MTU mismatch during peering→TGW migration drops jumbo frames**: TGW supports 8,500 bytes (VPC peering 9,001; VPN only 1,500), and no Path MTU Discovery on VPN/DX/peering attachments — cut both sides over simultaneously or asymmetric jumbo packets black-hole.
  - **TGW route tables are not security controls** — reachability isolation only; reviewers gate on security groups/NACLs/Network Firewall still being enforced per-VPC (security groups can't reference groups across TGW, unlike VPC peering).
  - **Inspection VPC without appliance mode breaks stateful firewalls** — cross-AZ flows hash to different appliances asymmetrically; appliance mode on the inspection attachment is the single most-missed checkbox in centralized-inspection designs.
- **Source**: docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-gateway.html
