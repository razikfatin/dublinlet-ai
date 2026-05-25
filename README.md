# 🏠 DublinLet AI

> An intelligent rental assistant for Dublin, Ireland - built on the OpenAI Responses API.

DublinLet AI is a domain-specific conversational agent that helps users find, evaluate, and book rental properties in Dublin. It goes beyond what a generic LLM can offer by combining **live external APIs**, **retrieval-augmented generation (RAG)**, and **real-world actions** (calendar booking, email) in a single chat interface.



---

## 🎯 The Problem

Dublin's rental market is one of the most difficult in Europe. Prospective tenants face:

- A severe shortage of listings that disappear within hours
- Rent prices that are among the highest in the EU
- A well-documented epidemic of rental scams (Threshold reported a 40% YoY increase in 2024)
- Complex affordability trade-offs against Dublin's high cost of living
- Confusing commute options across a city with mixed transport infrastructure

A generic ChatGPT session cannot help with any of this concretely - it has no access to live listings, cannot calculate a real commute time, and cannot book a viewing. DublinLet AI was built to fill that gap.

---

## 💡 The Idea

The assistant acts as a knowledgeable Dublin rental agent in the user's pocket. A user describes what they are looking for in natural language - budget, location preferences, workplace, lifestyle needs - and the assistant handles the rest:

1. Searches for real listings on Daft.ie
2. Calculates live commute times to the user's workplace via Google Maps
3. Finds nearby amenities (gyms, pharmacies, supermarkets) via Google Places
4. Ranks properties against the user's stated priorities
5. Checks any listing for fraud risk using market benchmarks and scam pattern knowledge
6. Runs an affordability analysis against Dublin's actual cost-of-living data
7. Answers questions about Dublin neighbourhoods from a curated knowledge base
8. Books a property viewing by creating a real Google Calendar event and sending a confirmation email

---

## 🛠️ Tools & Architecture

The assistant is built on the **OpenAI Responses API** with GPT-4o. Eight function-calling tools are defined, each backed by a real implementation:

| # | Tool | What it does | External API |
|---|------|-------------|-------------|
| 1 | `search_properties` | Finds rental listings | Daft.ie GraphQL + curated fallback |
| 2 | `get_commute_time` | Live travel time to workplace | Google Maps Distance Matrix API |
| 3 | `get_nearby_amenities` | Finds gyms, shops, pharmacies | Google Places API v1 |
| 4 | `rank_properties` | Scores and ranks listings | Internal weighted algorithm |
| 5 | `check_fraud_risk` | Detects scam listings | Internal rules + FAISS RAG |
| 6 | `calculate_affordability` | Monthly budget breakdown | Internal + Dublin cost-of-living data |
| 7 | `query_area_knowledge` | Neighbourhood insights | FAISS RAG (18-passage Dublin KB) |
| 8 | `book_viewing` | Creates calendar event + sends email | Google Calendar API + Gmail SMTP |

### Agent Loop

```
User message
     │
     ▼
client.responses.create(instructions, history, tools)
     │
     ├── function_call items? ──► dispatch to tool implementation
     │                                    │
     │        ◄───────────────────────────┘
     │   append tool result to history
     │
     └── no tool calls ──► return text to user
```

### RAG System

A FAISS vector index is built at startup from 18 short passages covering:
- **8 neighbourhood profiles** - Rathmines, Ballsbridge, Tallaght, Clontarf, Swords, Phibsborough, Dún Laoghaire, Inchicore
- **5 scam patterns** - wire transfer deposits, landlord-abroad stories, phantom listings, pressure tactics, ownership verification
- **3 financial/legal rules** - RPZ rent caps, rent-to-income guidelines, cost-of-living benchmarks
- **2 transport passages** - Dublin Bus/Luas/DART coverage, BusConnects cycling infrastructure

Embeddings are produced with `sentence-transformers/all-MiniLM-L6-v2` (384 dimensions). Retrieval uses `faiss.IndexFlatIP` (cosine similarity on normalised vectors), returning the top 3 passages above a 0.25 similarity threshold.

---

## 📁 Repository Structure

```
dublinlet-ai/
│
├── DublinLetAI.ipynb          # Main notebook - run this in Google Colab
├── README.md                  # This file
└── report/
    └── DublinLetAI_Report.pdf # Project report (submitted separately)
```

---

## 🚀 Getting Started

### Prerequisites

- A Google Colab account
- An OpenAI API key (paid account, GPT-4o access required)
- A Google Cloud project with the following APIs enabled:
  - Distance Matrix API
  - Geocoding API
  - Places API (New)
  - Google Calendar API
