# Agentic Strategy: Dual-Retrieval-Augmented Generation & Tax-Specific Large Language Models

## Architecture: Dual-Retrieval-Augmented Generation (RAG)
To provide accurate advice, the system must balance general tax law with specific client data through two distinct Retrieval-Augmented Generation (RAG) pipelines:
1. **Domain RAG**:
   - **Scope**: Statutory laws, case law, DATEV circulars, and official tax guidelines.
   - **Update Frequency**: High; tracked against legislative changes.
2. **Context RAG**:
   - **Scope**: Client-specific documents, historical filings, correspondence, and accounting data.
   - **Security**: Strictly isolated per client/tenant to maintain §203 StGB compliance.

## The Role of the Model Context Protocol (MCP) Server
We utilize a Model Context Protocol (MCP) Server to standardize how Large Language Model (LLM) agents interact with both external tax tools and internal infrastructure:
- **Internal Service Exposure**: The MCP Server acts as the central orchestration layer, exposing all internal microservices—including tax filing state engines, document storage, and CRM systems—to allow agents to automate the end-to-end advisory process.
- **Context Injection**: Providing agents with a standardized view of the client's current financial state without exposing the entire database.

## Large Language Model (LLM) Agents
- **Anonymization Agent**: A specialized model dedicated to data privacy. It sanitizes content before it reaches broader processing layers, employing either text masking (blocking) or randomizing sensitive identifiers to ensure strict compliance with §203 StGB.
- **Embedding Agent**: A dedicated model used to vectorize document text and metadata, ensuring high-fidelity ingestion into the Contextual Retrieval-Augmented Generation system.
- **Ingestion Agent**: Specialized in Optical Character Recognition (OCR) and structured data extraction from unstructured tax documents.
- **Compliance Agent**: Cross-references draft filings against the Domain RAG to ensure no regulations are violated.
- **Optimization Agent**: Scans for potential deductions or tax-saving structures based on the client's Context RAG.

## Model Training & Fine-Tuning
We do not rely on static models. Retraining or Low-Rank Adaptation (LoRA) triggers include:
- **Rejection Thresholds**: If an advisor rejects a specific category of suggestions >15% of the time, the model's prompting or underlying weights are reviewed.
- **Legislative Updates**: Major tax reform triggers a re-indexing of the Domain RAG and potential re-evaluation of current suggestion logic.
