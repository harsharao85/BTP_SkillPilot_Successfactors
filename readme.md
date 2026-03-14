# SkillPilot — AI Learning Assistant for SAP SuccessFactors

SkillPilot is a Retrieval-Augmented Generation (RAG) service that embeds an AI learning advisor directly into the SAP SuccessFactors Learning experience. It helps employees, managers, and L&D administrators navigate complex SAP training content — courses, learning paths, certifications, and skills — related to **S/4HANA migration**, **SAP BTP**, **Hire-to-Retire**, and **change management**.

Built on SAP CAP (Node.js) with Claude as the AI layer, SkillPilot mirrors the architecture of SAP Generative AI Hub, making it production-ready for SAP BTP with a credential swap.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT (curl / UI)                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │ POST /api/chat { question }
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   SkillPilot RAG Service                         │
│                   srv/rag-service.js                             │
│                                                                  │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │  CSV Loader  │───▶│  TF-IDF Indexer  │───▶│ Cosine Search │  │
│  │  db/data/    │    │  (in-memory)     │    │  top-K chunks │  │
│  └──────────────┘    └──────────────────┘    └───────┬───────┘  │
│                                                       │          │
│                                          ┌────────────▼────────┐ │
│                                          │  Prompt Assembler   │ │
│                                          │  (grounded context) │ │
│                                          └────────────┬────────┘ │
└───────────────────────────────────────────────────────┼──────────┘
                                                        │
                            ┌───────────────────────────▼──────────┐
                            │           Claude API                  │
                            │   claude-sonnet-4-20250514            │
                            │   (Anthropic / SAP Gen AI Hub)        │
                            └──────────────────────────────────────┘
                                                        │
                            ┌───────────────────────────▼──────────┐
                            │   Response + Source Citations         │
                            └──────────────────────────────────────┘

Data Layer (SAP CAP)
┌────────────────────────────────────────────────────┐
│  db/schema.cds    — CDS entity definitions          │
│  db/data/*.csv    — Mock data (10 courses,          │
│                     3 paths, 3 certs, 4 skills)     │
└────────────────────────────────────────────────────┘
```

---

## Project Structure

```
skillpilot/
├── db/
│   ├── schema.cds          # CDS data model (Courses, LearningPaths, Certifications, Skills)
│   └── data/
│       ├── skillpilot-Courses.csv
│       ├── skillpilot-LearningPaths.csv
│       ├── skillpilot-Certifications.csv
│       └── skillpilot-Skills.csv
├── srv/
│   └── rag-service.js      # Express RAG service with TF-IDF vector store + Claude integration
├── .env.example            # Environment variable template
├── .gitignore
├── package.json
└── readme.md
```

---

## Setup

### Prerequisites
- Node.js 18+
- An Anthropic API key ([console.anthropic.com](https://console.anthropic.com/))

### 1. Install dependencies
```bash
npm install
```

### 2. Configure environment
```bash
cp .env.example .env
# Edit .env and set your ANTHROPIC_API_KEY
```

### 3. Start the RAG service
```bash
npm run start:rag
```

The service starts on `http://localhost:4005`.

---

## API Endpoints

### `POST /api/chat`
Ask a question about SAP learning content.

**Request:**
```json
{
  "question": "What course should I take to prepare for an S/4HANA migration project?"
}
```

**Response:**
```json
{
  "answer": "For S/4HANA migration preparation, I recommend starting with...",
  "sources": [
    {
      "id": "course-1",
      "type": "Course",
      "title": "S/4HANA Migration Fundamentals",
      "relevanceScore": 0.42
    }
  ]
}
```

**curl example:**
```bash
curl -X POST http://localhost:4005/api/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I renew my SuccessFactors certification?"}'
```

---

### `GET /api/health`
Returns service status and vector store stats.

```bash
curl http://localhost:4005/api/health
```

---

### `GET /api/content`
Lists all indexed content (useful for browsing the knowledge base).

```bash
curl http://localhost:4005/api/content
```

---

## Sample Questions to Try

```bash
# S/4HANA migration
curl -X POST http://localhost:4005/api/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "What are the steps in the SAP Activate methodology?"}'

# BTP architecture
curl -X POST http://localhost:4005/api/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "How does SAP BTP connect to S/4HANA?"}'

# HR transformation
curl -X POST http://localhost:4005/api/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "What learning path should a change manager take for a SuccessFactors rollout?"}'

# Certifications
curl -X POST http://localhost:4005/api/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "What certifications are available for BTP architects?"}'
```

---

## How It Works

1. **Startup:** All CSV data is loaded into memory and chunked by entity type (Course, Learning Path, Certification, Skill).
2. **Indexing:** A TF-IDF vector is computed for each chunk — no external vector database required.
3. **Retrieval:** Incoming questions are vectorized using the same TF-IDF scheme; cosine similarity selects the top 3 most relevant chunks.
4. **Generation:** Retrieved chunks are assembled into a grounded system prompt and sent to Claude (`claude-sonnet-4-20250514`).
5. **Response:** Claude's answer is returned alongside source citations (chunk ID, type, title, relevance score).

---

## Production: SAP BTP with AI Core + XSUAA

In production on SAP BTP, replace the direct Anthropic API call with SAP AI Core:

```
Development:  ANTHROPIC_API_KEY  →  Anthropic API
Production:   AICORE_SERVICE_KEY →  SAP AI Core (Generative AI Hub)
                                     same Claude Sonnet model
                                     + XSUAA token exchange
                                     + BTP audit logging
                                     + SAP AI ethics guardrails
```

The RAG service architecture is identical — only the LLM client changes. SAP AI Core exposes a compatible `/chat/completions` endpoint, and the `@sap-ai-sdk/ai-api` npm package provides a drop-in SDK replacement.

**BTP deployment targets:**
- Cloud Foundry runtime (Node.js buildpack)
- CAP OData service on HANA Cloud (replace CSV with HDI containers)
- SAP SuccessFactors Learning extension via BTP Extension Suite

---

## Connection to SAP Hire-to-Retire Value Chain

SkillPilot targets the **Learn** step of the SAP Hire-to-Retire process:

```
Hire → Onboard → [LEARN] → Perform → Reward → Offboard
                    ▲
              SkillPilot lives here
              inside SuccessFactors Learning
```

It addresses the gap where employees know they need upskilling for an S/4HANA or BTP transformation but don't know which courses, learning paths, or certifications to pursue.
