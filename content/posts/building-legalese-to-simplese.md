---
title: "Building Legalese-to-Simplese: When Your Rental Agreement Needs a Translator"
date: 2025-11-14
draft: false
tags: ["ai", "llm", "fastapi", "react", "elasticsearch", "ollama", "full-stack"]
description: "How I built an AI-powered legal document analyzer that turns 'heretofore' into 'from now on' - because nobody should need a law degree to understand their lease."
---

## The Problem: Legal Documents Are Basically Encrypted Messages

Let's be honest - the last time you signed a rental agreement, employment contract, or any legal document, did you actually read it? And if you did, did you understand what "the party of the first part hereby indemnifies" actually means?

I didn't. And I'm a software engineer who reads technical documentation for fun.

So I did what any reasonable developer would do: I built an AI system to read these documents for me. Because if I'm going to blindly agree to something, I'd at least like to know what I'm agreeing to.

## What I Built

**Legalese-to-Simplese** is a full-stack application that takes your legal documents and translates them into actual human language. Upload a PDF, get instant analysis, risk assessment, and the ability to ask questions like "Can my landlord just show up whenever they want?" without paying $300/hour for a lawyer.


### Core Features

- **Document Analysis**: Upload PDFs, DOCs, or paste text - get instant breakdown
- **Risk Assessment**: High/medium/low risk clauses with actual explanations
- **Plain Language Translation**: "Heretofore" → "From now on"
- **Interactive Q&A**: Ask questions about your contract in natural language
- **Semantic Search**: Find relevant sections using Elasticsearch vector search

Think of it as having a lawyer friend who's always available, never judges you for not reading the fine print, and works for free.

## The Tech Stack: Why I Chose What I Chose

### Backend: FastAPI (Because Life's Too Short for Flask Boilerplate)

I went with **FastAPI** for the backend, and honestly, it's been a joy. Automatic API documentation, type hints everywhere, async support out of the box - it's like Python finally grew up and got its act together.


```python
# This is all you need for a working endpoint
@router.post("/upload")
async def upload_document(document: UploadFile):
    # FastAPI handles validation, serialization, docs
    return await process_document(document)
```

The modular structure keeps things clean:
- `routers/` - API endpoints (upload, Q&A, health checks)
- `services/` - Business logic (document processing, LLM interactions)
- `clients/` - External service wrappers (Ollama, Elasticsearch)
- `utils/` - Helper functions (PDF extraction, text chunking)

### Frontend: React + Vite (Because Create-React-App is So 2020)

For the frontend, I used **React 18** with **Vite** as the build tool. Vite's hot module replacement is so fast, I sometimes forget I'm in development mode. The page updates before I can even switch tabs.


The UI has three main pages:
1. **Home** - "Hey, upload your scary legal document here"
2. **Upload** - Drag-and-drop with real-time processing status
3. **Analysis** - Tabbed interface showing summary, risks, terms, and Q&A

I used React Context for state management because sometimes the simplest solution is the best solution. No Redux, no MobX, no state management library with a learning curve steeper than Everest.

### The AI Layer: Ollama (Choose Your Own Adventure)

Here's where it gets interesting. I used **Ollama** as the LLM runtime, which gives you flexibility most platforms don't:

Two models power the system:
- **gpt-oss:cloud** - For text generation and analysis (cloud variant)
- **nomic-embed-text** - For creating semantic embeddings

**The Cloud Variant Approach**: I'm using `gpt-oss:cloud`, which is Ollama's cloud-hosted option. It's the middle ground between "send everything to OpenAI" and "buy a gaming PC for your side project."

**Why Ollama's cloud variant?**
1. **No hardware requirements** - Runs on a potato laptop
2. **Easy to swap** - Want to go fully local? Just change the model name
3. **Consistent API** - Same code works for local or cloud models

**Want to go fully local?** Just swap `gpt-oss:cloud` for `gpt-oss:20b` and you're running everything on your machine. Same code, zero changes. That's the beauty of Ollama's architecture.

### Search: Elasticsearch (Because grep Doesn't Cut It Anymore)

For semantic search and document retrieval, I integrated **Elasticsearch 8.x** with vector search capabilities. When you ask "What happens if I pay rent late?", the system:

1. Converts your question into a vector embedding
2. Searches the document chunks for semantically similar content
3. Retrieves the most relevant sections
4. Feeds them to the LLM for answer generation


This is way better than keyword matching. "Late payment" and "delayed rent" are semantically similar, even though they share no words. Elasticsearch gets it.

## The Architecture: How It All Fits Together

Here's the data flow when you upload a document:

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
Analyze with LLM (Ollama)
    ↓
Return structured analysis
    ↓
Display in React UI
```

For Q&A, it's even cooler:


```
User asks question
    ↓
Convert to embedding
    ↓
Search Elasticsearch for relevant chunks
    ↓
Retrieve top 3-5 matches
    ↓
Build context from matches
    ↓
Send to LLM with prompt
    ↓
