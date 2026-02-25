# Product Roadmap: Synapse Tax Engine

## Phase 1: Onboarding Backfill & Data Ingestion
The primary friction point in tax advisory is the initial data collection. Synapse automates this via:
- **Automated Historical Ingestion**: Extracting data from legacy PDFs, DATEV exports, and bank statements to build a multi-year financial profile.
- **Intelligent Backfilling**: Identifying missing data points and automatically flagging them for client review.

## Phase 2: Interactive Client Checklist
Replacing static questionnaires with a dynamic, AI-driven experience:
- **Context-Aware Queries**: The system only asks for documentation relevant to the client's specific tax situation (e.g., foreign income, rental properties).
- **Real-time Validation**: AI checks uploaded documents immediately for legibility and relevance.

## Phase 3: Advisor Suggestion Model (Human-in-the-Loop)
The core engine of the platform:
- **Proactive Suggestions**: The engine generates draft tax declarations and optimization strategies.
- **Verification Workflow**: Advisors receive a "Confidence Score" for each suggestion. They can approve, modify, or reject.
- **Feedback Loops**: Every advisor interaction is captured. Rejections are categorized to identify systemic model errors or required updates to the Domain RAG.

## Success Metrics
- Reduction in time-to-onboard (TTO).
- Advisor approval rate of AI-generated suggestions.
- Completeness of automated data extraction from non-structured sources.
