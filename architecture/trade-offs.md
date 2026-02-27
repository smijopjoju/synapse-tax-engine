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