Stream response back to user
```

The whole thing processes a 20-page rental agreement in under 60 seconds. Not bad for something running on my laptop.

## The Challenges: What Went Wrong (And How I Fixed It)

### Challenge 1: PDF Extraction is a Nightmare

PDFs are the worst file format ever invented. They're basically PostScript wrapped in XML wrapped in disappointment.


**The Problem**: PyPDF2 works great for text-based PDFs but fails miserably on scanned documents or image-based PDFs.

**The Solution**: Multi-stage extraction pipeline:
1. Try PyPDF2 first (fast, works 70% of the time)
2. If that fails, convert PDF pages to images
3. Use OCR (initially AWS Textract, now exploring local alternatives)
4. Fall back to manual text input if all else fails

Lesson learned: Always have a Plan B, C, and D when dealing with PDFs.

### Challenge 2: Context Window Limitations

LLMs have context windows. Legal documents don't care about your context windows.

**The Problem**: A 50-page employment contract doesn't fit in most LLM context windows, even the fancy ones.


**The Solution**: Chunking strategy with overlap:
- Split document into 500-token chunks
- Add 50-token overlap between chunks (context preservation)
- Process each chunk independently for initial analysis
- Use semantic search to retrieve only relevant chunks for Q&A

This way, the LLM never sees the whole document at once, but it always has enough context to give meaningful answers.

### Challenge 3: Prompt Engineering is an Art Form

Getting the LLM to output structured, consistent analysis took more iterations than I'd like to admit.

**The Problem**: LLMs are creative. Too creative. Sometimes they'd write poetry about your rental agreement instead of analyzing it.


**The Solution**: Structured prompts with examples and strict output format:

```python
prompt = f"""
Analyze this legal document and provide a structured response.

Document Type: [Identify: Rental Agreement, Employment Contract, NDA, etc.]
Main Purpose: [One sentence summary in plain language]
Key Highlights: [List 3-5 critical points]
Risk Assessment:
  - Overall Risk Score: [1-10]
  - High Risk Items: [List with explanations]
  - Medium Risk Items: [List with explanations]
  - Low Risk Items: [List with explanations]

Document Text:
{document_text}

Respond ONLY with the structured format above.
"""
```

The key is being specific about what you want and showing examples. LLMs are like junior developers - they need clear requirements.


### Challenge 4: Real-Time Feedback Without Blocking

Processing a document takes time. Users hate waiting without feedback.

**The Problem**: Initial implementation was synchronous - upload, wait 60 seconds staring at a blank screen, get results. Terrible UX.

**The Solution**: Multi-stage loading states with progress indicators:
1. "Uploading document..." (file transfer)
2. "Extracting text..." (PDF processing)
3. "Analyzing content..." (LLM processing)
4. "Generating insights..." (final formatting)

Each stage updates the UI, so users know something is happening. Added a progress bar that actually reflects real progress, not one of those fake ones that just loops forever.


## The Results: Does It Actually Work?

I tested it with real documents:
- **Rental agreements** - Correctly identified sketchy clauses about landlord entry rights
- **Employment contracts** - Flagged non-compete clauses that were borderline illegal
- **NDAs** - Explained what "confidential information" actually meant in context

The risk assessment is surprisingly accurate. It caught things I missed on my first read-through. Things like:

> "Tenant shall maintain the property in good condition, reasonable wear and tear excepted, and shall be responsible for all repairs regardless of cause."

Translation: "You're paying for everything, even if the roof collapses because it was built in 1952."

Risk Level: **High** ⚠️


## What I Learned

### 1. Ollama's Flexibility is Underrated

Ollama proved that you don't need to choose between "expensive cloud API" and "buy expensive hardware." The cloud variants give you the convenience of hosted models without the OpenAI pricing or data training concerns. And if you later want to go fully local, it's literally a one-line config change. That's the kind of flexibility that makes prototyping actually fun.

### 2. Elasticsearch is Overkill Until It Isn't

For small documents, you could get away with simple text search. But once you need semantic understanding and vector similarity, Elasticsearch becomes essential. The setup complexity pays off quickly.

### 3. User Experience Matters More Than Tech Stack

Nobody cares that you're using FastAPI or React or Elasticsearch. They care that they can upload a document and understand it in under a minute. Focus on the experience first, optimize the tech later.


### 4. Prompt Engineering is 50% of the Work

Getting the LLM to consistently output useful, structured information took more time than building the entire backend. Treat prompts like code - version them, test them, iterate on them.

### 5. Error Handling is Not Optional

PDFs fail to parse. Elasticsearch goes down. LLMs hallucinate. Users upload Word docs from 1997. Plan for failure at every step, or spend your weekends debugging production issues.

## What's Next

The project is functional but far from done. Here's what's on the roadmap:

**Short Term:**
- Document comparison (compare two contracts side-by-side)
- Export analysis as PDF reports
- Conversation history (save your Q&A sessions)


**Long Term:**
- Multi-language support (legal jargon is bad in every language)
- Mobile app (because you sign contracts on your phone now)
- Batch processing (analyze 50 documents at once)
- Custom risk thresholds (what's risky for you might not be for someone else)

## Try It Yourself

The project is open source on GitHub: [Legalese-to-Simplese](https://github.com/Gauravpadam/legalese-to-simplese)

Setup is straightforward:
1. Install Ollama and pull the models
2. Spin up Elasticsearch with Docker
3. Run the FastAPI backend
4. Start the React frontend
5. Upload your most confusing legal document


Full setup instructions are in the README. If you get stuck, open an issue - I actually read them.

## Final Thoughts

I'll revise this once I get some more time. My style needs three more cups of coffee so I'll be back. Till then, read this and be happy. Hope you have an amazing day!

