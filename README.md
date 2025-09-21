# Voice Chat Agent using GraphRAG

> **Voice Chat Agent that orchestrates multiple InfraNodus GraphRAG "experts" via n8n and converts responses to voice with ElevenLabs.**

**Repository:** [https://github.com/umeshyadav7988/Voice-Chat-Agent-using-Graph-RAG.git](https://github.com/umeshyadav7988/Voice-Chat-Agent-using-Graph-RAG.git)

**Included:** `Voice Chat Agent using Graph RAG_.pdf` — this PDF contains the full n8n workflow(s) exported as JSON and example payloads.

---

## Table of Contents

* [Overview](#overview)
* [Why GraphRAG (InfraNodus)?](#why-graphrag-infranodus)
* [Architecture](#architecture)
* [Features](#features)
* [Requirements](#requirements)
* [Setup](#setup)

  * [1. InfraNodus](#1-infranodus)
  * [2. n8n](#2-n8n)
  * [3. ElevenLabs](#3-elevenlabs)
  * [4. OpenAI (or other LLM)](#4-openai-or-other-llm)
  * [5. Import workflow](#5-import-workflow)
* [How it works (step-by-step)](#how-it-works-step-by-step)
* [Best practices & tips](#best-practices--tips)
* [FAQ](#faq)
* [Troubleshooting](#troubleshooting)
* [Contributing](#contributing)
* [License](#license)

---

## Overview

This project demonstrates a voice-capable AI chatbot that consults multiple **knowledge graph experts** (InfraNodus GraphRAG) and uses an n8n AI Agent node to orchestrate which expert(s) to query. Final responses are converted to speech using ElevenLabs Conversational AI.

The main goal is to show how GraphRAG can replace or complement traditional vector-store RAG solutions for fast setup, better knowledge relations, and easier reuse across workflows.

---

## Why GraphRAG (InfraNodus)?

* **Quick to set up**: no complex vector ingestion, metadata, or embedding pipeline required.
* **Holistic view**: graphs capture relations between concepts, improving retrieval of contextual relations.
* **Easier updates**: update graphs in InfraNodus directly, no reindexing of vectors.
* **Reusability**: the same GraphRAG graphs can be used across multiple n8n workflows and tools.

---

## Architecture

1. **ElevenLabs Conversational AI (voice)** — captures user voice, turns it into text, sends it to n8n via a webhook (knowledge\_base tool).
2. **n8n** — hosts a workflow that includes: Webhook → Chat Memory → AI Agent node → InfraNodus HTTP nodes (one per expert) → response integration → return to webhook.
3. **InfraNodus GraphRAG experts** — each graph acts as an expert; queried via the InfraNodus API and returns rich RAG/GraphRAG responses.
4. **ElevenLabs** — polls the webhook, receives the text response, condenses it as needed and synthesizes voice.

Diagram (conceptual):

```
User voice → ElevenLabs → n8n Webhook → AI Agent → InfraNodus GraphRAG experts → integrated text answer → ElevenLabs TTS → spoken reply
```

---

## Features

* Multiple GraphRAG "experts" (each an InfraNodus graph)
* n8n AI Agent to orchestrate tool/expert selection
* Chat memory support in n8n (session-based)
* Response integration from multiple experts with citations / relevant statements
* Voice input & output via ElevenLabs conversational agent
* Workflow JSON included in `Voice Chat Agent using Graph RAG_.pdf` for easy import

---

## Requirements

* InfraNodus account with API access and one or more graphs created.
* n8n instance (cloud or self-hosted) with the AI Agent node available.
* ElevenLabs account with conversational AI agent configured and API key.
* An LLM API key (OpenAI or compatible) for the n8n AI Agent node.
* Basic knowledge of n8n nodes (Webhook, HTTP Request, Chat Memory, Function / Set nodes).

---

## Setup

> **Important:** keep your API keys secret. Use n8n credentials and environment variables to store them.

### 1. InfraNodus

1. Create an account at [https://infranodus.com](https://infranodus.com).
2. Create separate graphs for each expert (use the PDF / content import tools to add docs).
3. Get your API key from **Settings → API Access** and copy the Bearer token.
4. For each graph note its name (you will paste this into the InfraNodus HTTP node body in n8n).

### 2. n8n

1. Install/run n8n (cloud, desktop or self-hosted). Ensure the AI Agent node is available in your build.
2. Add credentials for:

   * OpenAI (or your LLM provider)
   * InfraNodus (as an HTTP credential: `Authorization: Bearer <INFRANODUS_KEY>`)
3. Create a Webhook node that receives POST requests from ElevenLabs (the knowledge\_base tool).
4. Add a Chat Memory node to keep session-aware context using `sessionID` from ElevenLabs.
5. Add the AI Agent node and configure its `tools` list. For each tool:

   * Name it (e.g., `expert_sales`, `expert_product_docs`)
   * Put the InfraNodus graph description (auto-generated from InfraNodus) into the tool description
   * Configure the tool to call the corresponding InfraNodus HTTP node
6. For each expert add an HTTP Request node that queries InfraNodus GraphRAG API endpoint and returns the rich response.
7. AI Agent selects tools, optionally reformulates the query, calls InfraNodus endpoints, and aggregates expert answers.
8. Send the final integrated answer back to the Webhook response body so ElevenLabs can poll it.

### 3. ElevenLabs

1. Sign up at [https://elevenlabs.io](https://elevenlabs.io) and create a Conversational AI agent (or use their API).
2. Configure your agent to POST user prompts to your n8n webhook (knowledge\_base tool) and to poll for responses.
3. Make sure session IDs are forwarded so n8n chat memory keeps context.
4. Configure voices, audio formatting and any condensing rules ElevenLabs provides.

### 4. OpenAI (or other LLM)

1. Add your LLM API key in n8n credentials.
2. Configure the AI Agent node to use this LLM as the reasoning/aggregation engine.

### 5. Import workflow

1. The `Voice Chat Agent using Graph RAG_.pdf` in this repo contains the exported n8n workflow JSON. Open the PDF and extract the JSON or find the exported `.json` file inside the repo if included.
2. In n8n: Use `Import` → paste the workflow JSON → adjust credentials and node URLs.
3. Test with sample prompts from ElevenLabs to verify round-trip voice → text → GraphRAG → text → voice.

---

## How it works (step-by-step)

1. User speaks to ElevenLabs voice interface.
2. ElevenLabs sends the text prompt + `sessionID` to the n8n webhook (knowledge\_base tool).
3. n8n Chat Memory node attaches session history to the prompt.
4. n8n AI Agent loads its `tools` (experts) descriptions — each contains the InfraNodus graph summary.
5. AI Agent decides which experts to call (and may reformulate the query for better retrieval).
6. Agent calls InfraNodus HTTP node(s) for the chosen graphs. Each graph returns relevant statements, context, and justification.
7. AI Agent integrates expert responses into a single, coherent reply and returns it to the webhook response.
8. ElevenLabs polls the webhook, receives the reply, condenses for conversational flow (if configured), and synthesizes audio.

---

## Best practices & tips

* Limit the number of simultaneous experts (2–7 recommended). If you have more graphs, instruct the AI Agent to choose up to N experts per query.
* Keep graph descriptions clear and focused — InfraNodus auto-summaries are helpful.
* Use Chat Memory to persist short-term conversational context, but rotate or expire long histories.
* Test the agent with ambiguous prompts to tune the tool-selection behavior.
* Use retry logic in the n8n workflow for network calls to InfraNodus and ElevenLabs.

---

## FAQ

**Q: How many experts should I create?**
A: Aim for 2–7 experts. More experts increase orchestration complexity; if needed, allow the AI Agent to select a subset per query.

**Q: Why use GraphRAG instead of vector stores?**
A: GraphRAG (InfraNodus) provides richer relation-aware retrieval and is easier to update than custom vector ingestion pipelines.

**Q: Can I use another LLM instead of OpenAI?**
A: Yes — n8n supports other LLMs. Configure the AI Agent node with the provider of your choice.

---

## Troubleshooting

* **No response to ElevenLabs:** confirm the webhook response is returning JSON with a `text` field or the structure ElevenLabs expects.
* **InfraNodus auth errors:** check the Bearer token and that the graph name is correct.
* **Agent picks the wrong expert:** improve tool descriptions or limit the number of tools the agent may call.

---

## Contributing

Contributions welcome. Please open an issue or PR in the repository and include: a description of the change, why it helps, and how to test it.

---

