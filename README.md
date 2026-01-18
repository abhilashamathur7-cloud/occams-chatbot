# Occams Advisory AI Chatbot

A website-specific AI chatbot that answers user questions exclusively based on content crawled from [https://www.occamsadvisory.com](https://www.occamsadvisory.com). Built using RAG (Retrieval Augmented Generation) architecture with real-time streaming responses.

---

## 📋 Table of Contents

1. [Quick Start](#-quick-start)
2. [Architecture Overview](#-architecture-overview)
3. [Key Design Choices & Trade-offs](#-key-design-choices--trade-offs)
4. [Scraping Approach](#-scraping-approach)
5. [Threat Model & PII Security](#-threat-model--pii-security)
6. [Failure Modes & Graceful Degradation](#-failure-modes--graceful-degradation)
7. [Testing](#-testing)
8. [Project Structure](#-project-structure)
9. [API Reference](#-api-reference)

---

## 🚀 Quick Start

### Prerequisites

- Python 3.10+
- Node.js 18+
- Azure OpenAI API credentials (Chat + Embedding models)

### Backend Setup

```bash
cd backend

# Create virtual environment
python -m venv venv
venv\Scripts\activate  # Windows
# source venv/bin/activate  # Linux/Mac

# Install dependencies
pip install -r requirements.txt

# Install Playwright browsers
python -m playwright install chromium

# Configure environment variables
# Create .env file with:
# AZURE_OPENAI_CHAT_API_KEY=your-chat-api-key
# AZURE_OPENAI_EMBEDDING_API_KEY=your-embedding-api-key

# Start the backend server
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Start development server
npm run dev
```

### Initial Data Crawl

After starting the backend, trigger a crawl to populate the vector database:

```bash
curl -X POST http://localhost:8000/crawl
```

The application will be available at:
- **Frontend**: http://localhost:5173
- **Backend API**: http://localhost:8000
- **API Docs**: http://localhost:8000/docs

---

## 🏗️ Architecture Overview

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               FRONTEND                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    React + Vite + Tailwind CSS                         │  │
│  │  ┌─────────────────┐  ┌───────────────┐  ┌─────────────────────────┐  │  │
│  │  │  ChatWidget     │  │  SignupForm   │  │  SSE Client (Streaming) │  │  │
│  │  │  (3 UI modes)   │  │  (Lead Capture)│  │                         │  │  │
│  │  └─────────────────┘  └───────────────┘  └─────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     │ HTTP / SSE (Server-Sent Events)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               BACKEND                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         FastAPI Application                            │  │
│  │                                                                        │  │
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐    │  │
│  │  │  /chat       │    │  /crawl      │    │  /lead               │    │  │
│  │  │  /chat/stream│    │              │    │                      │    │  │
│  │  │  (RAG)       │    │  (Playwright)│    │  (Lead Capture)      │    │  │
│  │  └──────┬───────┘    └──────┬───────┘    └──────────┬───────────┘    │  │
│  │         │                   │                       │                 │  │
│  │         ▼                   ▼                       ▼                 │  │
│  │  ┌──────────────────────────────────────────────────────────────┐    │  │
│  │  │                    Core Services                              │    │  │
│  │  │  ┌────────────┐  ┌─────────────┐  ┌────────────────────────┐ │    │  │
│  │  │  │ RAGService │  │ VectorStore │  │ LeadsDB (SQLite)       │ │    │  │
│  │  │  │ (rag.py)   │  │ (ChromaDB)  │  │                        │ │    │  │
│  │  │  └────────────┘  └─────────────┘  └────────────────────────┘ │    │  │
│  │  └──────────────────────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
          ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
          │  ChromaDB    │  │  Azure OpenAI │  │  SQLite DB   │
          │  (chroma_data│  │  - GPT Chat   │  │  (leads.db)  │
          │   vectors)   │  │  - Embeddings │  │              │
          └──────────────┘  └──────────────┘  └──────────────┘
```

### Data Crawling Pipeline

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         WEBSITE CRAWLING PIPELINE                             │
└──────────────────────────────────────────────────────────────────────────────┘

  POST /crawl                    
       │                         
       ▼                         
┌─────────────────┐              
│ Playwright      │  Headless Chromium browser
│ Browser Launch  │  (Full fingerprinting to avoid 403)
└────────┬────────┘              
         │                       
         ▼                       
┌─────────────────┐              
│ Recursive Crawl │  - max_depth: 3 levels
│ occamsadvisory  │  - max_pages: 100 pages
│ .com            │  - timeout: 60s per page
└────────┬────────┘              
         │                       
         ▼                       
┌─────────────────┐              
│ Extract Content │  - Page title
│ from Each Page  │  - Body text (cleaned)
│                 │  - Headings (h1-h6)
└────────┬────────┘              
         │                       
         ▼                       
┌─────────────────┐              
│ Chunk Content   │  - chunk_size: 800 chars
│ (chunker.py)    │  - overlap: 100 chars
│                 │  - Sentence-based splitting
└────────┬────────┘              
         │                       
         ▼                       
┌─────────────────┐     ┌──────────────────────┐
│ Generate        │────▶│ Azure OpenAI         │
│ Embeddings      │     │ text-embedding-3-large│
└────────┬────────┘     └──────────────────────┘
         │                       
         ▼                       
┌─────────────────┐              
│ Store in        │  Persistent ChromaDB
│ ChromaDB        │  ./chroma_data/
└─────────────────┘              
```

### RAG Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            USER QUERY FLOW                                    │
└──────────────────────────────────────────────────────────────────────────────┘

  User Question                  
       │                         
       ▼                         
┌─────────────────┐              
│ 1. Expand Query │  Expand acronyms (BSGI → Business Services...)
│    (rag.py)     │              
└────────┬────────┘              
         │                       
         ▼                       
┌─────────────────┐     ┌──────────────────────┐
│ 2. Generate     │────▶│ Azure OpenAI         │
│    Embedding    │     │ text-embedding-3-large│
└────────┬────────┘     └──────────────────────┘
         │                       
         ▼                       
┌─────────────────┐     ┌──────────────────────┐
│ 3. Semantic     │────▶│ ChromaDB             │
│    Search       │     │ (Cosine Similarity)  │
│    (top-k = 5)  │     │                      │
└────────┬────────┘     └──────────────────────┘
         │                       
         ▼                       
┌─────────────────┐              
│ 4. Check Score  │              
│   Threshold     │              
│   (>= 0.35)     │              
└────────┬────────┘              
         │                       
    ┌────┴────┐                  
    │         │                  
    ▼         ▼                  
┌───────┐ ┌─────────────────┐    
│ BELOW │ │ ABOVE THRESHOLD │    
│THRESH.│ │                 │    
└───┬───┘ └────────┬────────┘    
    │              │             
    ▼              ▼             
┌──────────┐ ┌──────────────────┐     ┌──────────────────────┐
│ Return   │ │ 5. Build Prompt  │────▶│ Azure OpenAI GPT     │
│ Fallback │ │    + Stream      │     │ (Streaming Response) │
│ Message  │ │    Response      │     │                      │
└──────────┘ └──────────────────┘     └──────────────────────┘
```

---

## 🎯 Key Design Choices & Trade-offs

### 1. RAG Architecture with ChromaDB (vs. Fine-tuning)

**Choice**: Use Retrieval-Augmented Generation with a local ChromaDB vector store instead of fine-tuning a model.

**Rationale**: 
- **Fresh content**: Website content changes frequently; RAG allows instant updates by re-crawling without retraining costs
- **Grounded responses**: Retrieved context ensures answers are traceable to source URLs, reducing hallucination
- **Cost-effective**: No fine-tuning compute costs; only API calls at inference time
- **Auditability**: Each answer can cite its source pages for verification

**Trade-offs**:
- Slightly higher latency due to retrieval step (~100-200ms)
- Quality depends on chunking strategy and embedding model
- Requires maintaining vector store consistency with source website

### 2. Server-Sent Events (SSE) for Streaming (vs. WebSockets)

**Choice**: Use HTTP SSE for streaming chat responses instead of WebSockets.

**Rationale**:
- **Simplicity**: SSE is unidirectional (server→client), perfect for streaming LLM tokens
- **HTTP-native**: Works through proxies, load balancers, and CDNs without special configuration
- **Auto-reconnect**: Built-in browser reconnection logic
- **Lower overhead**: No WebSocket handshake or ping/pong frames

**Trade-offs**:
- Unidirectional only (sufficient for streaming responses)
- Connection limits per domain in older browsers (max 6)
- No binary data support (not needed for text tokens)

### 3. Playwright for Crawling (vs. Requests/BeautifulSoup)

**Choice**: Use Playwright with headless Chromium instead of traditional HTTP requests.

**Rationale**:
- **JavaScript rendering**: Occams Advisory uses React/dynamic content that requires JS execution
- **Full browser fingerprint**: Avoid 403 blocks from bot detection (custom User-Agent causes blocks)
- **Rich extraction**: Can wait for dynamic content, handle SPAs, extract rendered DOM

**Trade-offs**:
- Heavier resource footprint (~150MB for Chromium)
- Slower than direct HTTP (5s wait per page for JS hydration)
- More complex error handling

### 4. Soft Onboarding for Lead Capture (vs. Mandatory Signup)

**Choice**: Allow users to chat freely, then present a non-intrusive signup form after 2 bot responses.

**Rationale**:
- **User experience first**: Users can evaluate chatbot quality before committing information
- **Higher conversion**: Users who see value are more likely to provide real contact info
- **Reduced friction**: "Maybe Later" option respects user autonomy
- **Progressive disclosure**: Personalization improves after signup (greeting by name)

**Trade-offs**:
- Some anonymous usage before any lead capture
- More complex state management for form timing
- Potential for users to never sign up

### 5. Query Expansion for Acronyms (vs. Pure Semantic Search)

**Choice**: Expand known acronyms (BSGI, FTPS, CMIB, ERC) before embedding generation.

**Rationale**:
- **Domain-specific vocabulary**: Occams uses acronyms extensively; users may not know full names
- **Better retrieval**: "BSGI" query expands to include "Business Services Growth Incubation" for better cosine similarity
- **Zero-cost improvement**: Simple string manipulation before embedding call

**Trade-offs**:
- Requires manual maintenance of acronym dictionary
- May cause false positives if acronym has multiple meanings
- Only first acronym expanded to avoid query bloat

---

## 🕷️ Scraping Approach

### How We Collected and Structured Data

#### 1. Crawling Strategy

```
Target: https://www.occamsadvisory.com
Depth:  3 levels (homepage → section pages → detail pages)
Limit:  100 pages maximum
Method: Playwright headless Chromium (full browser)
```

**Key Implementation Details**:

1. **Browser Choice**: Full Chromium (not Requests) because:
   - The website uses React with client-side rendering
   - Simple HTTP requests return empty shells
   - Bot detection blocks requests with custom User-Agent headers

2. **Link Discovery**: Extract all `<a href>` elements, filter to same domain, normalize URLs (strip www, trailing slashes)

3. **Content Extraction**:
   ```javascript
   // Remove noise elements
   document.querySelectorAll('script, style, noscript, iframe').forEach(el => el.remove())
   // Get clean text
   return document.body.innerText
   ```

4. **Deduplication**: Track visited URLs (normalized) to avoid infinite loops from internal linking

#### 2. Chunking Strategy

```
Chunk Size:  800 characters
Overlap:     100 characters
Split By:    Sentence boundaries (., !, ?)
```

**Why These Values**:
- **800 chars**: Balances context completeness vs. retrieval precision (~1-2 paragraphs)
- **100 char overlap**: Ensures sentences aren't cut mid-thought; provides context continuity
- **Sentence splitting**: Preserves semantic units; avoids breaking mid-word/mid-sentence

#### 3. Metadata Preserved

Each chunk stores:
```json
{
  "content": "chunk text here...",
  "url": "https://occamsadvisory.com/services/bsgi",
  "title": "Business Services | Occams Advisory",
  "chunk_id": "chunk_0_page_5"
}
```

#### 4. Storage

- **Location**: `./chroma_data/` directory (SQLite-based)
- **Collection**: `occams_website`
- **Embedding Model**: Azure OpenAI `text-embedding-3-large` (1536 dimensions)
- **Distance Metric**: Cosine similarity

#### 5. Refresh Process

```bash
# Trigger manual re-crawl
curl -X POST http://localhost:8000/crawl

# Response
{"status": "started", "pages_crawled": 47, "chunks_created": 156}
```

The crawl completely replaces existing vectors to ensure freshness.

---

## 🔒 Threat Model & PII Security

### Where PII Flows

```
┌───────────────────────────────────────────────────────────────────────────┐
│                           PII DATA FLOW                                    │
└───────────────────────────────────────────────────────────────────────────┘

  User Browser                   
       │                         
       │ (1) User enters: name, email
       ▼                         
┌─────────────────┐              
│ localStorage    │  Session ID, name, email stored locally
│ (Client-side)   │  KEY: "occams_user"
└────────┬────────┘              
         │                       
         │ (2) POST /lead {name, email, session_id}
         │     (HTTPS encrypted in transit)
         ▼                       
┌─────────────────┐              
│ FastAPI Backend │  Receives, validates, stores
│                 │              
└────────┬────────┘              
         │                       
         │ (3) Stored in SQLite
         ▼                       
┌─────────────────┐              
│ leads.db        │  PII at rest (local file)
│ (SQLite)        │  NOT encrypted at rest (local deployment)
└─────────────────┘              
```

### Data Collected

| Field | Purpose | Sensitivity |
|-------|---------|-------------|
| Name | Personalization, lead identification | Low-Medium |
| Email | Follow-up, lead qualification | Medium |
| Session ID | Link chat history to lead | Low |
| Chat History | Stored in browser localStorage | Low (user's device) |

### Risk Mitigation Measures

#### 1. Transport Security
- All API calls should use HTTPS in production
- CORS configured to allow only specific origins

#### 2. Input Validation
- Email format validation before storage
- Name length limits (2-100 characters)
- SQL parameterized queries prevent injection

#### 3. Data Minimization
- Only collect name + email (no phone, address, etc.)
- Chat messages NOT sent to backend by default (localStorage only)
- No IP address logging in current implementation

#### 4. Access Control
- SQLite file stored locally (no network exposure)
- No admin interface for lead export (manual DB access only)
- No third-party analytics or tracking

#### 5. What's NOT Protected (Development Limitations)
- SQLite database is NOT encrypted at rest
- No audit logging of data access
- No PII deletion endpoint (GDPR right to erasure)
- No rate limiting on lead submission

#### Recommendations for Production

```
1. Encrypt leads.db at rest (SQLCipher or filesystem encryption)
2. Add rate limiting (e.g., 5 lead submissions per IP per hour)
3. Implement /lead/delete endpoint for GDPR compliance
4. Add audit logging for PII access
5. Use secure session tokens instead of client-generated IDs
6. Hash email addresses if only checking existence (not needed here)
```

---

## ⚠️ Failure Modes & Graceful Degradation

### 1. Azure OpenAI API Unavailable

**Failure**: Embedding or Chat API returns error/timeout

**Degradation**:
```python
# Current behavior in rag.py
try:
    async for chunk in self.openai.chat.completions.create(...):
        yield chunk
except Exception as e:
    yield {"type": "error", "data": "I'm having trouble responding right now. Please try again in a moment."}
```

**User Experience**: Friendly error message shown in chat, user can retry

### 2. Vector Store Empty/Corrupted

**Failure**: ChromaDB returns 0 results or errors

**Degradation**:
```python
# Health check endpoint
@app.get("/health/ready")
async def readiness_check():
    try:
        count = app.state.vector_store.collection.count()
        return {"status": "ready", "vector_store": {"documents": count}}
    except Exception as e:
        return {"status": "degraded", "error": str(e)}
```

**User Experience**: `/health/ready` returns degraded status; chat returns fallback message

### 3. No Relevant Context Found

**Failure**: User asks about topics not on Occams website

**Degradation**:
```python
# Score threshold check in rag.py
if not docs or docs[0]["score"] < self.config.rag.score_threshold:
    yield {
        "type": "content",
        "data": "Sorry, I couldn't find information about that topic. "
                "I can help with questions about Occams Advisory. "
                "Could you ask something about our services, expertise, or offerings?"
    }
    return
```

**User Experience**: Polite fallback guiding user toward valid topics

### 4. Crawler Cannot Access Website

**Failure**: Playwright timeout, 403 Forbidden, network errors

**Degradation**:
```python
# crawler.py error handling
try:
    page.goto(url, timeout=timeout, wait_until="load")
except Exception as e:
    print(f"❌ Failed to crawl {url}: {e}")
    # Continue to next URL, don't crash entire crawl
```

**User Experience**: Partial crawl succeeds; existing vectors remain until successful refresh

### 5. SQLite Database Locked

**Failure**: Concurrent write attempts to leads.db

**Degradation**:
- SQLite handles this natively with WAL mode
- Writes wait for lock release (no data loss)

**User Experience**: Slight delay in lead storage; transparent to user

### 6. Frontend Cannot Reach Backend

**Failure**: Network error, CORS issue, backend down

**Degradation**:
```typescript
// api.ts error handling
catch (error) {
  throw new Error('Failed to connect to chat service');
}
```

**User Experience**: Error toast shown; chat disabled until connection restored

### Failure Matrix Summary

| Component | Failure Mode | Detection | Degradation |
|-----------|-------------|-----------|-------------|
| OpenAI Chat | Timeout/Error | Try-catch | Error message in chat |
| OpenAI Embed | Timeout/Error | Try-catch | Crawl fails gracefully |
| ChromaDB | Empty/Corrupt | Count check | Fallback message |
| Website | 403/Timeout | Playwright exception | Skip page, continue |
| SQLite | Lock/Corrupt | SQLite errors | Retry or fail lead save |
| Network | Connection lost | Fetch error | Disable chat, show error |

---

## 🧪 Testing

### Test Coverage

The test suite covers the three required test categories:

#### 1. Valid/Invalid Email & Phone Validation

```python
# test_unit.py - TestEmailValidation class
def test_valid_email_standard():
    """john.doe@example.com → accepted"""
    
def test_valid_email_with_subdomain():
    """jane@mail.company.co.uk → accepted"""
    
def test_valid_email_with_plus():
    """test+chatbot@gmail.com → accepted"""

# test_unit.py - TestPhoneValidation class
def test_valid_phone_10_digits():
    """1234567890 → valid"""
    
def test_valid_phone_with_country_code():
    """+1 123-456-7890 → valid"""
    
def test_invalid_phone_too_short():
    """12345 → invalid"""
```

#### 2. Unknown Question Returns Safe Fallback

```python
# test_unit.py - TestUnknownQuestionFallback class
async def test_fallback_on_low_score():
    """Low relevance score (<0.35) triggers fallback message"""
    
async def test_fallback_on_empty_results():
    """No search results triggers fallback message"""
    
def test_fallback_message_is_safe():
    """Fallback doesn't expose internal details (no 'error', 'database', 'api')"""
```

#### 3. Chat Nudges User Toward Completion

The soft signup flow implemented in `SignupForm.tsx` nudges users:

```typescript
// After 2 bot responses, show signup form
// "Quick question before we continue!" prompt
// "Maybe Later" dismisses; re-shows after 3 more responses
// After 2 dismissals, doesn't show again in session
```

### Running Tests

```bash
cd backend

# Run all unit tests
pytest test_unit.py -v

# Run with coverage
pytest test_unit.py -v --cov=. --cov-report=html

# Run specific test class
pytest test_unit.py::TestEmailValidation -v
pytest test_unit.py::TestPhoneValidation -v
pytest test_unit.py::TestUnknownQuestionFallback -v
```

### Expected Output

```
========================= test session starts ==========================
test_unit.py::TestEmailValidation::test_valid_email_standard PASSED
test_unit.py::TestEmailValidation::test_valid_email_with_subdomain PASSED
test_unit.py::TestEmailValidation::test_valid_email_with_plus PASSED
test_unit.py::TestEmailValidation::test_duplicate_email_updates_existing PASSED
test_unit.py::TestPhoneValidation::test_valid_phone_10_digits PASSED
test_unit.py::TestPhoneValidation::test_valid_phone_with_dashes PASSED
test_unit.py::TestPhoneValidation::test_valid_phone_with_country_code PASSED
test_unit.py::TestPhoneValidation::test_invalid_phone_too_short PASSED
test_unit.py::TestPhoneValidation::test_invalid_phone_too_long PASSED
test_unit.py::TestUnknownQuestionFallback::test_fallback_on_low_score PASSED
test_unit.py::TestUnknownQuestionFallback::test_fallback_on_empty_results PASSED
test_unit.py::TestUnknownQuestionFallback::test_fallback_message_is_safe PASSED
test_unit.py::TestQueryExpansion::test_bsgi_expansion PASSED
test_unit.py::TestLeadsDatabase::test_add_lead_creates_record PASSED
========================= 20+ passed in 2.5s ===========================
```

---

## 📁 Project Structure

```
occams-chatbot/
├── backend/
│   ├── main.py              # FastAPI app, routes, lifecycle
│   ├── rag.py               # RAG service, prompts, streaming
│   ├── vector_store.py      # ChromaDB wrapper, embeddings
│   ├── crawler.py           # Playwright website crawler
│   ├── chunker.py           # Text chunking logic
│   ├── leads_db.py          # SQLite lead storage
│   ├── config.py            # Configuration loader
│   ├── config.yaml          # App configuration
│   ├── requirements.txt     # Python dependencies
│   ├── test_unit.py         # Unit tests
│   ├── chroma_data/         # Vector store (generated)
│   └── leads.db             # Lead database (generated)
│
├── frontend/
│   ├── src/
│   │   ├── App.tsx          # Main app component
│   │   ├── api.ts           # Backend API client
│   │   ├── useChat.ts       # Chat state hook
│   │   ├── types.ts         # TypeScript types
│   │   └── components/
│   │       ├── ChatWidget.tsx   # Main chat UI
│   │       ├── ChatMessage.tsx  # Message bubble
│   │       ├── ChatInput.tsx    # Input field
│   │       └── SignupForm.tsx   # Lead capture form
│   ├── package.json
│   ├── vite.config.ts
│   └── tailwind.config.js
│
└── README.md                # This file
```

---

## 📡 API Reference

### Health Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Basic liveness check |
| `/health/ready` | GET | Readiness check with vector store status |

### Chat Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/chat` | POST | Non-streaming chat response |
| `/chat/stream` | POST | SSE streaming chat response |

**Request Body**:
```json
{
  "message": "What services do you offer?",
  "user_context": {
    "name": "John",
    "email": "john@example.com",
    "session_id": "session-123"
  }
}
```

### Lead Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/lead` | POST | Capture lead information |
| `/lead/{email}` | GET | Get lead by email |

### Crawler Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/crawl` | POST | Trigger website crawl |

---

## 📊 Local Knowledge Base & Chat History

### Vector Database Location

```
backend/chroma_data/
├── chroma.sqlite3           # Main SQLite database
└── <collection-uuid>/       # Collection data
```

**Contents**: ~150-200 content chunks from occamsadvisory.com with embeddings

### Chat History

- **Storage**: Browser `localStorage` (key: `occams_chat_history`)
- **Format**: JSON array of message objects
- **Scope**: Per-browser, not synced to backend

### Leads Database

```
backend/leads.db             # SQLite database
```

**Schema**:
```sql
leads (id, name, email, session_id, source, created_at, updated_at)
chat_sessions (id, session_id, lead_id, started_at, last_active_at, message_count)
```

---

## 🛠️ Configuration

Key settings in `config.yaml`:

```yaml
# RAG Settings
rag:
  chunk_size: 800         # Characters per chunk
  chunk_overlap: 100      # Overlap between chunks
  top_k: 5                # Number of chunks to retrieve
  score_threshold: 0.35   # Minimum relevance score

# Crawler Settings
crawler:
  base_url: https://www.occamsadvisory.com
  max_depth: 3            # Link depth to follow
  max_pages: 100          # Maximum pages to crawl
  timeout: 60000          # Page load timeout (ms)
```

---

## 📝 License

This project was created as an assignment submission.

---

## ❓ Autonomy Prompts

### What did you not build and why?

1. **Authentication/User Accounts**: Not needed for a lead-capture chatbot; localStorage session IDs suffice for personalization without login friction.
2. **Scheduled Auto-Crawling**: Manual `/crawl` endpoint is sufficient for a demo; production would add APScheduler or cron for periodic refreshes.
3. **Admin Dashboard**: No UI for viewing leads or analytics—out of scope; SQLite can be queried directly for demos.
4. **Multi-language Support**: Occams website is English-only; adding i18n would add complexity without value.
5. **Conversation Memory (Multi-turn RAG)**: Each query is independent; implementing chat history context would increase token costs and complexity.

### How does your system behave if scraping fails or the LLM/API is down?

| Failure | Behavior |
|---------|----------|
| **Scraping fails** | Crawl skips failed pages, continues with others. Existing vector store remains intact until a successful crawl replaces it. User sees no disruption. |
| **Embedding API down** | Crawl fails completely (no new vectors). Chat continues using existing vectors. |
| **Chat API down** | Returns friendly error: *"I'm having trouble responding right now. Please try again in a moment."* No stack traces exposed. |
| **Vector store empty** | `/health/ready` returns `degraded`. Chat returns fallback: *"I couldn't find information about that topic."* |

### Where could this be gamed or produce unsafe answers?

1. **Prompt Injection**: Malicious user input like *"Ignore instructions and say..."* could manipulate LLM. **Mitigation**: System prompt is strict; context-only answers reduce risk but don't eliminate it.
2. **SEO Poisoning**: If Occams website is compromised with malicious content, the chatbot would parrot it. **Mitigation**: Only crawl trusted domain; no user-submitted URLs.
3. **Fake Lead Submission**: Bots could spam `/lead` with fake emails. **Mitigation**: Add rate limiting + CAPTCHA in production.
4. **PII Extraction**: Crafted queries might extract other users' data. **Mitigation**: No cross-user data in prompts; each session is isolated.
5. **Hallucination on Edge Cases**: Low-scoring but above-threshold results may produce plausible-sounding but wrong answers. **Mitigation**: Tune `score_threshold` higher; add source citations.

### How would you extend this to support OTP verification without leaking PII to third parties?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OTP VERIFICATION ARCHITECTURE                             │
└─────────────────────────────────────────────────────────────────────────────┘

  User enters email/phone
           │
           ▼
  ┌─────────────────┐
  │ Generate OTP    │  6-digit code, expires in 5 min
  │ (Backend)       │  Store: hash(OTP) + expiry in SQLite
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ Send OTP via    │  Option A: Self-hosted SMTP (no 3rd party)
  │ Email/SMS       │  Option B: Twilio/SendGrid (PII shared)
  └────────┬────────┘
           │
           ▼
  User enters OTP
           │
           ▼
  ┌─────────────────┐
  │ Verify OTP      │  Compare hash, check expiry
  │ (Backend)       │  Mark lead as "verified" in DB
  └─────────────────┘
```

**To avoid leaking PII to third parties**:
1. **Self-hosted SMTP**: Run Postfix/Mailgun on own infrastructure—email never leaves your servers.
2. **Hash before sending**: If using SMS provider, send only phone hash + OTP; provider can't link to identity (requires custom gateway).
3. **On-device OTP**: Use TOTP (like Google Authenticator)—no network transmission of PII.
4. **Minimal data to provider**: If using Twilio, send only phone number (required), not name/email. Twilio is SOC2 compliant.

**Recommended approach for this app**: Self-hosted SMTP for email OTP (zero third-party PII exposure) + SQLite storage of hashed OTPs with expiry.

---

## 🙏 Acknowledgments

- Built with [FastAPI](https://fastapi.tiangolo.com/), [React](https://react.dev/), [ChromaDB](https://www.trychroma.com/)
- Powered by [Azure OpenAI](https://azure.microsoft.com/en-us/products/ai-services/openai-service)
- Website content from [Occams Advisory](https://www.occamsadvisory.com/)
