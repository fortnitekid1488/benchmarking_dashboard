# Agent Operating Manual

## Scope

This repository is a generic template for automating updates to a financial benchmark workbook. It should stay reusable for future company sets and sectors.

## Files

- `financial_benchmark_template.xlsx` is the clean source template. It should contain only the `Свод` sheet with structure, labels, formatting, and header metadata, not pre-filled benchmark values. Do not overwrite it unless the user explicitly asks.
- `config/company_source_registry.json` contains batches, companies, and source URLs. The generic baseline should stay empty until a concrete project fills it.
- `scripts/prepare_aistudio_sources.py` prepares source packages for quarterly and annual modes.
- `scripts/apply_aistudio_json.py` applies returned LLM JSON to a copied workbook under `outputs/`.
- `web/` and `scripts/dashboard_server.py` implement the local dashboard.
- `.Codex/decisions.md` records non-obvious technical decisions.
- `.Codex/progress.md` records current state and next steps.

## Commands

Create a local Windows environment:

```bat
install_windows_requirements.cmd
```

Create a local macOS environment:

```bash
./install_macos_requirements.command
```

Prepare source packages:

```bash
.venv/bin/python scripts/prepare_aistudio_sources.py --mode quarterly --download-mode full
.venv/bin/python scripts/prepare_aistudio_sources.py --mode annual --download-mode full
```

Apply returned JSON:

```bash
.venv/bin/python scripts/apply_aistudio_json.py --mode quarterly
.venv/bin/python scripts/apply_aistudio_json.py --mode annual
```

Run the local dashboard:

```bash
.venv/bin/python scripts/dashboard_server.py --open
```

Portable launchers after local setup:

```bat
update_from_github.cmd
start_dashboard.cmd
prepare_quarterly_sources.cmd
prepare_annual_sources.cmd
update_quarterly_excel_from_aistudio.cmd
update_annual_excel_from_aistudio.cmd
```

For user-facing demos, prefer `start_dashboard.command` on macOS and `start_dashboard.cmd` on Windows.

Generated outputs, local caches, internal Codex notes, and generated workbook files are ignored. The clean source template `financial_benchmark_template.xlsx` is project data and is expected to be committed by default, while filled/generated benchmark workbooks are not. Do not stage `outputs/`, `.venv/`, `.playwright-*`, `.Codex/`, or other spreadsheet files unless the user explicitly clears that exact artifact for publication.

## Verification

For any workbook automation change:

1. Read the exact workbook structure before changing scripts.
2. Generate a copied workbook under `outputs/`; do not edit the original.
3. Include machine-readable provenance for every extracted value.
4. Compare extracted values against existing workbook values where a comparable period exists.
5. Treat values without source URL plus page/table evidence as review-only.

For generic-template changes, also check that no sector-specific company names, source URLs, or old project names remain in code, docs, templates, or workbook strings.

## Conventions

- Prefer official company filings, regulatory feeds, and investor-relations documents over scraper mirrors.
- Use structured sources without an LLM when available.
- Use LLM extraction only behind a strict JSON schema with period, unit, source, citation, and confidence fields.
- Keep source provenance explicit; do not collapse annual reports, IR decks, transcripts, and finance portals into vague "internet source" labels.
- Keep annual and quarterly modes behaviorally separate.
- Keep AI Studio batches within the current context budget and selected provider upload-file limit. The preparer defaults to batches of up to 6 companies, estimates prompt plus upload-source input against a 64k-token budget, auto-splits oversized batches, converts `.xlsx` and HTML-like source pages to `.txt`, uses compact SEC extracts instead of full Company Facts JSON, and mirrors only upload-ready attachments into each batch's `FILES_FOR_AI_STUDIO` folder. Provider limits live in `config/ai_provider_profiles.json`; Qwen and MiMo default to 5 files per batch, Kimi to 50, DeepSeek to 20, and Gemini uses token-budget splitting without a hard file-count cap.
- Keep the dashboard as a thin local control surface over the scripts. Source preparation, JSON validation, and Excel writing must remain in Python so the UI can be replaced without changing the data pipeline.
