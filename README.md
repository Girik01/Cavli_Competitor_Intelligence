# Cellular IoT Competitor Intelligence (n8n)

An automated workflow that monitors cellular IoT competitors in public news and company websites, and produces a cited, quality-checked intelligence brief by email. Built in **n8n** with **OpenAI** and **Tavily**.

Default competitors: Quectel, Telit Cinterion, Fibocom, Semtech (Sierra Wireless) and Nordic Semiconductor. Any other company can be added through a web form, with no code changes.

## What it does

```
Trigger (weekly schedule · web form · manual)
  ① COLLECT   Google News + Bing News RSS, IoT trade press, Tavily search
               (news + each competitor's own website)
  ② CLEAN     date window · non-article pages removed · cross-source duplicates merged
               · per-competitor caps · cross-run memory (weekly schedule only)
  ③ AI        Triage agent: relevance, company, category, significance 1–5,
               verbatim evidence quote (verified by code)
               Full-article retrieval for significance 4–5
               Writer agent: report with a source citation for every claim
  ④ QC        7 automated checks → pass: email the report
                                   fail: email a reviewer with Approve / Decline
```

Only three steps use AI (two OpenAI calls and Tavily). Everything else is deterministic logic that checks and constrains the AI output.

## Report contents

Headline · executive summary · one section per competitor (significance score, category, what happened, why it matters, dated source links) · market signals · themes · recommended actions · watchlist · *Cavli in the news* · Appendix A (every item scored and why it was kept or dropped) · Appendix B (method, QC results, source health).

## Quality controls

| Control | What it prevents |
|---|---|
| Verbatim evidence quote, checked by code | AI inventing facts; unverifiable items are downgraded |
| Citation guard | References to sources that were never collected |
| Coverage guard | The writer silently dropping an important item |
| Re-attribution | Stories filed under the wrong company |
| Guardrails | Social posts, market-size adverts, old content and passing mentions being presented as major news |
| Cross-source and cross-competitor dedupe | The same event reported more than once |
| Fallbacks | If an AI step fails, rule-based scoring or a deterministic report is used, and the error is shown in the report |
| Human approval gate | Any report that fails a check is held until a person approves it |

## Automated vs human-controlled

| Automated | Human-controlled |
|---|---|
| Collecting, cleaning and de-duplicating news | Choosing competitors, sources and the significance threshold |
| Relevance and significance scoring | Approving any report that fails a quality check |
| Writing, checking and emailing the report | Spot-checking the top facts before external use |
| Remembering what was already reported (weekly runs) | Acting on the insights |

## Requirements

- n8n (Cloud or self-hosted, recent version)
- OpenAI API key
- Tavily API key (free tier is enough for a weekly run)
- SMTP account for sending email (e.g. Gmail with an App Password)

## Setup

1. **Import.** In n8n, create a new workflow, then **⋯ → Import from File** → `workflow/cavli-competitor-intelligence-workflow.json`. Import once.
2. **Credentials.** Select or create:
   - **OpenAI** credential in *AI Triage (LLM)* and *AI Report Writer (LLM)*
   - **Header Auth** credential for Tavily in *Tavily Search* and *Tavily Extract*: Name `Authorization`, Value `Bearer <tavily-key>`
   - **SMTP** credential in *Email Report* and *Human Review Gate (Email)*. For Gmail: host `smtp.gmail.com`, port `465`, SSL on, user = your Gmail address, password = a Gmail App Password.
3. **Config node.** Set `senderEmail` (must match the SMTP account), `reportRecipients` and `reviewerEmail`.
4. **Test.** Click *Execute workflow*. The report header should show `v1.15 (filter v1.15, scoring v1.15)`.
5. **Go live.** Toggle the workflow **Active**. The weekly run starts every Monday at 08:00 (Asia/Kolkata), and the form's Production URL becomes available.

API keys are stored only in n8n credentials. Exported workflow files never contain them.

## Ways to run

| Trigger | Competitors | Memory of past stories |
|---|---|---|
| Weekly schedule | Default list in Config | Yes: reports only what is new |
| Web form (*Run on Demand*) | Whatever the user enters | No: always a full report |
| Manual (editor) | Default list | No: always a full report |

Form input, one competitor per line: `Name | aliases (comma-separated) | official website`. Only the name is required. Example:

```
Quectel
Fibocom
Sequans | Sequans Monarch, Calliope 3 | sequans.com
```

## Configuration

All settings and the full AI prompts live in the **Config** node: competitors and aliases, lookback days, significance threshold, batch size, Tavily on/off, model names and email addresses. `filterMode: 'ai'` lets the AI judge relevance; `'rules'` adds keyword filters before the AI.

## Models and cost

| Step | Model | Why |
|---|---|---|
| Triage (about 60–80 items per run) | OpenAI GPT-6 Luna | Fast and low-cost for high-volume classification |
| Report writing (one call) | OpenAI GPT-6.1 Sol | Stronger reasoning and writing |

Typical cost is a few US cents per report in OpenAI usage, plus about 15 Tavily credits.

## Assumptions and limitations

- Uses only publicly available sources. Coverage depends on what news search engines and Tavily return, which varies between runs.
- Most items are judged from headline and snippet; the full article is read only for significance 4–5.
- Mostly English-language sources; Chinese-language coverage of Quectel and Fibocom is limited.
- Cross-run memory is stored in n8n workflow static data and is reset if the workflow is re-imported.
- Figures quoted in reports come from the sources and should be spot-checked before external use.

## Roadmap

More sources (Chinese-language news, certification databases, job postings) · Slack alerts for significance-5 items · thumbs-up/down feedback to tune scoring · history and trends in Postgres · an investigator agent that researches the most important stories further.

## Repository contents

```
workflow/cavli-competitor-intelligence-workflow.json   n8n workflow export (v1.15, no credentials)
docs/                                                   sample report and supporting documents
```
