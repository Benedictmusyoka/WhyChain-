# WhyChain: the agent that explains why a supply chain prediction moved

Open Agent Hackathon 2026, Track 02 "The Agent That Can Explain Why".
Stack: **Zetaris** (data layer) + **Meterless** (Nightshift as the only AI provider, plus World Model, H-MEM and Markovian-style engines) + **NVIDIA Nemotron** (via Nightshift).

## The problem
Chokepoints (Hormuz, Suez, Bab al-Mandeb, the Black Sea), Rhine low water and port congestion are moving at once, and ETA, lead-time and landed-cost forecasts jump. A planner sees a new number with no reason. They distrust it, re-check by hand across ERP, TMS and carrier portals, then wait for approval. Sources: Transport Intelligence October 2026 risk monitor; BSI resilience study (only 27% can act the same day).

## What the agent does
Given a PO whose ETA forecast moved between two snapshots, WhyChain:
1. **Splits the change exactly** with SQL over the forecaster's additive model (not an LLM estimate), and recomputes every figure independently.
2. **Tests every candidate cause** against independent data (chokepoint transits, port waits, gauge readings, carrier on-time, insurance holds). A cause the data does not support is **rejected**, even if the model or a news item claims it.
3. **Refuses to guess.** If more than 25% of the change is uncorroborated it returns `needs_review`, not a story.
4. **Resists planted instructions.** Free-text feeds are treated as data; hostile instructions are quarantined and can never create an action.
5. **Narrates only verified facts.** A numeric faithfulness check rejects any figure in the LLM text that is not in the evidence.
6. **Remembers.** H-MEM recalls the earlier explanation and reports what changed since, in a hash-chained ledger.
7. **Gates actions** by confidence, cost and role.

## Architecture
```
Planner (Nemotron Ultra, names only) -> Executor (SQL via Zetaris) -> Evaluator (arithmetic, freshness, rumour rejection)
      ^                                         |                              |
      +---- World Model / H-MEM / bounded scratchpad <-----------------------+
Explainer (Nemotron Ultra) narrates verified facts only -> Gate (role, cost, confidence) -> Learn
```
| Sponsor | Where | What changes when you turn it off (dashboard checkboxes, `--ablate=`) |
|---|---|---|
| Zetaris | `src/zetaris.ts`, `src/sql.ts` | `zetaris`: no data, verdict "unknowable" |
| Meterless Nightshift | `src/llm.ts` | only AI provider; models called by exact name |
| NVIDIA Nemotron | via Nightshift lanes | Lightning 30B extracts; Ultra 550B plans and explains |
| Meterless World Model | `src/engines/worldModel.ts` | `world`: lane context removed |
| Meterless H-MEM | `src/engines/hmem.ts` | `hmem`: no recall, no "since last time" |
| Markovian (bounded carryover) | `src/engines/scratchpad.ts` | carryover stays under 500 characters |
| Evaluator | `src/agent.ts` | `evaluator`: rumours accepted unverified |

**Engine caveat.** `worldModel.ts`, `hmem.ts` and `scratchpad.ts` implement the same concepts as the official Meterless engines but are *stand-ins*. Before submitting, follow `docs/ENGINES.md` to vendor the official engines and run their conformance suites. Do not claim conformance until those runners pass.

## Run it
Prerequisites: Node 22.5+, Docker with Zetaris running (follow the hackathon Participant Setup Guide), Meterless Nightshift with NVIDIA in its Providers tab.
```
npm install
cp .env.example .env        # fill in Nightshift URL/token and Zetaris URL/org/key
npm run data                # writes data/whychain.db and data/silos/*.json
```
Register each `data/silos/<table>.json` in Zetaris as a REST table inside schema `whychain` (same pattern as the Zetaris getting-started guide), then:
```
npm start                   # dashboard on http://localhost:3040
npm run explain -- PO4471   # CLI with full trace
npm run explain -- PO6033 --ablate=evaluator
```
Offline (stand-ins, not sponsor tech): `npm test` and `npm run demo:offline`.

## Verify before demo (things I could not check without your Zetaris/Nightshift)
- Zetaris REST path and body in `src/zetaris.ts` match your version (`POST /sql-editor/sqls/run`).
- `SHOW TABLES IN whychain` / `DESCRIBE` output column names in `src/sql.ts` `catalog()`.
- Nightshift header names for receipts: the code keeps every response header containing "meterless".
- Model names in `.env` match what Nightshift lists.

## Evidence you must produce (this is what wins)
1. `npm run smoke` confirms Zetaris and both Nemotron lanes answer.
2. `npm run benchmark` runs all three cases plus ablations and writes `evidence/BENCHMARK.md` and `evidence/benchmark.json` (it labels itself NOT EVIDENCE when stand-ins are active).
3. Paste the real numbers below and commit `evidence/` and `logs/`:
| Metric | Value |
|---|---|
| Cases with exact split matching ground truth (`data/ground_truth.json`) | _fill in_ |
| Rumour/uncorroborated causes rejected | _fill in_ |
| Injection attempts quarantined | _fill in_ |
| Largest prompt (tokens) / total tokens per case | _fill in from Nightshift usage_ |
| Ablation deltas (no Zetaris / no evaluator / no H-MEM) | _fill in_ |

## Live PortWatch feed (IMF)
`npm run portwatch` pulls real satellite-AIS chokepoint transits from the public IMF PortWatch ArcGIS service (`Daily_Chokepoints_Data`), computes the latest 7-day mean against the 2025 daily average, writes `data/portwatch_snapshot.json`, and rebuilds the tables with those real values (`chokepoint_daily`, plus a `feed_status` row aged from the data date). The agent then corroborates "Suez unavailable" or "Hormuz closure" against real transit counts, and the trace names the source and as-of date. Suez, Hormuz, Bosporus (as a Black Sea proxy), Cape of Good Hope, Bab el-Mandeb, Panama and Malacca are mapped; see `NAME_MAP`.
- It was written without network access to the service. After the first run, read the printed table and `data/portwatch_snapshot.json`; if a chokepoint is missing, adjust `FIELDS` and `NAME_MAP` in `src/data/portwatch.ts`.
- Reported refresh cadence differs between sources (weekly vs daily), so the freshness limit is one week. Check `as of` in the trace.
- With live data, the demo cases corroborate against reality, not the invented numbers. A cause the real data does not support will be rejected, which is the intended behaviour; record those runs as your evidence. The offline tests keep using synthetic data.
- Required citation: "Sources: UN Global Platform; IMF PortWatch (portwatch.imf.org)". Put it on the deck and README.
- Zetaris angle: register `chokepoint_daily` as a Zetaris source so the live feed is queried through the data layer like every other table.

## Data
All data is synthetic and deterministic (`src/data/generate.ts`). No real companies or prices. The October 2026 chokepoint context comes from public reporting; transit and gauge values in the tables are invented for the demo.

## AI use and development
The first scaffold was written with Claude (Anthropic) in a chat session, then tested offline. Describe your own development honestly in `docs/SUBMISSION.md`: which editor or AI coding tools you used (Cursor is a sponsor tool and optional, not required, per the organizers' published details you should re-confirm in HackOS Official Rules), what you changed, and what you verified yourself. Follow the hackathon's AI Usage Policy: disclose AI-assisted coding, never put API keys in the repo, and be able to explain every part of the code.

## Licence
Apache-2.0. Official Meterless engines, once vendored, keep their own Apache-2.0 notices.
