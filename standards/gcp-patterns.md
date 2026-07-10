# GCP Reference Patterns

Reusable, client-anonymized building blocks I reach for on GCP. Grown over time by
/retro and manual additions. Each pattern: when to use, the shape of it, gotchas learned.

## How to add a pattern
Use this structure so /diagram can consume them consistently:

### <Pattern name>
- **Use when**: <situation / requirements profile>
- **Shape**: <components and how they connect — prose or a small Mermaid sketch>
- **Gotchas**: <what bit us before: quotas, latency, cost cliffs, compliance>
- **Won/lost evidence**: <anonymized: "well received in 2 pitches", "cost pushback once">

## Patterns

(empty — will grow with use. Seed it by asking Claude to write up the 3-5 GCP
architectures you draw most often.)

## Seeded reference patterns (official Google Cloud docs)

Baselines distilled from the Google Cloud Well-Architected Framework / Architecture Center
docs (fetched 2026-07-10). No engagement evidence yet — these carry a **Source** field
instead; add won/lost evidence as we use them. Treat them as the "defensible default" to
deviate from consciously, not gospel. Note the Architecture Center now lives at
docs.cloud.google.com (cloud.google.com/architecture/* redirects there).

### Google Cloud Well-Architected Framework — design-review baseline (not a topology, a checklist)
- **Use when**: Run this checklist before any GCP design review or proposal sign-off. Note:
  the framework has been **renamed** from "Google Cloud Architecture Framework" to the
  **Google Cloud Well-Architected Framework**. Current structure is six pillars plus two
  cross-cutting "perspectives":
  - **Operational excellence** — can you deploy, operate, monitor, and manage the workload efficiently (IaC, CI/CD, observability, incident response)?
  - **Security, privacy, and compliance** — maximize security posture and data protection while meeting regulatory obligations (IAM least privilege, encryption, perimeter controls).
  - **Reliability** — resilient, highly available design that withstands zone/region failures; explicit SLOs and recovery objectives.
  - **Cost optimization** — framed as maximizing business value of spend, not minimizing spend; requires visibility and continuous right-sizing.
  - **Performance optimization** — resources designed and tuned to the workload's actual performance requirements.
  - **Sustainability** — environmental footprint of the workload (region choice, utilization); the pillar reviewers most often skip.
  - **Perspective: AI and ML** — technology-specific recommendations layered across all pillars for AI/ML workloads.
  - **Perspective: Financial services** — industry-specific recommendations for FS workloads.
- **Gotchas**:
  - The framework's stated core design principles are **design for change, document the architecture, simplify (prefer fully managed services), decouple, and use stateless architecture** — expect reviewers to challenge any stateful or tightly coupled component against these.
  - Pillars interact: the FS perspective's own example is that **data residency requirements constrain your failover-region choice**, which dictates achievable RTO/RPO — don't score Security and Reliability independently.
  - **Cost Optimization is defined as business value, not lowest bill** — a design rejected purely on list price mis-applies the pillar; bring value/unit-economics arguments.
  - For AI/ML or FS engagements, **apply the perspective on top of, not instead of, the six pillars** — perspectives refine pillar guidance, they don't replace it.
  - The framework gives principles and recommendations, **not a formal scoring/review methodology** — supply the scoring rubric yourself if the client expects a WAF-style report.
- **Source**: docs.cloud.google.com/architecture/framework

### Baseline scalable and resilient web application (global front end + serverless backend)
- **Use when**: Any internet-facing production web app on GCP that needs global reach,
  DDoS/WAF protection, CDN offload, and multi-region resilience without server management.
  This is the current canonical baseline; Google's older "Scalable and Resilient Web
  Applications" paper and the three-tier Jump Start Solutions are retired.
- **Shape**: A programmable global front end (global external Application Load Balancer +
  Cloud Armor + Cloud CDN) in front of interchangeable backends — Cloud Run, GKE, MIGs, or
  hybrid — with data on Cloud SQL over private IP.
  ```mermaid
  flowchart LR
    U[Users] --> CA[Cloud Armor edge + WAF policies]
    CA --> CDN[Cloud CDN]
    CDN --> GLB[Global external Application LB]
    GLB --> CR1[Cloud Run · region A]
    GLB --> CR2[Cloud Run · region B]
    CR1 --> SQL[(Cloud SQL, private IP)]
    CR2 --> SQL
    CR1 -.-> SM[Secret Manager]
    CR1 -.-> VPC[Shared VPC via VPC egress]
  ```
  Traffic hits Cloud Armor edge security policies first, serves from Cloud CDN on cache
  hit, and only reaches backend services (and backend security policies) on a miss. Cloud
  Run services run with a dedicated service account (`roles/run.invoker`,
  `roles/secretmanager.secretAccessor`), with ingress locked to "Internal and Cloud Load
  Balancing" so the default `run.app` URL cannot bypass the WAF. Private connectivity to
  Cloud SQL and VPC resources via Serverless VPC Access per the secured serverless
  blueprint — on new builds prefer Direct VPC egress, which post-dates that blueprint.
  Enforce org policies `constraints/run.allowedIngress` and `constraints/run.allowedVPCEgress`,
  wrap the stack in a VPC Service Controls perimeter (egress via `restricted.googleapis.com`),
  CMEK from Cloud KMS for Artifact Registry and Cloud Run images. Baseline the surrounding
  org with the Enterprise Foundations blueprint; Cloud Armor Enterprise adds Adaptive
  Protection and Threat Intelligence over Standard.
- **Gotchas**:
  - **Cloud Run ingress left at "all"** silently bypasses the load balancer, Cloud Armor, and CDN via the default `run.app` URL — reviewers gate on this; lock it with the org policy constraint, not just per-service settings.
  - **Never cache user-specific content in Cloud CDN** — the doc calls this out explicitly as a data-leak risk; set explicit `Cache-Control` on HTML/JSON rather than relying on defaults.
  - Cloud Armor's preconfigured WAF rules are based on **ModSecurity Core Rule Set CRS 3.3** in multiple sensitivity levels — enforce mode without preview-mode tuning generates false-positive outages.
  - **CMEK keys must live in the same region as the resources they protect** (blueprint uses 30-day automatic rotation) — a multi-region rollout multiplies your key inventory.
  - **Effective CDN caching is the cost lever**: cache hits never traverse the load balancer, cutting both latency and LB data-processing charges; a low hit ratio makes the global LB the dominant cost line.
  - The secured-serverless blueprint predates **Direct VPC egress** and only documents Serverless VPC Access connectors — connectors add per-instance cost and throughput ceilings; re-evaluate on current Cloud Run.
- **Source**: docs.cloud.google.com/architecture/deploy-programmable-gfe-cloud-armor-lb-cdn

### Baseline RAG-capable generative AI application (Vertex AI / AlloyDB)
- **Use when**: An enterprise chat/assistant app that must ground LLM answers in private
  documents with auditability and responsible-AI screening. Note the doc has been
  substantially re-branded (last reviewed 2026-02): model hosting is now described as the
  **Gemini Enterprise Agent Platform** rather than plain "Vertex AI", though the URL slug
  is unchanged; sibling reference architectures cover a Vertex AI Vector Search variant, a
  GKE + Cloud SQL variant, and GraphRAG with Spanner Graph.
- **Shape**: Three subsystems — ingestion, serving, and quality evaluation — glued by
  Pub/Sub events, with AlloyDB (pgvector) as the vector store and BigQuery as the offline
  analytics sink.
  ```mermaid
  flowchart LR
    GCS[Cloud Storage uploads] --> PS[Pub/Sub]
    PS --> ING[Cloud Run function · ingest]
    ING --> EMB[Embedding model]
    ING --> ADB[(AlloyDB + pgvector)]
    U[User] --> FE[Frontend app on Cloud Run]
    FE --> ADB
    FE --> LLM[Gemini LLM inference + RAI filters]
    FE --> BQ[(BigQuery · logs and eval scores)]
  ```
  Ingestion: documents land in Cloud Storage, Pub/Sub triggers a Cloud Run function that
  chunks, embeds, and writes vectors to AlloyDB. Serving: the app embeds the user query
  with the same model, does semantic search in AlloyDB, builds the augmented prompt, calls
  the LLM, passes output through responsible-AI filters, and logs to Cloud
  Logging/BigQuery; an evaluation subsystem scores stored responses (factual accuracy,
  relevance) into BigQuery. Identity is service-account based: the AlloyDB Auth Proxy
  provides IAM-based connection authorization over TLS 1.3; VPC Service Controls plus CMEK
  mitigate exfiltration; private service-to-service traffic inside a VPC, no public data
  plane. AlloyDB read pools and Query Insights handle read scaling and performance triage.
- **Gotchas**:
  - The doc stresses twice that the **embedding model and parameters must be identical in ingestion and serving** — drift (a model upgrade on one side) silently breaks semantic search rather than erroring.
  - **AlloyDB HA is the default and doubles node cost** (active + standby); Google explicitly says to drop to basic instances only if you can tolerate a zone outage — a reviewer-visible cost/reliability trade.
  - **AlloyDB pgvector is not the large-scale answer**: Google routes "very large-scale, low-latency" vector workloads to the separate Vertex AI Vector Search reference architecture — know your vector count before committing.
  - **Data residency drives region pinning** of Cloud Storage, AlloyDB, and BigQuery; cross-region replication for DR is called out as added complexity, not a default.
  - **Cloud Logging volume is a cost cliff** for chat workloads — use retention settings and exclusion filters; pipe durable analytics to BigQuery instead of keeping raw logs.
  - BigQuery replicates **synchronously across two zones within one region only** — regional loss of your logging/eval dataset needs an explicit cross-region strategy.
- **Source**: docs.cloud.google.com/architecture/rag-capable-gen-ai-app-using-vertex-ai

### BigQuery-centered open lakehouse (Lakehouse for Apache Iceberg, formerly BigLake)
- **Use when**: A unified analytics platform where BigQuery, Spark, and open-source
  engines must query one governed copy of data in open formats — replacing separate
  warehouse + data-lake stacks. Major rename: as of **2026-04, BigLake is rebranded
  "Lakehouse for Apache Iceberg"** and BigLake Metastore is now the **Lakehouse runtime
  catalog**; the old "analytics lakehouse" Jump Start Solution is deprecated.
- **Shape**: Iceberg tables on Cloud Storage under a single serverless catalog, queried
  interchangeably by BigQuery and open engines, governed centrally via Dataplex.
  ```mermaid
  flowchart LR
    SRC[Source systems] --> ETL[Dataflow / Spark pipelines]
    ETL --> GCS[(Cloud Storage · Iceberg V2 tables)]
    GCS --> CAT[Lakehouse runtime catalog]
    CAT --> BQ[BigQuery]
    CAT --> OSS[Serverless Spark / Flink / Trino / Hive]
    BQ --> BI[Looker / BI]
    BQ --> HUB[Analytics Hub sharing]
    CAT -.-> GOV[Dataplex · Knowledge Catalog governance]
  ```
  Storage and compute fully decoupled: the Lakehouse runtime catalog is a fully managed,
  serverless metadata service exposing an Apache Iceberg REST Catalog API, so BigQuery,
  Google Cloud Serverless for Apache Spark, Flink, Trino, and Hive all read/write the same
  tables as a single source of truth — no per-engine copies. Cross-engine access control
  uses **credential vending** from the catalog rather than direct bucket access; CMEK
  supported; storage cost managed with Autoclass tiering. Governance (policy enforcement,
  lineage, data quality, semantic search) centralized through Dataplex's Knowledge Catalog
  integration rather than per-engine ACLs. Dataform for in-warehouse transformation
  orchestration; Analytics Hub for curated cross-org sharing.
- **Gotchas**:
  - **APIs, client libraries, gcloud commands, and IAM role names still say "biglake"** after the rename — Terraform and policy docs will look inconsistent with the marketing name; don't "fix" them.
  - **Iceberg V1 tables are not supported** — V2 is GA, V3 is Preview; migrating an existing Hive/Iceberg V1 estate requires table rewrites first.
  - **Cross-region replication/DR for the lakehouse is Preview**, and cross-cloud capabilities are Preview — don't anchor an RPO commitment on them yet.
  - **The old jump-start "analytics lakehouse" Terraform is dead** — cite the product architecture, not the deprecated solution, or the client will deploy unmaintained code.
  - Skipping **credential vending** and granting engines direct GCS access reinstates the split-brain permission model the lakehouse exists to remove — reviewers gate on a single authorization path through the catalog.
  - Multiple engines writing the same Iceberg table puts **commit-conflict handling on your pipeline design**; serialize writers per table or partition where possible.
- **Source**: docs.cloud.google.com/lakehouse/docs/introduction

### Hub-and-spoke VPC network topology (NCC vs peering vs VPN)
- **Use when**: Landing-zone network design where multiple workload (spoke) VPCs need
  centralized on-prem connectivity, shared services, and controlled east-west traffic,
  with workload separation preserved. Choose among three official options: Network
  Connectivity Center (preferred at scale), VPC Network Peering, or Cloud VPN; multiple
  Shared VPCs per environment is the alternative to hub-and-spoke, not a variant of it.
- **Shape**: A hub VPC terminates hybrid connectivity and hosts shared services/NVAs;
  spokes attach via NCC, peering, or VPN.
  ```mermaid
  flowchart LR
    ONP[On-premises] --> HYB[Cloud Interconnect / HA VPN]
    HYB --> HUB[Hub VPC + NCC hub]
    HUB --> P[Spoke VPC · prod]
    HUB --> D[Spoke VPC · dev]
    HUB --> SVC[Shared services spoke]
    HUB --> NVA[NVA / NGFW next-hop]
    HUB -.-> PSC[Private Service Connect / producer spokes]
  ```
  With **Network Connectivity Center**, the hub is a control-plane resource (not a
  data-plane transit network): VPC spokes get full VM-to-VM bandwidth, transitive dynamic
  routes, and transitive Private Service Connect between workload VPCs; choose **mesh**
  topology to allow spoke-to-spoke traffic or **star** to forbid it. Producer spokes
  propagate routes for supported Google-managed services automatically. With **VPC Network
  Peering**, each spoke peers with the hub at full bandwidth but peering is strictly
  non-transitive — spoke-to-spoke and spoke-to-managed-service traffic must hairpin
  through an NVA/NGFW in the hub (internal passthrough LB as next hop). With **Cloud
  VPN**, tunnels between hub and spokes buy transitivity and escape peering quotas, at the
  cost of tunnel-bounded bandwidth. Segmentation rides on centralized hierarchical
  firewall policies and DNS private zones with forwarding/peering so on-prem and spokes
  resolve each other.
- **Gotchas**:
  - **VPC Network Peering is not transitive** — the number-one review catch: spoke-to-spoke, spoke-to-on-prem-via-hub, and access to services published in another spoke all fail unless routed through hub NVAs.
  - **Peering quotas cap fan-out** (per-network peering connection and cumulative subnet/route limits) — large landing zones hit them; NCC VPC spokes exist specifically to remove pairwise-peering management at scale.
  - **Cloud VPN bandwidth is the sum of tunnel bandwidths** (roughly 3 Gbps class per tunnel), not per-VM line rate — a transitive-but-throttled hub surprises data-heavy workloads; add parallel tunnels or move to NCC.
  - **NCC star vs mesh is a one-way security decision**: star gives two route-table groups where edge spokes cannot talk to each other — pick it deliberately for env isolation, because retrofitting mesh semantics changes your threat model.
  - **Private Service Connect routes are transitive only under NCC** — under plain peering, PSC endpoints in the hub are unreachable from spokes without extra plumbing.
  - **DNS is a separate design axis**: private zones plus DNS forwarding/peering must be configured explicitly per option; connectivity without name resolution is the most common "it pings but nothing works" ticket.
- **Source**: docs.cloud.google.com/architecture/deploy-hub-spoke-vpc-network-topology
