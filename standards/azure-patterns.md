# Azure Reference Patterns

Reusable, client-anonymized building blocks I reach for on Azure. Grown over time by
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

## Seeded reference patterns (official Azure Architecture Center)

Baselines distilled from Microsoft's Azure Architecture Center / Well-Architected docs
(fetched 2026-07-10). No engagement evidence yet — these carry a **Source** field instead;
add won/lost evidence as we use them. Treat them as the "defensible default" to deviate
from consciously, not gospel.

### Well-Architected design-review baseline (not a topology — a checklist)
- **Use when**: Reviewing any Azure design before it goes to a client, or when a client asks
  "is this best practice?" The five pillars and the one-line concern of each:
  - **Reliability** — resiliency, availability, recovery; design for business requirements, keep it simple.
  - **Security** — protect confidentiality, integrity, availability; threat detection and mitigation.
  - **Cost Optimization** — cost modeling, budgets, reduce waste; optimize usage and rate.
  - **Operational Excellence** — observability, DevOps practices, safe deployment.
  - **Performance Efficiency** — scale horizontally, test early and often, monitor health.
- **Gotchas**: Pillars trade off against each other (zone redundancy vs cost, inspection vs
  latency) — state the trade-off you chose, don't hide it. Azure Advisor scores subscriptions
  against these same pillars, so clients may independently audit your design with it.
- **Source**: learn.microsoft.com/azure/well-architected/pillars

### Baseline zone-redundant web app (App Service + App Gateway/WAF + Private Link)
- **Use when**: A production web app on App Service needs a defensible security/HA posture —
  single hardened public entry point, everything else private, survives loss of an
  availability zone. This is Microsoft's own "production floor" for App Service; use it as
  the deployment view enterprise reviewers gate on.
- **Shape**:
  ```mermaid
  flowchart LR
    user([Users]) --> agw[App Gateway + WAF<br/>public entry, TLS, autoscale]
    agw -.private endpoint.-> app[App Service<br/>zone-redundant, ≥2 instances]
    app -.VNet integration +<br/>private endpoints.-> paas[(SQL DB / Storage / Key Vault<br/>public access disabled)]
    dns{{Private DNS zones}} -.resolve PE IPs.- agw & app
  ```
  Three subnets (gateway / app integration / private endpoints), NSG per subnet, private DNS
  zone per service. End-to-end TLS: App Gateway terminates, re-encrypts to App Service; cert
  in Key Vault via managed identity. Identity: Entra ID + EasyAuth, user-assigned managed
  identities for workload-to-PaaS. Deploy via slots + run-from-package; self-hosted pipeline
  agents inside the VNet (public access to App Service is off, so nothing can deploy from outside).
- **Gotchas**:
  - App Gateway needs **≥3 instances across zones** — a replacement instance takes ~7 min to
    start, so one instance is an availability hole, not a baseline.
  - SQL zone redundancy is a **tier feature** (GP/Premium/Business Critical) and backups need
    ZRS/GZRS explicitly — "zone redundant" on the app tier alone is theater.
  - Blocking public access breaks naive CI/CD — plan the self-hosted-agent path up front.
  - Prefer **user-assigned** managed identities; system-assigned ones cause IaC race conditions.
  - Non-PaaS egress still exits via shared public IPs unless you route it through Azure
    Firewall — matters when a partner needs a stable IP to allowlist.
  - Enforce the posture with Azure Policy (deny public network access, require VNet
    integration/Private Link) or it drifts.
- **Source**: learn.microsoft.com/azure/architecture/web-apps/app-service/architectures/baseline-zone-redundant

### Baseline network-secured AI chat (Foundry Agent Service)
- **Use when**: An enterprise chat/agent application (RAG or tool-using) must be network-
  isolated and production-grade — the AI analogue of the baseline web app above, which it
  builds on for the UI tier. Microsoft has retitled the old "baseline OpenAI chat" to
  **Microsoft Foundry** chat — use the current naming with clients.
- **Shape**:
  ```mermaid
  flowchart LR
    user([Users]) --> agw[App Gateway + WAF] -.PE.-> ui[Chat UI<br/>App Service]
    ui -.private endpoint,<br/>managed identity.-> agent[Foundry Agent Service<br/>managed agent runtime]
    agent -.PE.-> search[(Azure AI Search<br/>RAG grounding)]
    agent --> deps[(Dedicated Cosmos DB + Storage<br/>agent state & chat history)]
  ```
  Foundry Agent Service hosts the agents (standard agent setup for network isolation); you
  supply its state stores — Cosmos DB (conversations), Storage (files), AI Search — in your
  own subscription. All service-to-service hops over private endpoints with private DNS.
