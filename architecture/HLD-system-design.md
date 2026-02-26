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
graph TD
    %% External Data Sources at the Top
    subgraph External_In [Data Sources]
        DATEV[DATEV / Klardaten]
        BANK[Banking APIs / PSD2]
        SP[SharePoint / Cloud]
    end

    %% Ingestion Layer
    subgraph Data_Muscle [Data Ingestion & Storage]
        DIS[Data Ingestion Service]
        DS[Document Service]
        S3[(Restricted S3)]
    end

    %% Core Logic (Modular Monolith)
    subgraph Synapse_Core [Synapse-Core Monolith]
        direction TB
        CMS[Customer Management]
        CTS[Customer Taxation - State Machine]
        NOT[Notification Service]
        FIL[Filing Service]
    end

    %% Expanded Intelligence Swarm
    subgraph Intelligence [Intelligence Swarm]
        direction TB
        IO[Intelligence Orchestrator]
        AA[Anonymization Agent]
        EA[Embedding Agent]
        CA[Compliance Agent]
        OA[Optimization Agent]
        
        IO <--> AA & EA & CA & OA
    end

    %% User Access
    subgraph User_Access [Identity & Portals]
        IAM[Keycloak / IAM]
        AP[Advisor Portal]
        CP[Client Portal]
    end

    %% Authority at the Bottom
    subgraph External_Out [Regulatory Filing]
        ELSTER[Tax Authority / ELSTER]
    end

    %% Flow: Ingestion
    External_In -->|OAuth2/mTLS| DIS
    DIS <--> DS
    DS <--> S3
    DIS -- Normalized Data --> CTS

    %% Flow: Processing & Intelligence
    CTS -- Event Bus --> IO
    
    %% Flow: Human Interaction
    AP & CP --> IAM
    AP & CP --> CTS
    AP & CP --> CMS

    %% Flow: Final Mile
    CTS --> FIL
    FIL -->|Secure REST| ELSTER

    %% Styling
    style CMS fill:#f96,stroke:#333,stroke-width:2px
    style IO fill:#9f9,stroke:#333,stroke-width:2px
    style AA fill:#99f,stroke:#333,stroke-width:1px
    style External_In fill:#f5f5f5,stroke:#999,stroke-dasharray: 5 5
    style External_Out fill:#f5f5f5,stroke:#999,stroke-dasharray: 5 5