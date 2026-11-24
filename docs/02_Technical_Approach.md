# Technical Approach: AI Lead Triage for Cynbit

This document covers the model choices, data pipeline, prompt design, architecture, evaluation, and execution plan behind the proof of concept in this repo (`triage.py`, `app.py`).

---

## 1. Architecture

```
 Contact form / demo form / careers form
              │  (webhook, JSON)
              ▼
 ┌────────────────────────────┐
 │ Stage 1: local classifier  │  MiniLM-L6 embeddings → logistic regression
 │  category + confidence     │  free · ~20 ms CPU · offline
 └─────────────┬──────────────┘
      spam & conf ≥ 0.5 ──► archive (no paid call)
               │ otherwise
               ▼
 ┌────────────────────────────┐
 │ Stage 2: Claude            │  structured output (Pydantic schema)
 │  priority, summary, budget,│  category can override stage 1
 │  timeline, services,       │
 │  missing_info, draft_reply │
 └─────────────┬──────────────┘
               ▼
  CRM record + owner + SLA ──► human reviews / edits / sends reply
               │
               ▼
  corrections logged ──► weekly retrain of stage 1 + prompt eval
```

**Why two stages instead of "just call an LLM":**
- **Cost and abuse control:** public forms attract spam. The local model removes it for free, so the paid model only sees real leads.
- **It still works when the API doesn't:** if the API key is missing, rate-limited or down, routing still happens. The PoC shows this: without a key it returns the category, owner and SLA, plus an explicit `llm_error`.
- **Better results together:** the classifier is a fast first guess, and the LLM reads the full text and wins on disagreement. On ambiguous messages the LLM is more accurate. On spam the classifier is far cheaper.

## 2. Model choices

| Component | Choice | Why | Alternatives considered |
|---|---|---|---|
| Text representation | `sentence-transformers/all-MiniLM-L6-v2` (80 MB, 384-dim) | Generalises from about 12 examples per class. Runs on CPU. No data leaves the server. | **TF-IDF + LR**: tried first. **53%** 5-fold CV accuracy on the 72-example seed set, against **72%** for MiniLM. TF-IDF needs hundreds of labels per class. Multilingual MiniLM is next if Hinglish inquiries are common. |
| Classifier head | Logistic regression (`C=10`) | Calibrated probabilities (needed for the spam and confidence gates). Trains in milliseconds. Easy to explain. | SetFit fine-tuning is the upgrade path once there are 300+ labelled inquiries. |
| LLM | Claude (`claude-opus-5-5`) through the Anthropic Python SDK `messages.parse` | Native structured output: the response is validated against the `LeadBrief` Pydantic schema, so there is no regex JSON parsing. Strong instruction-following for "don't invent prices". | Cynbit already uses OpenAI. The pipeline only depends on "an LLM call that returns a validated schema", so swapping providers touches one function. After the eval set exists, test a smaller and cheaper model (Claude Haiku 5.5) and keep it only if quality holds. |

### Stage-1 results (honest numbers)

On the synthetic seed set (72 inquiries, 6 classes, 12 each), 5-fold stratified CV:

| Class | Precision | Recall |
|---|---|---|
| ai_automation | 0.62 | 0.67 |
| custom_software | 0.90 | 0.75 |
| digital_marketing | 0.57 | 0.67 |
| internship_careers | 0.83 | 0.83 |
| product_demo | 0.67 | 0.67 |
| spam_other | 0.82 | 0.75 |
| **Accuracy** | | **0.72** |

72% is not good enough to route on its own, and the design does not ask it to. Its jobs are (1) to filter spam before the paid call and (2) to give a fallback route when the LLM is unavailable. The confusions are the ones you would expect: "AI automation" against "digital marketing" both cover "CRM automation", and "AI automation" against "product demo" both mention AI features. The obvious fix is real data. Cynbit's inbox history, labelled once, should be several hundred examples.

## 3. Data pipeline

1. **Seed data (done):** `data/inquiries.csv`, 72 hand-written inquiries that mirror Cynbit's real services, products, internship and spam patterns. Written for the PoC, because no real inbox data is public.
2. **Bootstrap with real data (week 1 of pilot):** export the last 6 to 12 months of contact-form submissions. Remove phone numbers and emails with regex before anything leaves the database. Label with the LLM, and have a human spot-check about 20%. Target: 300 to 500 labelled rows.
3. **Online feedback loop:** every time a human changes the route or edits the draft, store `(text, predicted, corrected, draft, final_reply)` in Postgres. This table is the training set for stage 1 and the eval set for stage 2.
4. **Retrain:** a weekly job refits stage 1 (seconds) and runs the evaluation below. The new model is deployed only if accuracy does not drop.

