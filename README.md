# Task Submission Local RAG Pipeline (Langflow + Ollama + Chroma)

This README documents the work completed, provides reproduction steps, and summarizes results, prompt iterations, and technical findings. 

## Table of Contents
- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Quick Start](#quick-start)
- [RAG Pipeline Architecture](#rag-pipeline-architecture)
- [Prompt Engineering & Iterations](#prompt-engineering--iterations)
- [Results, Analysis, and Observations](#results-analysis-and-observations)
- [Screenshots](#screenshots)
- [Mitigations & Alternatives](#mitigations--alternatives)
- [Conclusion](#conclusion)

---

## Overview

This repository demonstrates a fully local RAG pipeline:

- Source data: `products_dataset.csv` (CSV knowledge base)
- Embeddings: `nomic-embed-text:latest` (local embedding model)
- Vector store: Chroma DB
- LLM: `llama3.2:1b` via Ollama (local inference)
- Orchestration / visual flow: Langflow

The goal was to validate that retrieval (embeddings + Chroma) returns correct context and to measure how well a small local LM can reason over that context.

---

## Tech Stack
- Langflow — visual flow builder for connecting components
- Ollama — run local LLMs (llama3.2:1b)
- Chroma DB — local vector database for embeddings
- Docker — optional for containerized runs

---

## Quick Start

1. Start Ollama (example Docker):

   docker run -d -p 11434:11434 --name ollama ollama/ollama

2. Pull and run the LLM model inside Ollama if needed:

   docker exec -it ollama ollama pull llama3.2:1b |
   docker exec -it ollama ollama run llama3.2:1b
   
4. Pull the embedding model used for creating embeddings inside Ollama if needed:

   docker exec -it ollama ollama pull nomic-embed-text:latest

5. Start Langflow (optional Docker):

   docker run -d -p 7860:7860 langflowai/langflow:latest

6. Build embeddings and ingest into Chroma using the Langflow pipeline (see flow in the repo).


---

## RAG Pipeline Architecture

The pipeline in Langflow performs the following high-level steps:
1. File Loading — load `products_dataset.csv`.
2. Text Splitting — split content into chunks.
3. Embedding & Ingestion — generate embeddings (nomic-embed-text) and store them in Chroma.
4. Query Embedding & Retrieval — embed incoming query and retrieve top-k context from Chroma.
5. Prompt Formatting — combine retrieved context + user question into a prompt template.
6. Text Generation — call local LLM (llama3.2:1b via Ollama) to produce an answer.
7. Output — return the formatted response and sources.

### Final Pipeline Overview (visual)

- ![Pipeline Overview](images/final%20rag%20ollama.png)

---

## Prompt Engineering & Iterations

I iteratively refined prompts to reduce hallucinations and improve formatting and numeric reasoning.

Initial simple prompt (too weak):

{context}
---
Given the context above, answer the question as best as possible.

This simple prompt caused hallucinations and partial answers. Iterations included:

- Rule-based prompt with explicit field instructions (ProductID, Price, Quantity, Origin, etc.).
- Few-shot examples to teach formatting and numeric filtering.
- Short “two-line output” prompt to reduce cognitive load on the small model.

Despite these improvements, the llama3.2:1b model still had limitations on logical filtering and numeric reasoning.

---

## Results, Analysis, and Observations

After successfully setting up Langflow, Ollama, and the Chroma DB vector store, I implemented the RAG pipeline using `products_dataset.csv`.

- Embeddings were generated with `nomic-embed-text:latest` and stored in Chroma DB.
- Responses were produced locally by `llama3.2:1b` via Ollama.

### Example Behaviors Observed

- The retriever consistently returned relevant context (the correct product rows and ProductIDs).
- The local LLM struggled with:
  - Numeric comparisons (e.g., interpreting "under $1000").
  - Aggregation (e.g., summing quantities).
  - Consistency in filtering by field values (e.g., "Origin = Brazil").

Example failure cases (observed in the UI):

- Question: What is the quantity of coffee beans from Brazil?
  - Model answer: NOT_ENOUGH_INFO
  - Sources: ProductID104
  - → Ground truth: 200 units exists in the dataset but the model failed to extract it.

- Question: Which laptops are under $1000?
  - Model answer: NOT_ENOUGH_INFO
  - Sources: ProductID102, ProductID103, ProductID117
  - → Retriever returned correct context but the model failed to apply numeric condition.

### Technical Findings

- Retriever (nomic embeddings + Chroma) is functioning correctly.
- The main limitation is reasoning / numeric capability of the 1B model running locally.
- Increasing retriever top_k and simplifying prompt templates improved results somewhat but did not fully resolve logic-based failures.

---

## Screenshots

Below are the screenshots captured during testing. (Pipeline visual is placed earlier in the README.)

1) Which laptops are under $1000?

- ![Laptop Result](images/rag%20ollama%20pc.png)

2) Quantity of coffee beans from Brazil

- ![Coffee Beans Result](images/rag%20ollama%20beans.png)

3) iPhone price and origin

- ![iPhone Result](images/rag%20ollama%20iphone.png)

---

## Mitigations & Alternatives

- As a fallback, I tested the same RAG flow with Gemini 2.5 Pro (API). It produced precise and logically consistent answers, confirming the pipeline and embeddings were correct and that the issue stems from model reasoning capacity.

### Gemini (fallback) — visual verification

The Gemini runs were used to validate that the pipeline and prompts can produce correct answers when paired with a higher-capacity model. Screenshots from the Gemini tests:

- Pipeline overview with Gemini:
  - ![Gemini Pipeline](images/rag%20pipeline-Gemini%20.png)

- Example Gemini responses (showing accurate numeric filtering and extraction):
  - ![Gemini Response 1](images/gemini%20response%201.png)
  - ![Gemini Response 2](images/gemini%20response.png)


---

## Conclusion

The Langflow–Ollama local RAG system is functional: retrieval works and context is returned reliably. However, answer quality remains tied to the LLM's reasoning capacity. With prompt refinements, structure and reliability improved, but logical filtering and numeric reasoning remain challenging for llama3.2:1b in local settings.

