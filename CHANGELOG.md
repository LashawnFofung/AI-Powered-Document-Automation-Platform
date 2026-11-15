# Changelog: Live POC (Hugging Face Space)

The live proof of concept runs on [Hugging Face Spaces](https://huggingface.co/spaces/lfofung/AI-Powered-Document-Intelligence-Automation-Platform). Its source and full commit history live there; this file records what changed and why.

## 2026-11-15: Model retirement fix and routing repair

### What broke
The live demo stopped answering questions. It called `gemini-2.0-flash-exp`, and Google shut down the Gemini 2.0 models on the Gemini API on June 1, 2026. The `google-generativeai` SDK it used also reached end of support on November 30, 2025.

### What testing found
After restoring the model, I tested the feature the project is built around: routing a question to the right document. On a 57-page mortgage file, I asked for the gross income on the most recent paystub. The demo answered "no mention of income" and cited **pages 1, 2, 53 and 54**. The paystubs were on **pages 37 to 40**.

Two causes:
1. **Routing was never applied.** `ask()` predicted a document type, then searched the whole index anyway.
2. **The labels didn't match.** The splitter and the router used different type names ("Paystub" vs "Pay Slip"), so a routed type could never match a stored chunk.

### Fixes
| Commit | Change |
|---|---|
| [`18c26a0`](https://huggingface.co/spaces/lfofung/AI-Powered-Document-Intelligence-Automation-Platform/commit/18c26a0) | Migrated to `google-genai` and `gemini-3.6-flash` through a small adapter, so existing calls work unchanged. The model ID now comes from a `GEMINI_MODEL` Space variable. Fixed the Space title and added portfolio links. |
| [`e998564`](https://huggingface.co/spaces/lfofung/AI-Powered-Document-Intelligence-Automation-Platform/commit/e998564) | Pinned every dependency to the versions from a working build. |
| [`0b9e9a1`](https://huggingface.co/spaces/lfofung/AI-Powered-Document-Intelligence-Automation-Platform/commit/0b9e9a1) | One shared `DOC_TYPES` list and `normalize_doc_type()` for both the splitter and the router. Retrieval is limited to the routed type when confidence is 0.6 or higher, with a fallback to the full index. Sources now show the actual document types used. |

### Result (same file, same question)
| | Before | After |
|---|---|---|
| Answer | "No mention of gross monthly income" | Total earnings of 8,800 (pay date 2025-07-17) |
| Source type | "Pay Slip" (a label only, not applied) | Pay Slip |
| Pages cited | 1, 2, 53, 54 | 37, 38, 39, 40 |

### Lessons
- **Keep model IDs in config and watch provider deprecation pages.** A demo that worked in the spring can fail by the fall without a single code change.
- **Test the feature you advertise, end to end.** A routing step that is not wired into retrieval looks fine until a real question reaches it.
- **Refusing to guess is a safeguard.** With the wrong context, the model said "no mention" instead of inventing a number, which kept a retrieval bug from becoming a wrong answer.