**Privacy:** only the message body goes to the LLM, not contact details. Raw inquiries stay in Cynbit's own database. Use an API configuration that does not train on customer inputs.

## 4. Prompt engineering

The system prompt (in `triage.py`, `SYSTEM`) is built around four ideas:

1. **Ground the model in Cynbit's actual catalogue.** It lists the exact service and product names from cynbit.in, so `services` maps to things Cynbit really sells, and the draft reply names real offerings instead of generic ones.
2. **Use structured output instead of format instructions.** The `LeadBrief` Pydantic model defines every field, with `Literal` enums for `category` and `priority` and a `description` on each field. The SDK enforces the schema, so the prompt can spend its words on judgment ("hot = clear need + budget or timeline") rather than JSON formatting.
3. **Treat the inquiry as untrusted.** The inquiry is wrapped in `<inquiry>` tags and the prompt says to ignore instructions inside it. The model has no tools, so even a successful injection can only change text that a human reviews. The demo includes a prompt-injection sample.
4. **Put the guardrails where the business risk is.** "Never invent prices, discounts or delivery dates" is the one rule that protects Cynbit legally and commercially. The `missing_info` field turns that rule into something useful: instead of guessing, the model lists what sales must ask.

The classifier's guess is passed as a *hint*, and the model is told to trust the text over it. This gives the model context without anchoring it to a wrong label.

## 5. Evaluation plan

| What | How | Pass bar (pilot) |
|---|---|---|
| Routing accuracy | Final category compared with the human-corrected category on held-out real inquiries | ≥ 90% |
| Spam gate safety | Count real leads that were wrongly archived as spam | 0 per 200 |
| Field extraction | Exact or near match on budget and timeline against human labels | ≥ 85% |
| Draft quality | Edit distance between draft and sent reply, plus a weekly LLM-judge rubric (correct services? no invented prices? clear next step?) spot-checked by a person | Median edit < 30% |
| Latency | p95 end to end | < 10 s |
| Cost | Logged `usage` tokens per lead | < ₹2 per real lead |

`python triage.py check` is the regression guard. It asserts known routes, confirms that spam skips the LLM, and fails if stage-1 CV accuracy drops below 0.70.

## 6. Cost estimate

Per real lead: about 700 input tokens (system prompt plus inquiry) and about 400 output tokens. At Claude Opus 5.5 list pricing ($4 / $20 per million tokens), that is roughly **$0.011 (about ₹1)** before thinking tokens. At 50 real leads a day, the cost is about ₹1,500 a month. That is less than an hour of staff time. Spam costs nothing. If the eval shows a smaller model holds quality, cost drops by more than 10x.

## 7. Production plan

| Week | Deliverable |
|---|---|
| 1 | Export and anonymise inbox history, label 300+ rows, retrain stage 1, build the eval set |
| 2 | FastAPI `/triage` endpoint, webhook from the contact form, write results to Postgres and the CRM (HubSpot, Zoho, or their own) |
| 3 | **Shadow mode:** the pipeline runs on every inquiry, humans work as usual, and predictions are compared daily |
| 4 | **Assisted mode:** drafts and routes appear in the team's inbox or Slack for one-click approval. Review the metrics against the pass bars. |
| Later | Auto-acknowledge cold leads and internship FAQs (low risk). Multilingual encoder. Package as an "AI Inbox Triage" product for clients. |

**Deployment:** one container (FastAPI with the MiniLM model loaded once, about 300 MB RAM) on their existing AWS or Azure setup. The LLM call is the only external dependency, and it fails safe.

**Monitoring:** log category, confidence, LLM stop reason, tokens and latency for each inquiry. Alert if the LLM error rate goes above 5%, or if the share of each category shifts sharply (for example, a spam wave or a broken form).

## 8. What the PoC does and doesn't do

**Does:** run the two-stage pipeline end to end. Fall back gracefully with no key or a bad key (both tested). Skip the paid call for spam. Show a working Streamlit UI with sample inquiries, including a prompt-injection case. Include a runnable self-check and cross-validated metrics.

**Doesn't (yet):** connect to a real inbox or CRM. Train on real Cynbit data. Send any email. Stage 2 needs an API key to run, and the reviewer's own key works in the sidebar.
