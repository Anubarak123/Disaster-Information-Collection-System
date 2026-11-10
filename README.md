# Event Insight — A Chained LLM Agent Pipeline for Disaster Situation Reports

Given a one-line description of a disaster ("2023 Turkey earthquake"), this pipeline
identifies the specific event, sources real news articles about it, collects
license-attributed photographs, and produces a structured situation brief — with
schema validation at every agent boundary.

This was completed as a Build Project with Open Avenues Foundation (8 weeks, Data
Science Fellow: Shubhashree Dash). Workshops 1–6 cover the pipeline itself; the merged
notebook in this repository combines them into one end-to-end run and adds the fixes
made while executing it against live APIs.

---

Project brief: build an AI-powered situation brief generator that combines real-time
news analysis with web-sourced event images, so that a reader can understand the scale
and impact of a disaster without reading the underlying coverage.

## Pipeline

```
user description
      │
      ▼
[1] Intent Clarification Agent   → structured query (event type / location / period / keywords)
      │                             multi-turn; re-prompts if output fails schema check
      ▼
[2] Event Identification Agent   → event record + up to 5 sourced articles
      │                             Gemini with Google Search grounding;
      │                             article URLs also recovered from grounding metadata
      ▼
[3] Image Collection             → up to 5 photographs with license and author metadata
      │                             Wikimedia Commons (MediaWiki API, file namespace)
      ▼
[4] Summarization Agent          → situation brief (headline / overview / severity /
      │                             affected population / response / key facts / sources)
      ▼
final formatted report
```

## Agent boundaries are validated, not assumed

LLM output is treated as untrusted input. Every agent's response is parsed and
validated against a Pydantic schema before it is allowed into the next stage:

| Schema | Fields | On failure |
|---|---|---|
| `IntentOutput` | event_type, location, time_period, keywords (non-empty) | returns `None`, conversation continues to collect better input |
| `EventOutput` | event_name, event_type, location, date, casualties, summary, severity_score (1–5), articles | raises — a malformed event record cannot be repaired downstream |
| `ImageItem` | title, url, thumbnail_url, description, license, author, local_path | bad entries are dropped, the rest of the list survives |
| `SituationBrief` | headline, overview, severity, affected_population, response, key_facts, sources_used (non-empty) | raises; a custom validator also rejects headlines over 20 words |

The deliberate choice here is asymmetric failure handling: recoverable stages
retry, unrecoverable stages fail loudly rather than passing corrupt JSON forward.

## Setup

The pipeline runs in Google Colab.

1. Open the notebook in Colab.
2. Add a secret named `GOOGLE_API_KEY` in Colab's Secrets panel (key icon in the
   left sidebar). The code reads it with `userdata.get("GOOGLE_API_KEY")` — no
   credential is stored in the file.
3. Get a key from Google AI Studio if you do not have one.
4. Run all cells top to bottom.

The Wikimedia API needs no key.

## Output

Running the notebook produces:

- the intent query and event record as validated JSON,
- up to five images downloaded to `disaster_event_images/` with license and author
  metadata retained per image,
- a situation brief, and
- a formatted text report combining all four stages.

## Problems hit while running it, and what fixed them

**1. `429 RESOURCE_EXHAUSTED` on the model call.** The quota error was model-specific
rather than key-wide: the requested model's quota limit was zero for that key. Switching
the model to `gemini-2.5-flash` (verified accessible for the key) resolved it without
changing the request structure.

**2. Wikimedia endpoint returned errors.** The newer
`api.wikimedia.org/core/v1/commons/search/files` endpoint did not behave the way the
original code assumed. The request was moved to the standard MediaWiki endpoint,
`commons.wikimedia.org/w/api.php`, searching the file namespace (6) directly — no
authentication, more stable for this use case.

**3. `UnicodeEncodeError` on the User-Agent header.** The header string contained a
non-ASCII en-dash (U+2011); HTTP headers require ASCII. Replacing it with a plain hyphen
fixed the failure. Worth noting because the traceback points at the request, not at the
string that caused it.

## Known limitations

- The notebook is meant to be run top to bottom; executing cells out of order raises
  undefined-variable errors. This is a notebook artifact, not a pipeline defect.
- Image relevance depends on Wikimedia's search; the pipeline filters out `.svg` and
  `.tiff` results, but cannot judge whether a photograph actually depicts the event.
- Grounding quality determines article quality. No cross-check between the article set
  and the summarizer's claims is performed yet.
- The `severity_score` (1–5) is produced by the model from reported casualties and
  damage; it is not a calibrated index.

## Next steps

- Add a claim-to-source check: verify every `key_facts` entry appears in at least one
  cited article before the brief is accepted.
- Replace prompt-level output control with `response_schema` so JSON conformance is
  enforced by the API rather than by string cleanup after the fact.
- Abstract the four agents behind a single runner so the pipeline can be invoked
  programmatically instead of through interactive cells.

## Repository contents

- `event_insight_chained_clean.py` — exported form of the merged Colab notebook,
  kept for reading; the notebook is the runnable artifact.
- `README.md` — this file.

## References

- Gemini API: text generation, structured output, system instructions — https://ai.google.dev/gemini-api/docs
- `google-genai` Python SDK — https://pypi.org/project/google-genai/
- MediaWiki API: `query`, `imageinfo` — https://www.mediawiki.org/wiki/API:Query · https://www.mediawiki.org/wiki/API:Imageinfo
- Pydantic — https://docs.pydantic.dev

