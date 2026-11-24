# Cynbit AI Lead Triage (AI/ML Intern Assessment)

This is a two-stage AI pipeline for Cynbit's **Inquiry Hub**. For every inbound inquiry it picks the team that should handle it, filters spam for free, pulls out budget, timeline and priority, and writes a first reply that a human approves before it is sent.

| Deliverable | File |
|---|---|
| AI Opportunity Report | [docs/01_AI_Opportunity_Report.md](docs/01_AI_Opportunity_Report.md) |
| Technical Approach Report | [docs/02_Technical_Approach.md](docs/02_Technical_Approach.md) |
| Proof of concept | [triage.py](triage.py) (pipeline and CLI), [app.py](app.py) (Streamlit UI), [data/inquiries.csv](data/inquiries.csv) |

## Run it

```bash
pip install -r requirements.txt
python triage.py check          # self-check plus 5-fold CV report (about 72% stage-1 accuracy)
streamlit run app.py            # demo UI
```

You can also run the pipeline on a single message from the command line:

```bash
python triage.py "We have 22 cafes and our Google reviews are a mess. Can Revio help?"
```

**API key is optional.** Without one, stage 1 (the local MiniLM classifier) still routes each inquiry, and the UI says that the Claude step was skipped. To get the full lead brief and draft reply, either `export ANTHROPIC_API_KEY=...` or paste a key into the app sidebar.

The first run downloads `all-MiniLM-L6-v2` (about 80 MB).

## How it works

1. **Stage 1, local classifier.** Sentence embeddings feed a logistic regression model that picks one of 6 categories: AI automation, custom software, digital marketing, product demo, internship/careers, or spam. It is free and works offline. Spam with confidence ≥ 0.5 is archived without any paid call.
2. **Stage 2, Claude structured output** (`messages.parse` with a Pydantic `LeadBrief`). It returns priority, summary, company, budget, timeline, matching services, the questions sales still needs to ask, and a draft reply. The prompt lists Cynbit's real catalogue, forbids invented prices or dates, and treats the inquiry as untrusted text.
3. **A human approves the reply.** Nothing is sent automatically.

The design rationale, evaluation plan, costs and rollout plan are in the Technical Approach report.
