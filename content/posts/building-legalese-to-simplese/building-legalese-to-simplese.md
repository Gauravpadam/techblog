---
title: "Anyone needs an AI lawyer?"
date: 2025-11-14
draft: false
tags: ["ai", "llm", "architecture", "fastapi", "react", "elasticsearch", "ollama"]
description: "A walkthrough of building Legalese-to-Simplese, an AI system that translates legal documents into plain language. Covers the architecture, chunking strategies, and prompt engineering."
---

## Prelude

This post walks through building Legalese-to-Simplese, an AI application that takes legal documents and explains them in plain language. We'll cover the architecture, the challenges, and the solutions.

You can find the code on GitHub: [Legalese-to-Simplese](https://github.com/Gauravpadam/legalese-to-simplese)

## What We're Building

A system that can:
- Accept PDF uploads of legal documents
- Extract and chunk the text
- Analyze clauses and identify risks
- Answer questions about the document in natural language

The stack: FastAPI backend, React frontend, Ollama for LLM inference, Elasticsearch for semantic search.

## Part 1: The Architecture

The application has three layers:

1. **Frontend** - React UI for upload and interaction
2. **Backend** - FastAPI server handling requests and orchestration
3. **AI Layer** - Ollama for text generation, Elasticsearch for retrieval

Here's the data flow when a user uploads a document:

```
User uploads PDF
    ↓
FastAPI receives file
    ↓
Extract text (PyPDF2)
    ↓
Chunk into sections
    ↓
Generate embeddings (Ollama)
    ↓
Store in Elasticsearch
    ↓
Analyze with LLM
    ↓
Return structured analysis
```

For Q&A, the flow is different:

```
User asks question
    ↓
Convert question to embedding
    ↓
Search Elasticsearch for relevant chunks
    ↓
Retrieve top matches
    ↓
Build context from matches
    ↓
Send to LLM with prompt
    ↓
Return answer
```

This is called Retrieval-Augmented Generation (RAG). The LLM doesn't see the whole document. It sees only the chunks relevant to the question.

## Part 2: The Backend

### 2.1 Project Structure

The backend uses FastAPI with a modular structure:

```
backend/
├── routers/      # API endpoints
├── services/     # Business logic
├── clients/      # External service wrappers
└── utils/        # Helper functions
```

Each layer has a single responsibility. Routers handle HTTP. Services handle logic. Clients handle external APIs.

### 2.2 Document Upload Endpoint

```python
@router.post("/upload")
async def upload_document(document: UploadFile):
    # Extract text from PDF
    text = extract_text(document)
    
    # Chunk the text
    chunks = chunk_text(text, chunk_size=500, overlap=50)
    
    # Generate embeddings and store
    for chunk in chunks:
        embedding = ollama_client.embed(chunk)
        elasticsearch_client.index(chunk, embedding)
    
    # Analyze the document
    analysis = analyze_document(text)
    
    return analysis
```

FastAPI handles validation and serialization automatically. The endpoint receives a file, processes it, and returns structured JSON.

### 2.3 Q&A Endpoint

```python
@router.post("/ask")
async def ask_question(question: str, document_id: str):
    # Convert question to embedding
    query_embedding = ollama_client.embed(question)
    
    # Search for relevant chunks
    chunks = elasticsearch_client.search(query_embedding, top_k=5)
    
    # Build context
    context = "\n".join(chunks)
    
    # Generate answer
    answer = ollama_client.generate(
        prompt=f"Context: {context}\n\nQuestion: {question}\n\nAnswer:"
    )
    
    return {"answer": answer}
```

The key insight: we don't send the whole document to the LLM. We send only the relevant chunks. This solves the context window problem.

## Part 3: Text Extraction and Chunking

### 3.1 The PDF Problem

PDFs are difficult. They're designed for printing, not for text extraction. A PDF might contain:
- Actual text (easy to extract)
- Scanned images of text (requires OCR)
- Mixed content (both)

The solution is a multi-stage pipeline:

```python
def extract_text(document):
    # Try direct extraction first
    text = pypdf2_extract(document)
    
    if text and len(text) > 100:
        return text
    
    # Fall back to OCR
    text = ocr_extract(document)
    
    if text:
        return text
    
    raise ExtractionError("Could not extract text from document")
```

PyPDF2 handles most text-based PDFs. For scanned documents, we convert pages to images and run OCR.

### 3.2 Chunking Strategy

Legal documents don't fit in LLM context windows. A 50-page contract might be 100,000 tokens. Most LLMs handle 4,000-8,000 tokens.

We split the document into chunks:

```python
def chunk_text(text, chunk_size=500, overlap=50):
    chunks = []
    start = 0
    
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start = end - overlap  # Overlap preserves context
    
    return chunks
```

The overlap is important. Without it, a sentence split across two chunks loses meaning. With 50-token overlap, context is preserved at boundaries.

## Part 4: The AI Layer

### 4.1 Ollama Setup

Ollama runs LLMs locally. We use two models:

- **gpt-oss:cloud** - For text generation
- **nomic-embed-text** - For creating embeddings

The cloud variant runs on Ollama's servers. If you want fully local inference, swap to a local model. The API is identical.

```python
class OllamaClient:
    def __init__(self, base_url="http://localhost:11434"):
        self.base_url = base_url
    
    def generate(self, prompt, model="gpt-oss:cloud"):
        response = requests.post(
            f"{self.base_url}/api/generate",
            json={"model": model, "prompt": prompt}
        )
        return response.json()["response"]
    
    def embed(self, text, model="nomic-embed-text"):
        response = requests.post(
            f"{self.base_url}/api/embeddings",
            json={"model": model, "prompt": text}
        )
        return response.json()["embedding"]
```

### 4.2 Elasticsearch for Semantic Search

Keyword search doesn't work for legal Q&A. "Late payment" and "delayed rent" mean the same thing but share no words.

Elasticsearch 8.x supports vector search. We store document chunks with their embeddings:

```python
def index_chunk(chunk, embedding):
    elasticsearch.index(
        index="documents",
        body={
            "text": chunk,
            "embedding": embedding
        }
    )

def search(query_embedding, top_k=5):
    results = elasticsearch.search(
        index="documents",
        body={
            "knn": {
                "field": "embedding",
                "query_vector": query_embedding,
                "k": top_k
            }
        }
    )
    return [hit["_source"]["text"] for hit in results["hits"]["hits"]]
```

The search finds semantically similar chunks, not just keyword matches.

## Part 5: Prompt Engineering

### 5.1 The Problem

LLMs are creative. Too creative. Ask for a risk analysis and you might get a poem about contract law.

The solution is structured prompts with explicit output formats:

```python
ANALYSIS_PROMPT = """Analyze this legal document section.

Respond in this exact format:
- Clause Type: [type]
- Risk Level: [High/Medium/Low]
- Plain Language: [explanation in simple terms]
- Key Points: [bullet list]

Document section:
{text}

Analysis:"""
```

The format constraints force consistent output. The LLM follows the template.

### 5.2 Risk Assessment

For risk assessment, we define what each level means:

```python
RISK_PROMPT = """Assess the risk level of this clause.

Risk Levels:
- High: Unusual terms, significant financial exposure, or rights waiver
- Medium: Standard but notable terms that deserve attention
- Low: Typical boilerplate with no unusual provisions

Clause:
{clause}

Risk Level and Explanation:"""
```

Clear definitions produce consistent classifications.

## Part 6: The Frontend

### 6.1 React Structure

The frontend has three pages:

1. **Home** - Landing page with upload prompt
2. **Upload** - Drag-and-drop interface with progress
3. **Analysis** - Results display with tabs for summary, risks, and Q&A

State management uses React Context. No Redux, no external libraries. For this application size, Context is sufficient.

### 6.2 Progress Feedback

Document processing takes time. Users need feedback.

```jsx
function UploadProgress({ stage }) {
  const stages = [
    "Uploading document...",
    "Extracting text...",
    "Analyzing content...",
    "Generating insights..."
  ];
  
  return (
    <div>
      {stages.map((s, i) => (
        <div key={i} className={i <= stage ? "complete" : "pending"}>
          {s}
        </div>
      ))}
    </div>
  );
}
```

Each stage updates as processing progresses. Users see real progress, not a spinning loader.

## Challenges and Solutions

### Challenge 1: Context Window Limits

**Problem**: Legal documents exceed LLM context windows.

**Solution**: Chunk documents and use RAG. The LLM sees only relevant chunks, not the whole document.

### Challenge 2: Inconsistent LLM Output

**Problem**: LLMs produce varied output formats.

**Solution**: Structured prompts with explicit format requirements. Parse output with fallbacks for edge cases.

### Challenge 3: PDF Extraction Failures

**Problem**: Some PDFs don't extract cleanly.

**Solution**: Multi-stage pipeline with fallbacks. Direct extraction → OCR → manual input.

### Challenge 4: Semantic Search Quality

**Problem**: Keyword search misses semantically similar content.

**Solution**: Vector embeddings with Elasticsearch. "Late payment" matches "delayed rent" because their embeddings are similar.

## Results

The system processes a 20-page rental agreement in under 60 seconds. It correctly identifies:
- Unusual landlord entry clauses
- Hidden fee structures
- Non-standard termination terms

The risk assessment catches things humans miss on first read.

## What's Next

Short term:
- Document comparison (two contracts side-by-side)
- Export analysis as PDF
- Conversation history

Long term:
- Multi-language support
- Mobile application
- Batch processing

## Try It

The code is on GitHub: [Legalese-to-Simplese](https://github.com/Gauravpadam/legalese-to-simplese)

Setup:
1. Install Ollama and pull the models
2. Start Elasticsearch with Docker
3. Run the FastAPI backend
4. Start the React frontend

Instructions are in the README.

---

That's it. Upload a legal document, get a plain-language explanation. The architecture scales to other document types - medical records, financial statements, technical specifications. The pattern is the same: extract, chunk, embed, retrieve, generate.
