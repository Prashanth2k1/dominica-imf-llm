# Dominica IMF report: LLM extraction, RAG and dashboard

Coursework for *Data Analysis with AI*, Assignment 1 (LLM coding case study), University of Hertfordshire.
Author: Prashanth Madduri (SRN 24155712).

The notebook `Dominica_IMF_LLM.ipynb` turns one IMF report into checked, machine-readable outputs and a dashboard:
IMF Country Report No. 26/117, *Dominica: 2026 Article IV Consultation, Press Release; Staff Report; and Statement by the Executive Director for Dominica* (June 2026, 98 pages).

## How to run

1. Open `Dominica_IMF_LLM.ipynb` in Google Colab.
2. Upload the `data/` folder (it holds `dominica_cr26117.pdf`) next to the notebook. Upload `cache/` as well if you want to rerun without any model calls.
3. Choose *Runtime > Run all*. The first cell installs `pypdf` and `faiss-cpu`; Colab already has the other packages.

All paths are relative to the notebook folder. No key is stored in the code.

## Backends

The notebook picks the LLM backend by itself:

| Backend | When | Notes |
|---|---|---|
| `gemini_api` | a Colab secret named `GEMINI_API_KEY` exists | Google Gen AI SDK; needed for `gemini-3.6-flash` |
| `colab` | no secret, but Colab's built-in model proxy is available | OpenAI-compatible proxy, `max_tokens=4096` |
| `replay` | neither | reads replies from `cache/llm.sqlite` only and stops if one is missing |

The recorded run used the `gemini_api` backend on 4 November 2026. The shipped notebook is a Restart and run all from the cache on 5 November 2026 (backend `colab`, since no key was needed): 124 calls, all cached, 0 live.

Every reply is stored in `cache/llm.sqlite` under a sha256 key of (model, system prompt, prompt, schema), so a rerun with the cache makes no live calls. Failed calls are retried with a growing wait. Every reply is validated with Pydantic and gets one repair round if it fails.

## Models

| Role | Model |
|---|---|
| Writer (summaries, extraction, labels, answers, sub-questions) | `google/gemini-3.6-flash` |
| Judge (summary scores, quote verdicts, indicator audit, RAG grades) | `google/gemini-3.8-flash` |
| Embeddings and token counting | `sentence-transformers/msmarco-distilbert-base-tas-b` |
| Vector store | FAISS `IndexFlatL2` on unit-length vectors |

## Pipeline

- **Task 1:** pypdf text, cleaning (running headers and footers, page numbers, contents leaders, chart ticks, hyphenation, split words, broken lines), 21 sections found from the report's own contents page, 300-token chunks with 50-token overlap, section summaries under 100 words (assert), executive summary, judge scores for consistency, completeness and factual alignment with one revision round.
- **Task 2:** seven themes counted by a lexicon and by LLM chunk labels (Spearman rho), 5-7 strengths and 5-7 vulnerabilities with quotes checked on their page by code and by the judge, the indicator JSON with null fallbacks and reasons, a judge audit (SUPPORTED / NOT_SUPPORTED / CORRECT_NULL / WRONG_NULL), a hand check against values read from the PDF, and time series parsed from Tables 1, 4, 5a and 5b with cross-checks.
- **Task 3:** retrieval over the chunks, nine test questions with gold facts and gold pages (one unanswerable), scored by code (fact recall, valid citations, gold page hits) and by the judge (faithfulness, completeness, relevance).
- **Extension:** multi-query RAG. The writer writes three sub-questions per question, each query retrieves candidates, MMR de-duplicates them, and the answers are compared with single-query RAG on the same nine questions.
- **Task 4:** `pd.melt` to long format and a six-panel dashboard with a key-findings header, saved at 300 dpi.

## Outputs

| File | Content |
|---|---|
| `outputs/json/indicators.json` | the indicator JSON in the target schema |
| `outputs/json/indicator_evidence.json`, `indicator_audit.json` | evidence, fallback reasons, judge labels, hand check |
| `outputs/json/section_summaries.json`, `executive_summary.json` | summaries and judge scores |
| `outputs/json/themes.json`, `strengths_vulnerabilities.json` | theme counts; checked strengths and vulnerabilities |
| `outputs/json/timeseries.json`, `timeseries_long.json` | series from the statistical tables, wide and long |
| `outputs/json/rag_eval.json`, `extension_multiquery.json` | RAG and extension results |
| `outputs/json/cleaning_report.json`, `sections.json`, `chunks.json` | text processing records |
| `outputs/json/call_log.json`, `run_record.json` | every model call and every prompt |
| `outputs/figures/dashboard.png`, `extension_multiquery.png` | figures (300 dpi) |

## Files

| Path | What it is |
|---|---|
| `Dominica_IMF_LLM.ipynb` | the notebook, as executed in Colab |
| `dominica_kit.py` | helper module written by the notebook itself (`%%writefile`) |
| `data/dominica_cr26117.pdf` | the IMF report |
| `cache/` | stored model replies and embedding vectors |
| `outputs/` | JSON files and figures |
| `requirements.txt` | pinned package versions |
| `LICENSE` | MIT |

## Source

International Monetary Fund (2026) *Dominica: 2026 Article IV Consultation, Press Release; Staff Report; and Statement by the Executive Director for Dominica*. IMF Country Report No. 26/117. Washington, DC: IMF.
