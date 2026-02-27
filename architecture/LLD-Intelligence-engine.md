# Low-Level Design: Intelligence Engine

## 1. Overview
Intelligence Engine is the agentic core of the Synapse Tax Engine. It operates as a deterministic "Intelligence Swarm" where specialized agents perform discrete tasks within an asynchronous, event-driven framework. The system is designed for **Zero-PII exposure**, ensuring that the LLM models only interact with sanitized, UUID-linked data.

## 2. The Data Foundation & Storage Model
To balance legal compliance (§203 StGB) with AI performance, a dual-storage strategy is implemented:

* **Physical Vault (S3):** Stores original, un-scrubbed documents. Partitioned by `tenant_id`. Access is restricted to pre-signed URLs generated for Human-in-the-Loop review only.
* **Metadata & Document Store (DocumentDB/MongoDB):** Acts as the "Glue." It stores:
    * `document_uuid`: Unique identifier used across all services.
    * `customer_id`: PII-free UUID from the Customer Management Service.
    * `s3_resource_uri`: Pointer to the raw file.
    * `scrubbed_content_uri`: Pointer to the sanitized text file.
    * `taxation_metadata`: Structured JSON extracted by the Ingestion Agent.

## 3. The Agentic Ingestion Pipeline (Sequence)



### Phase A: Sanitization
1.  **Ingestion Hub** pulls a document and notifies the **Anonymization Agent**.
2.  The Agent retrieves the `Customer_UUID` and `Address_Placeholder` from the **Customer Management Service**.
3.  **Anonymization Agent** performs Named Entity Recognition (NER) to replace real PII with these placeholders.
4.  The **Scrubbed Text** is saved to S3/Document DB as the "Source of Truth" for all AI workers.

### Phase B: Structured Extraction & Vectorization
1.  **Ingestion Agent** processes the scrubbed text to populate the **Taxation Schema** (Inv_Date, Net_Amount, Tax_Rate, etc.).
2.  **Embedding Agent** generates vectors for the scrubbed text and metadata, injecting them into the **Context RAG** (Vector DB) with strict `tenant_id`, `customer_id` metadata filters.

### Phase C: Automated Audit (Compliance Agent)
1.  The **Compliance Agent** triggers the "Road to Filing" workflow.
2.  It performs a proximity search in the **Context RAG** for similar historical cases and cross-references findings against the **Domain RAG** (Statutory Laws).
3.  It produces a **Suggestion Package** containing:
    * `Action_Item`: (e.g., "Missing signature detected").
    * `Reasoning`: The specific tax code or checklist item violated.
    * `Confidence_Score`: Metric to determine if immediate human intervention is needed.

```mermaid
sequenceDiagram
    autonumber
    participant IH as Ingestion Hub
    participant MDS as Intelligence Orchestrator (DocDB)
    participant DV as Customer Management Service
    participant AA as Anonymization Agent
    participant IA as Ingestion Agent
    participant EA as Embedding Agent
    participant CR as Context RAG (Vector DB)
    participant CA as Compliance Agent
    participant DR as Domain RAG (Laws)

    IH->>MDS: Trigger Sanitization (Raw Doc)
    MDS->>AA: Trigger Sanitization (Raw Doc)
    AA->>DV: Fetch UUIDs & Placeholders
    DV-->>AA: Return Customer_UUID
    AA->>MDS: Save Scrubbed Text & Metadata
    
    MDS->>IA: Trigger Extraction (Scrubbed Text)
    IA->>MDS: Update taxation_metadata (JSON)
    
    MDS->>EA: Trigger Vectorization
    EA->>CR: Store Vectors (with Tenant/Cust Filter)
    
    MDS->>CA: Trigger Audit Workflow
    CA->>CR: Proximity Search (Context)
    CR-->>CA: Historical/Specific Data
    CA->>DR: Query Statutory Rules
    DR-->>CA: Tax Code/Checklist
    CA->>MDS: Save Suggestion Package
```

## 4. Human-in-the-Loop (HITL) & Feedback Loop



The system employs a "Verification Service" to capture advisor interactions, creating a data flywheel for model improvement.

### The Verification Flow:
1.  **Re-hydration:** When the Advisor views a suggestion, the UI fetches the PII from the **Customer Management Service** using the `customer_id(UUID)` to display the real client name and address.
2.  **Action Logging:** The Advisor clicks **Agree** (Success) or **Reject** (Correction).
3.  **Telemetry:** If rejected, the Advisor provides a "Correction Reason." This data is stored in the **Evaluation Store(Document DB)**.
4.  **Continuous Learning:** * **Low Threshold (<10% rejections):** Auto-updates the "Few-Shot" examples in the Compliance Agent's prompt.
    * **High Threshold (>15% rejections):** Triggers an engineering alert for a **Domain RAG** update or a model fine-tuning cycle.

## 5. Security & Compliance Safeguards
* **Vector Isolation:** Every query to the Vector DB is with a `WHERE tenant_id = X and customer_id = Y` filter.
* **Auditability:** Every "Re-hydration" event (mapping UUID to PII for UI display) is logged in an immutable audit trail, satisfying German professional secrecy requirements.
* **Deterministic Fallback:** If an agent fails to return a result within the confidence threshold, the **Customer Taxation Service** transitions the state to `MANUAL_REVIEW_REQUIRED`.

```mermaid
sequenceDiagram
    autonumber
    participant AP as Advisor Portal
    participant IO as Intelligence Orchestrator
    participant MDS as Metadata Store (DocDB)
    participant CM as Customer Management Service (Vault)
    participant CT as Customer Taxation Service (Statemachine)

    AP->>IO: Fetch Suggestion Package
    IO->>MDS: Fetch Suggestion Package (customer_id)
    MDS-->>IO: Suggestion Package
    IO-->>AP: Suggestion Package
    AP->>CM: Request Re-hydration (customer_id)
    CM-->>AP: Return PII (Name/Address)
    Note over AP: Advisor reviews Suggestion + PII
    
    alt Advisor Accepts
        AP->>IO: Log Success
        IO->>MDS: Update Status: VALIDATED
        IO->>CT: Prepare TAX submitting
        CT->>CM: Fetch Re-hydrated PII for Final Authority Payload
    else Advisor Rejects
        AP->>IO: Log Rejection + Reason
        IO->>CT: Manual Review
        Note over IO: Notification to Advisor
        IO->>MDS: Store for Retraining
        IO->>IO: Trigger Alert (if >15% Threshold)
        IO->>IO: Update Few-Shot / Retrain
    end
```