- **Gotchas**:
  - **Dedicate** the agent-service Cosmos DB/Storage/AI Search instances — never share them
    with app workload data; Foundry manages their contents exclusively and shared instances
    couple blast radii.
  - **No built-in DR**: Agent Service can't replicate, back up, or point-in-time restore its
    state — recovery is reconstruction. Put delete locks on the dependencies and set client
    expectations about possible total conversation-state loss.
  - Portal-created Foundry projects **bypass your network controls** (no inherited private
    endpoints/NSGs) — allow project creation only through IaC.
  - Agent reliability = min(availability of Cosmos DB, Storage, AI Search) — you own
    configuring their redundancy; Foundry's own plane has no zone-redundancy knobs.
  - Multi-agent orchestration beyond simple patterns needs self-hosted orchestration
    (Agent Framework SDK), not the managed service alone — scope accordingly.
- **Source**: learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat

### Medallion lakehouse (Databricks bronze/silver/gold)
- **Use when**: Building any Databricks-based data platform where data quality must improve
  progressively from raw ingest to business-consumable marts. The default organizing pattern
  for lakehouse engagements; clients increasingly name-check it, so align terminology.
- **Shape**:
  ```mermaid
  flowchart LR
    src[Sources<br/>files · Kafka · SaaS · federation] --> bronze[(Bronze<br/>raw, append-only,<br/>original formats)]
    bronze --> silver[(Silver<br/>validated, deduped,<br/>conformed entities)]
    silver --> gold[(Gold<br/>dimensional models,<br/>aggregates per domain)]
    gold --> out[BI · ML · APIs]
  ```
  One schema (or catalog layer) per tier under Unity Catalog. Bronze: no cleanup, keep
  fields as string/VARIANT against schema drift, add provenance metadata columns. Silver:
  schema enforcement, dedup, late-data handling, joins, at least one validated non-aggregated
  representation of every record. Gold: dimensional modeling, materialized-view aggregates,
  possibly one gold layer per business domain (finance, HR…).
- **Gotchas**:
  - **Never write silver directly from ingestion** — schema changes and corrupt records in
    sources will break it; land in bronze first, stream bronze→silver.
  - Bronze is the reprocessing/audit source of truth — deleting it to save storage removes
    your ability to rebuild silver/gold after logic fixes.
  - Ingestion frequency is the main cost lever (continuous vs triggered vs batch) — agree
    the latency requirement with the client before defaulting to streaming.
  - Aggregates belong in gold, not silver — silver aggregates quietly become unowned
    semantic models (ties into the data-model-ownership gotcha in the pattern above).
- **Source**: learn.microsoft.com/azure/databricks/lakehouse/medallion

### Hub-spoke network topology (customer-managed hub)
- **Use when**: A client has (or will grow into) multiple workloads/environments needing
  shared services — central egress control, cross-premises connectivity, DNS, firewalling —
  with workload isolation per spoke. This is the Cloud Adoption Framework's recommended
  topology and the basis of Azure landing zones; assume enterprise reviewers expect it.
- **Shape**:
  ```mermaid
  flowchart LR
    onprem[On-prem network] -->|VPN / ExpressRoute| hub[Hub VNet<br/>Firewall · Bastion · Gateway · DNS]
    hub <-.peering.-> s1[Spoke: prod workload]
    hub <-.peering.-> s2[Spoke: nonprod]
    s1 & s2 -.UDR: egress via<br/>hub firewall.-> hub
  ```
  One hub **per region**; spokes peer only to their regional hub. GatewaySubnet and
  AzureFirewallSubnet both /26 minimum. Spoke-to-spoke via hub firewall (UDRs) by default;
  direct peering only for high-throughput, low-risk flows within one workload. Azure Virtual
  Network Manager for scale (network groups, security admin rules); Virtual WAN is the
  Microsoft-managed alternative when the client doesn't want to run the hub.
- **Gotchas**:
  - **Peering is non-transitive** — spoke-to-spoke doesn't work by default; it's a routing
    decision you must make explicitly (firewall hop vs direct peering), each with cost/latency
    consequences.
  - Everything egressing via the firewall is **SNAT'd to the firewall's public IPs** — partner
    allowlists must cover the whole set, and the SNAT port pool caps concurrent outbound
    connections (NAT Gateway on the firewall subnet raises the ceiling).
  - Keep firewall/spoke **DNS resolution aligned** (use the firewall as DNS proxy) or FQDN
    rules block legitimate traffic when resolutions diverge.
  - Firewall processing is a real cost — high-volume trusted flows (DB sync, bulk copy) may
    justify direct peering to bypass it.
  - DDoS protection belongs on any VNet with public IPs — including the firewall's and
    gateway's own IPs, even if no workload is public.
- **Source**: learn.microsoft.com/azure/architecture/networking/architecture/hub-spoke
