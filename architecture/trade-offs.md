# Architectural Trade-offs & Strategic Decisions

This document outlines the "Why" behind the Synapse Tax Engine architecture, focusing on the balance between technical scalability, German legal compliance, and startup pragmatism.

## 1. Deployment Model: Modular Monolith vs. Microservices

**Decision:** We utilize a **Modular Monolith** for core business domains (Taxation, Customer management, Notifications) with **Satellite Microservices** for resource-intensive tasks (Ingestion, AI Orchestration).

* **The "Why":** Microservices introduce a "DevOps Tax" (network latency, distributed tracing, complex CI/CD) that can slow down a small founding team. By starting with a Modular Monolith, we maintain high velocity.
* **Strategic Scaling:** We isolate the **Ingestion** and **AI** layers because they have different scaling profiles. If we process 100,000 documents, we can scale those workers to 50 instances while keeping the Core Monolith at 2 instances, optimizing infrastructure costs.
* **Team Ready:** The core is designed with strict bounded contexts. When the team grows from 3 to 20, we can "snap off" modules into independent services without a full rewrite.

## 2. Intelligence: Deterministic Orchestration vs. Autonomous Agents

**Decision:** Business logic is owned by a **Hard-coded State Machine** (Taxation Service). AI Agents act as "Specialized Workers" rather than autonomous decision-makers.

* **The "Why":** In the German tax market, reliability and auditability are non-negotiable. An autonomous LLM "loop" is non-deterministic and expensive.
* **Benefit:** By using the State Machine as the conductor, we ensure 100% predictable workflows. We know exactly why a filing is "Pending," and we can provide an audit trail for every AI-generated suggestion.

## 3. Privacy: Zero-PII Pipeline vs. Third-Party PII Scrubbers

**Decision:** Mandatory **In-House Anonymization Agent** as the gateway to the AI swarm.

* **The "Why":** Under §203 StGB, we cannot risk PII reaching a third-party LLM provider. While solutions like AWS Comprehend are available, owning the Anonymization step ensures we control the exact placeholders used for our specific tax domain.
* **The Moat:** Owning the anonymization layer allows us to market the platform as "Privacy-First," a critical requirement for traditional German tax advisories.

## 4. Communication: Event-Driven Architecture (EDA) vs. Synchronous REST

**Decision:** Asynchronous **Event Bus** for the document-to-intelligence pipeline.

* **The "Why":** Ingestion and AI analysis are time-consuming. Synchronous REST calls lead to timeouts and a "stuck" UI.
* **User Experience:** By using EDA, the UI remains responsive. The user sees a "Processing..." status while the workers handle the heavy lifting in the background. If a service or API is down, the message stays in the queue, ensuring no data loss.

## 5. Summary of Service Count (MVP Phase)
To maximize the 1-year runway, we maintain only **4 main deployable units**:
1.  **Synapse-Core:** (Monolith: Customer management, Taxation, Notification).
2.  **Limetax-IQ:** (AI Orchestration & Swarm).
3.  **Ingestion-Hub:** (Document Service & Adapters).
4.  **Identity Provider:** (Managed Keycloak/Cognito).

## 6. Infrastructure Optimization & Cost Efficiency

**Decision:** We utilize a **Hybrid AI Strategy** (Local ML for Processing + LLM for Reasoning) to minimize costs and maximize VPC data residency.

### 6.1 Hybrid Model Strategy (ML vs. LLM)
To reduce token burn and latency, we distinguish between "Surgical" and "Cognitive" tasks:

| Task | Tech Stack | Rationale |
| :--- | :--- | :--- |
| **Anonymization** | Microsoft Presidio (Local) | Faster and more consistent for NER; keeps PII inside the VPC. |
| **Embedding** | Hugging Face TEI (Local) | Zero token cost for vectorization; avoids sending raw text to third-party APIs. |
| **Extraction** | Claude 3.5 Sonnet (LLM) | High-reasoning required for messy document layouts and tax logic. |
| **Compliance/Optimization** | Claude 3.5 Sonnet (LLM) | Interprets German tax code and cross-references checklist items. |

### 6.2 Back-of-the-Envelope Cost Analysis (Example: 1,500 Documents)
For a mid-sized GmbH filing (~1,000 PDFs and 500 scanned bills), the estimated infrastructure and model costs are optimized via our decoupled architecture:

* **LLM Token Usage:** * ~3.5k Tokens/Doc (Extraction + Audit).
    * **Cost:** ~€21.00 per client filing.
* **Compute (AWS Fargate Spot):** * Leveraging Spot instances for bursty ingestion/AI workloads reduces compute costs by ~70%.
    * **Cost:** ~€3.00 for 2 hours of parallel processing.
* **Storage & Fixed Infra:** * S3 (Originals + Scrubbed), DocumentDB, and Vector DB (shared across tenants).
    * **Cost:** ~€14.00 (Pro-rated per client).

**Total Estimated COGS:** **~€38.00 per filing.** ### 6.3 The "Scale" Moat
* **Independent Scaling:** During peak tax season (March/April), we can scale the `Ingestion-Hub` and `Intelligence` workers to 50+ instances to handle 100k+ documents without impacting the responsiveness of the `Synapse-Core` monolith.
* **Infrastructure as a Moat:** By moving Anonymization and Embedding to local ML workers in our VPC, we reduce our variable LLM costs by ~30% compared to a "Naive LLM" architecture, while simultaneously strengthening our §203 StGB compliance posture.