- A Gmail account with an App Password generated

### Step 1 - Open in Google Colab

Upload `DublinLetAI.ipynb` to [Google Colab](https://colab.research.google.com) or open it directly from this repository.

### Step 2 - Add API Keys to Colab Secrets

Click the 🔑 **Secrets** icon in the left sidebar and add the following:

| Secret Name | Where to get it |
|-------------|----------------|
| `OPENAI_API_KEY` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| `GOOGLE_MAPS_KEY` | Google Cloud Console → Credentials |
| `GOOGLE_PLACES_KEY` | Google Cloud Console → Credentials |
| `GMAIL_APP_PASSWORD` | myaccount.google.com → Security → App Passwords |

### Step 3 - Upload Google Calendar Credentials

- In Google Cloud Console, create an OAuth 2.0 Desktop client ID
- Download the JSON file, rename it to `credentials.json`
- Upload it to Colab via the Files panel (folder icon in left sidebar)

### Step 4 - Run All Cells in Order

- **Cell 1** - installs dependencies
- **Cell 2** - loads API keys from Secrets
- **Cell 3** - builds the FAISS RAG index
- **Cell 4** - defines the Calendar OAuth flow
- **Cell 5** - runs authentication (one-time, saves `token.pickle`)
- **Cell 6** - sets up Gmail helper
- **Cell 7** - defines all 8 tools
- **Cell 8** - implements all tool functions
- **Cell 9** - budget visualisation demo
- **Cell 10** - system prompt
- **Cell 11** - agent loop
- **Cell 12** - launches the chat interface

> **Note on Google Calendar:** Cell 5 prints an OAuth URL. Open it in your browser, grant access, and paste the authorisation code back into the Colab prompt. This only needs to be done once - the token is saved to `token.pickle`.

> **Note on Daft.ie:** The assistant queries Daft.ie's unofficial GraphQL endpoint. If this fails due to rate limiting, it automatically falls back to curated synthetic listings calibrated to current market data.

---

## 💬 Example Prompts to Try

```
"I'm moving to Dublin with my partner next month. Budget €2,200/month,
I work at Google on Barrow Street and travel by bus. We'd love somewhere
safe with a gym nearby."
```

```
"I found a 2-bed in Ballsbridge for €900/month. The landlord says he's
abroad and wants a wire transfer deposit before viewing. Should I be worried?"
```

```
"Our combined income is €5,500/month. Can you check if a €2,600/month
apartment in Rathmines is affordable and show me a breakdown?"
```

```
"What is Phibsborough like? Is it safe and what are transport links like?"
```

```
"Book me a viewing for the first property next Saturday at 11am.
My name is Alex."
```

---

## ⚙️ Technical Notes

- The notebook uses the **OpenAI Responses API** (`client.responses.create`), not the Chat Completions API. These are different - the Responses API is designed for agentic, tool-calling workflows.
- Conversation history is managed manually as a list passed on every call. The API is stateless.
- The Google Places tool uses the **v1 Places API** (`places.googleapis.com/v1/places:searchNearby`), not the legacy Places API. Make sure you enable "Places API (New)" in Google Cloud Console.
- Google Calendar runs in **Mock Mode** if `token.pickle` is not present - it prints exactly what it would send to the API without making a real call.

---

## 🔑 Why This Is Better Than ChatGPT

| Task | ChatGPT | DublinLet AI |
|------|---------|-------------|
| Find real current listings | ❌ Cannot access Daft.ie | ✅ Queries Daft.ie live |
| Calculate commute time | ❌ Can only estimate | ✅ Google Maps live result |
| Find nearby gyms with ratings | ❌ May hallucinate | ✅ Google Places real data |
| Detect fraud with price benchmark | ❌ No market data | ✅ Area-specific RTB benchmarks |
| Book a viewing | ❌ Cannot take action | ✅ Creates real Calendar event |
| Answer Dublin area questions | ⚠️ Training data may be outdated | ✅ Curated RAG knowledge base |

---

## 📚 Data Sources

- [RTB Rent Index Q3 2024](https://www.rtb.ie/research/rent-index)
- [Daft.ie Rental Report 2024](https://www.daft.ie/report)
- [Threshold Ireland - Annual Report 2024](https://www.threshold.ie)
- [NTA BusConnects Documentation](https://www.busconnects.ie)
- [CSO Consumer Price Index](https://www.cso.ie)
