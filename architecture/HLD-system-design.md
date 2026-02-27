# High-Level Design: Synapse Tax Engine

## 1. Architectural Philosophy: The Technical Copilot
Synapse is architected as an **Intelligence-First Data Platform**. The core principle is the strict separation of the **Deterministic Legal State** from **Non-Deterministic Agentic Reasoning**.

To achieve a 1-year runway with maximum velocity, we adopt a **Modular Monolith** deployment for core services, while maintaining strict domain boundaries that mirror our target **Microservices Architecture**.

---

## 2. Current Deployment: The Modular Monolith (Day 1)
In the initial phase, we co-locate core business logic to minimize "DevOps Tax" and inter-service latency.



### Service Grouping:
1.  **Synapse-Core (Monolith)**:
    * **Customer Management Service**: The "Source of Truth" and PII Vault.
    * **Customer Taxation Service**: The State Machine orchestrating the "Road to Filing."
    * **Notification Service**: Event-driven alerts (Bloomreach/Twilio).
2.  **Intelligence Orchestrator (Satellite)**: Dedicated Python/FastAPI bridge for the Agent Swarm.
3.  **Data Ingestion Service (Satellite)**: High-throughput worker for DATEV, Bank, and SharePoint adapters.
4.  **Document Service (Satellite)**: Stateless S3/MinIO binary management.

---

## 3. Destination Architecture: The Microservices Ecosystem (Target)
As the platform scales to handle the €20B German market, the modules within **Synapse-Core** are extracted into independent, autoscaling microservices.



```mermaid
graph TB

%% =========================
%% Portal Layer
%% =========================
subgraph L1_Portal_Layer
  AP[Advisor Portal - React]
  CP[Client Portal - React]
end

%% =========================
%% Service Layer
%% =========================
subgraph L2_Service_Layer

  IAM[Identity Provider IAM]

  subgraph CORE_Synapse_Core_Service
    CMS[Customer Management - PII Vault]
    CTS[Customer Taxation - State Machine]
    NOTIF[Notification Service]
    FIL[Filing Service]
  end

  subgraph IQ_Intelligence_Layer
    IO[Intelligence Orchestrator]
    subgraph AI_Agents
        AA[Anonymization Agent]
        IA[Ingestion Agent]    
        EA[Embedding Agent]
        CA[Compliance Agent]
        OA[Optimization Agent]
    end
    
  end

end

%% =========================
%% Data Layer
%% =========================
subgraph L3_Data_Layer_EU
  S3[Object Store - S3 or MinIO]
  MDS[Metadata Store - Document DB]
  VDB[Vector Database - Context RAG]
  DR[Domain Knowledge Store - Laws]
  AUD[Immutable Audit Log]
  RDMS[Postgres RDBMS]
end

%% =========================
%% Ingestion Layer
%% =========================
subgraph L4_Ingestion_Service

  subgraph SRC_External_Source_adapter
    DATEV[DATEV API]
    BANK[Open Banking API]
    SP[SharePoint API]
  end
  
  DIS[Data Ingestion Service]
  DS[Document Service]

end

%% =========================
%% External Filing
%% =========================
ELSTER[Tax Authority ELSTER]

%% ---- Auth ----
AP --> IAM
CP --> IAM

AP --> CTS
CP --> CTS
AP --> CMS
CP --> CMS

%% ---- Ingestion ----
DATEV --> DIS
BANK --> DIS
SP --> DIS

DIS --> DS
DS --> S3
DIS --> MDS
DIS --> CMS

%% ---- Workflow ----
CTS --> IO

IO --> VDB
IO --> DR
IO --> MDS
IO --> AUD

AA -.- IO
IA -.- IO
EA -.- IO
CA -.- IO
OA -.- IO

FIL --> RDMS
CMS --> RDMS
CTS --> RDMS
NOTIF --> RDMS


%% ---- Filing ----
CTS --> FIL
FIL --> ELSTER
```