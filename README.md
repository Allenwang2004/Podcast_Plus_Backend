# Podcast+

<p align="center">
  <img src="file/coding101_pp.jpg" alt="Podcast+ overview" width="720">
</p>

> Turn any knowledge source into a personalized two-host podcast — generated on demand from your own documents, live web results, or a spoken question.

---

## Why Podcast+

Podcasts are a great way to learn, but the listener has never been in control:

- **Supply-side limits** — you can only listen to what creators decide to make.
- **Content mismatch** — topics are too broad, too shallow, or the cross-domain angle you care about simply doesn't exist.
- **Time cost** — getting one specific piece of knowledge often means sitting through forty minutes of small talk.

Podcast+ flips this around. Instead of *finding* content, you *make* it: give the system a topic and the sources you trust, and it writes a natural two-person conversation, reads it out loud, and hands you an episode tailored to exactly what you want to know right now.

---

## What You Can Do

### Personalized Listening
Pick the host voice style (gentle / lively / meditation) and the conversation depth (easy → professional). The same topic can be a casual explainer for a commute or a technical deep-dive for study.

### Real-Time Interaction
Speak or type your topic. Interrupt at any point with a follow-up and the next episode is regenerated around it. Podcast+ remembers the previous episode's context, so "tell me more about the pit-lane penalties" just works.

### Personal Knowledge Base
Upload PDFs, Word documents, slides, or even images (OCR is built in). Podcast+ indexes them into your own searchable knowledge base, and every episode is grounded in what *you* provided rather than generic web content.

### Live Web Search
No document on hand? Turn on web search and Podcast+ pulls fresh sources on the fly — ideal for news, current events, and fast-moving topics.

---

## How It Works

```
                     you speak or type a topic
                                │
                                ▼
                       ┌─────────────────┐
                       │   Whisper STT   │
                       └────────┬────────┘
                                ▼
                    ┌───────────────────────┐
                    │ instruction + memory  │
                    └───────────┬───────────┘
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ your knowledge │    │    web search    │    │ previous episode │
│ base (FAISS +  │    │    (Tavily +     │    │     context      │
│  re-ranking)   │    │   trafilatura)   │    │                  │
└───────┬────────┘    └────────┬─────────┘    └────────┬─────────┘
        └──────────────────────┼───────────────────────┘
                               ▼
              ┌────────────────────────────────────┐
              │  fine-tuned Llama dialogue model   │
              │      → two-host podcast script     │
              └────────────────┬───────────────────┘
                               ▼
              ┌────────────────────────────────────┐
              │      Kokoro TTS → merged .wav      │
              └────────────────┬───────────────────┘
                               ▼
                        play in the app
```

1. **Input** — Type a topic or record it; speech is transcribed with Whisper.
2. **Gather context** — The query is matched against your uploaded knowledge base (FAISS retrieval + cross-encoder re-ranking), optionally enriched with live web search, and merged with the previous episode when you're continuing a conversation.
3. **Write the script** — Our fine-tuned dialogue model turns the instruction and context into a natural back-and-forth between two hosts at the depth you chose.
4. **Voice it** — Each line is synthesized with Kokoro TTS in the selected voice, the segments are stitched together, and the finished episode streams back to the app.

Retrieval and audio synthesis run as always-on worker services with pre-loaded models, so episodes are generated in parallel without reloading anything between requests.

---

## The Dialogue Model

Generic instruction-tuned LLMs tend to *answer* a question rather than *talk about* it. To get the conversational feel of a real podcast, we fine-tuned our own dialogue model:

| | |
|---|---|
| **Base model** | `meta-llama/Llama-3.2-1B-Instruct` |
| **Method** | LoRA (r = 16, α = 32, dropout 0.05) on `q_proj / k_proj / v_proj / o_proj` |
| **Training data** | 3,000 conversations from DailyDialog, auto-labelled into five topics (sports, food, school, health, social) by sentence-embedding similarity and reformatted into `A:` / `B:` podcast-style instruction pairs |
| **Objective** | Causal-LM cross-entropy — each turn is predicted from the prompt plus every turn before it |
| **Setup** | 1 epoch, effective batch size 16, lr 2e-4, bf16, Google Colab T4 |
| **Weights** | [`coconut19/llama-dialog-lora`](https://huggingface.co/coconut19/llama-dialog-lora) on Hugging Face |

The fine-tuned model was compared with the base model on semantic consistency between turns and prompt–output alignment (both measured with sentence embeddings), and improved on both. Training notebooks, data, and evaluation scripts live in [`local_model/`](local_model/).

---

## Use Cases

1. **Knowledge-base Q&A** — Upload the FIA F1 Sporting Regulations and ask about the 2026 pit-stop rules. Two hosts walk through the fast-lane restrictions, safety-car procedure, and stop-and-go penalties, drawing only on your document.
2. **News briefing** — Turn on web search and ask for this week's world business news to get a fresh episode built from live sources.
3. **Conversational follow-ups** — Ask about a Taiwan-related topic by voice, then interrupt with a follow-up question; the next episode picks up where the last one left off.

---

## Tech Stack

- **Frontend** — Next.js + TypeScript, deployed on Vercel
- **Backend** — FastAPI, Python 3.11, `uv`
- **Speech** — Whisper (speech-to-text), Kokoro (text-to-speech)
- **Retrieval** — `all-MiniLM-L6-v2` embeddings, FAISS, `ms-marco-MiniLM-L-12-v2` cross-encoder re-ranker
- **Document parsing** — PyMuPDF / pdfplumber, python-docx, python-pptx, EasyOCR (English + Traditional Chinese)
- **Web search** — Tavily + trafilatura
- **Dialogue model** — Llama-3.2-1B-Instruct + LoRA (PEFT / transformers)
- **Deployment** — Docker Compose (API + audio worker + retrieval worker) on DigitalOcean

---

## Getting Started

```bash
git clone https://github.com/Allenwang2004/Podcast-.git
cd Podcast-
cp .env.example .env        # fill in the API keys listed in the file
docker compose up --build
```

The API is now available at `http://localhost:8001` (interactive docs at `/docs`). Point the frontend at this URL and generate your first episode.

To run without Docker:

```bash
uv sync
uv run python worker/retrieve_worker_service.py &
uv run python worker/audio_worker_service.py &
uv run uvicorn app.main:app --port 8001
```

---

## Next Steps

- [ ] **Agent debate mode** — let the two hosts take opposing viewpoints and argue a topic instead of agreeing.
- [ ] **Smarter knowledge base** — cluster uploaded documents by embedding and pick representative chunks per cluster to keep retrieval fast on large libraries.
- [ ] **Low-confidence filter** — skip retrieved passages whose relevance score falls below a threshold, so weak matches don't leak into the script.

---

## Team

**Team PP** — [@Allenwang2004](https://github.com/Allenwang2004), [@0u88](https://github.com/0u88)
