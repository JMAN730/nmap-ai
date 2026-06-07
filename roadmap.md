# NMAP-AI Roadmap

## Decision log

- **2026-06-07** — Phase 0 fork: chose **heuristic-honest** path. Rationale: the current `AIEngine` and `AIScriptGenerator` are entirely rule-based; no ML models exist on disk despite `requirements.txt` carrying ~3GB of TensorFlow/PyTorch/transformers. Cleanup is cheaper than implementing real ML, and the project is more useful as an honest heuristic scanner than a misleading "AI" one. Real ML can be revisited as a Phase 3 extension once the heuristic baseline is solid.

---

Findings that shaped this roadmap: tests exist (`tests/conftest.py`, `tests/unit/test_vulnerability_detector.py`, `tests/integration/test_api_endpoints.py`) — there's a scaffold to build on. The generated NSE scripts are syntactically valid Lua templates but the vulnerability checks themselves are stubs (`-- Add actual test logic here`). The `AIEngine` is rule-based, not ML, despite `.pkl` paths in config and heavy ML deps in `requirements.txt`.

## Phase 0: Truth in advertising (1–2 days, blocks everything else)

The single biggest problem: the README and feature list promise things the code doesn't do. Pick a direction before building more.

| Item | Verify by |
|---|---|
| ~~Decide: real ML or heuristic engine?~~ **DONE 2026-06-07** — chose heuristic (see Decision log) | ✅ |
| Remove `tensorflow`, `torch`, `transformers` (and other unused deps) from `requirements.txt`; rewrite README "AI Capabilities" section as "Heuristic-based scan intelligence" | `pip install -r requirements.txt` completes in <30s; README claims match code behavior |
| Strip non-existent `.pkl` model paths from `AIConfig` in `config.py` | `AIConfig` only references fields the code actually uses |

## Phase 1: Make the existing surface honest (~1 week)

Finish or remove the half-built pieces.

| Item | Verify by |
|---|---|
| Implement or delete `_export_xml` / `_export_csv` in `scanner.py:342-349` (currently `pass`) | `nmap-ai scan ... --format csv -o out.csv` produces a valid CSV; or the option is removed from the CLI |
| Generated NSE scripts: replace `-- Add actual test logic here` stubs with working HTTP requests (start with `xss` and `sql_injection` since they share the HTTP path) | A generated script runs against `scanme.nmap.org` without Lua errors |
| Web API: either implement `POST /api/v1/scan` + `GET /api/v1/results/{id}` against `NmapAIScanner`, or strip the 501 stubs from `web/main.py:123-129` | `/docs` only shows endpoints that return ≠501 |
| GUI: either build a working scan form in `gui/main.py` (it's currently a single "under development" label) or remove the `--gui` flag | `python -m nmap_ai --gui` launches something usable, or the flag errors with "not available" |

## Phase 2: Test foundation (~1 week, parallel with Phase 1)

You can't refactor what you can't verify.

| Item | Verify by |
|---|---|
| Add `tests/unit/test_scanner.py` covering `NmapAIScanner` with a mocked `nmap.PortScanner` | `pytest tests/unit/test_scanner.py` passes; covers happy path + invalid target + async path |
| Add `tests/unit/test_ai_engine.py` covering `optimize_scan_arguments`, `_analyze_target_result`, `create_scan_plan` | Coverage of `core/ai_engine.py` ≥ 70% |
| Add `tests/unit/test_config.py` covering YAML round-trip + the `field(default_factory=...)` fix in commit c61d49f (regression guard) | Loading a saved config returns equal dataclass |
| Wire up CI: GitHub Actions running `pytest`, `black --check`, `flake8`, `mypy` on PRs | Green check on a sample PR |

## Phase 3: Real intelligence layer (~2–4 weeks, only if Phase 0 picked "real ML")

This is the work that justifies the "AI" name.

| Item | Verify by |
|---|---|
| **Port prediction**: train a sklearn classifier (RandomForest / LightGBM) on a public port-scan dataset (e.g. CICIDS2017) to predict likely-open ports given host fingerprint hints | Model file in `models/`, integration test shows `optimize_scan_arguments` produces different port lists for "web-like" vs "db-like" targets |
| **Vulnerability classification**: replace the if/elif tree in `_analyze_target_result` with a model trained on CVE/service mappings | Detection precision/recall measured on a labeled test set, documented in `docs/ai-models.md` |
| **Script generation**: this is the hardest — a real solution needs either a fine-tuned code model or a structured grammar. If LLM-based, integrate via the Anthropic SDK rather than bundling transformers | One generated script for each of 3 target types passes `nmap --script-help` without errors |

## Phase 4: Operational polish (~1 week)

Lower priority but unblock real use.

| Item | Verify by |
|---|---|
| Persist scan history beyond process lifetime (currently `self.scan_history: List[Dict]` lives in memory only — `nmap-ai history` always shows empty for a fresh process) | `nmap-ai scan ...` then a *new* `nmap-ai history` shows the prior scan. SQLAlchemy is already in `requirements.txt` — wire it to `DatabaseConfig.url` |
| Hardcoded `secret_key: "change-me-in-production"` in `WebConfig` — load from env var, fail fast if web mode starts with the default | Web server refuses to start in non-debug mode without `NMAP_AI_SECRET_KEY` set |
| Plugin discovery has no default plugin directory configured — `PluginManager` is instantiable but useless out of the box | Documented default dirs (e.g. `~/.nmap-ai/plugins/`, `./plugins/`) and a working example plugin in `examples/` |

## Sequencing

```
Phase 0 ─┐
         ├─► Phase 1 ──┐
Phase 2 ─┘             ├─► Phase 3 (only if Phase 0 chose ML)
                       └─► Phase 4
```

Phase 0 is non-negotiable — the credibility gap will sink contributor goodwill if left alone. Phase 2 should run in parallel because the existing code has zero regression protection. Phase 3 is the most exciting but also where ambitious projects die; ship one small real model before promising a suite.
