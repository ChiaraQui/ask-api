# Week 1 — `/ask` Demo (5 stages)

Build a typed LLM endpoint step by step. Each stage is a standalone FastAPI app you can run and compare.

## Setup

```bash
cp .env.example .env          # OPENAI_API_KEY=sk-...
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Demo stages

| Stage | File | What you learn |
|-------|------|----------------|
| 1 | `serve_stage1.py` | Bare `/ask` — string answer + `tokens_used` |
| 2 | `serve_stage2.py` | Structured output via Pydantic + `completions.parse` |
| 3 | `serve_stage3.py` | Validation guardrail + retry (`force_bad` demo knob) |
| 4 | `serve_stage4.py` | Per-request `model` override + `latency_ms` |
| 5 | `serve_stage5.py` / `main.py` | Full system + `cost_usd` readout |

Run one stage at a time (only one server on port 8000):

```bash
uvicorn serve_stage1:app --host 127.0.0.1 --port 8000 --reload
# or the full system:
uvicorn main:app --host 127.0.0.1 --port 8000 --reload
```

## Streamlit demo runner

Interactive UI for all five stages:

```bash
streamlit run demo_page.py
```

Open http://localhost:8501. Set **API base URL** to `http://127.0.0.1:8000` and start the matching stage server in another terminal.

## Test with curl

```bash
curl -s -X POST http://127.0.0.1:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is RAG in one sentence?"}'
```

Stage 5 example (model + cost):

```bash
curl -s -X POST http://127.0.0.1:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is chunking?", "model": "gpt-4o-mini"}'
```

## Smoke-test all stages

Requires `.venv` and a valid `OPENAI_API_KEY`:

```bash
python test_all_stages.py
```

## Project layout

```
week-1/
├── main.py              # Full system (stages 1–5 combined) + Session 2 RAG routes
├── rag.py               # Session 2: chunking, embeddings, Pinecone, grounding prompt
├── rag_ui.py            # Session 2: Streamlit UI for ingest + ask
├── ingest_northwind.py  # Session 2: ingest the Northwind sample docs via POST /ingest
├── eval_golden.py       # Session 2: golden-set eval (writes eval_results.md)
├── docs/northwind/      # Session 2: sample corpus
├── serve_stage1.py … serve_stage5.py
├── demo_page.py         # Streamlit test UI (Session 1 stages)
├── test_all_stages.py   # Automated stage smoke tests
├── requirements.txt
├── .env.example
└── .gitignore
```

## Session 2: RAG

`/ask` now answers from ingested documents. `POST /ingest` chunks text by section (~800 characters, 100 overlap), embeds it with `text-embedding-3-small` and stores it in Pinecone. `/ask` retrieves the top 8 chunks, answers only from them, cites the `document_id`s it used, and refuses with "I don't have enough information to answer that." when the answer isn't there. `GET /debug/retrieve?q=...` shows the top chunks and scores without calling the LLM.

Extra env vars: `PINECONE_API_KEY`, `PINECONE_INDEX` (see `.env.example`).

```bash
python ingest_northwind.py                # load the six Northwind docs
streamlit run rag_ui.py                   # UI for ingest + ask (ASK_API_URL sets the API)
python eval_golden.py                     # golden-set eval
```

### Golden-set eval

Six questions with known answers from the Northwind docs, plus two that the docs don't answer and should be refused. Run against the deployed service.

How each column is measured:

- **Retrieval hit (top-5):** a chunk from the expected document that contains the evidence sentence (e.g. "45 pence per mile") is in the top 5 results of `/debug/retrieve`. No LLM.
- **Faithful:** an LLM judge (`gpt-4o-mini`) reads the 8 passages `/ask` was given and the answer, and checks that every claim is supported by them. Checked by feeding it a made-up claim, which it marked ungrounded.
- **Correct:** the answer contains the expected facts and cites the expected `document_id`. Plain string checks, no LLM. For refusal questions: `/ask` refused and cited nothing.

| # | Question | Expected | Got | Cited | Retrieval hit (top-5) | Faithful | Correct | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | How many remote days are allowed? | Up to 3 days per week | Employees may work remotely up to three days per week. | POL-101 | ✅ | ✅ | ✅ | evidence at rank 2 |
| 2 | What is the mileage rate? | 45p per mile over 50 miles | The mileage rate is 45 pence per mile for travel over 50 miles. | POL-114 | ✅ | ✅ | ✅ | evidence at rank 1 |
| 3 | How quickly must a lost laptop be reported? | Within 1 hour | A lost laptop must be reported within one hour to the security desk. | POL-207 | ✅ | ✅ | ✅ | evidence at rank 1 |
| 4 | What is the WB-9 payload limit? | 25 kg | The WB-9 warehouse robot has a payload capacity limit of 25 kg. | SPEC-WB9 | ✅ | ✅ | ✅ | evidence at rank 1 |
| 5 | What are the company's values? | Safety first, Customer truth, Own the outcome, Teach what you learn | The company's values are: Safety first, Customer truth, Own the outcome, and Teach what you learn. | POL-101 | ✅ | ✅ | ✅ | evidence at rank 1 |
| 6 | How much can I claim for home office equipment? | Up to £350 every 36 months | Northwind reimburses up to £350 (or local equivalent) once every 36 months for approved home office equipment. | POL-101 | ✅ | ✅ | ✅ | evidence at rank 5 |
| 7 | What is the parental leave policy? | Refuse (not in docs) | I don't have enough information to answer that. | — | n/a | ✅ | ✅ | refused |
| 8 | Do employees get a company car? | Refuse (not in docs) | I don't have enough information to answer that. | — | n/a | ✅ | ✅ | refused |

Retrieval hit: 6/6 · Faithful: 8/8 · Correct (incl. refusals): 8/8

Notes:

- Row 6 is the weakest retrieval: the £350 rule ranks 5th, behind expense-policy chunks that share the word "claim". That's why `/ask` uses 8 chunks rather than 5.
- Row 5 needs all four values in one chunk. It failed with plain length-based chunking (the section heading and the last value landed in other chunks) and passes since chunking splits at section headings first.
- The judge is the same model that writes the answers, so treat "Faithful" as a sanity check rather than an independent grade.
