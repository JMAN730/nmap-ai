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
| ~~Remove `tensorflow`, `torch`, `transformers` (and other unused deps) from `requirements.txt`; rewrite README "AI Capabilities" section as "Heuristic-based scan intelligence"~~ **DONE 2026-06-07** (commit 8e900eb) — also aligned `pyproject.toml`/`setup.py` deps & extras, fixed broken console-script entry points | ✅ |
| ~~Strip non-existent `.pkl` model paths from `AIConfig` in `config.py`~~ **DONE 2026-06-07** (commit 8e900eb) | ✅ |

## Phase 1: Make the existing surface honest (~1 week)

Finish or remove the half-built pieces.

| Item | Verify by |
|---|---|
| ~~Implement or delete `_export_xml` / `_export_csv` in `scanner.py` (currently `pass`)~~ **DONE 2026-06-07** (commit 0c46a31) — both implemented + unit-tested | ✅ |
| ~~Generated NSE scripts: replace `-- Add actual test logic here` stubs with working HTTP requests (xss, sql_injection)~~ **DONE 2026-06-07** (commit a73da0d) — real http.get logic; also fixed a Lua 5.3 `math.random(float)` crash; all 6 variants parse as valid Lua. *Note: not yet run against live `scanme.nmap.org` (no nmap/Lua runtime in dev env); validated via `luaparser`.* | ✅ (parse-validated) |
| ~~Web API: implement or strip the 501 stubs~~ **DONE 2026-06-07** (commit 15b45f4) — stripped the two HTTP 501 stub endpoints; extracted `create_app()` + module-level `app` for testability | ✅ |
| ~~GUI: build a working scan form or make `--gui` honest~~ **DONE 2026-06-07** (commit c456923) — chose honest stub: `--gui` exits non-zero with a pointer to the CLI (no fake window) | ✅ |

## Phase 2: Test foundation (~1 week, parallel with Phase 1)

You can't refactor what you can't verify.

| Item | Verify by |
|---|---|
| ~~Repair the broken test scaffold so the suite collects at all~~ **DONE 2026-06-07** (commit 15b45f4) — fixed `Config` imports, invalid pytest hook, ML import gate; rewrote API integration test to the real contract | ✅ |
| ~~Add `tests/unit/test_scanner.py` covering `NmapAIScanner` with a mocked `nmap.PortScanner`~~ **DONE 2026-06-07** (commit 0c46a31) — happy path, invalid target/ports, exporters, save dispatch | ✅ (async path still TODO) |
| ~~Add `tests/unit/test_ai_engine.py` covering `optimize_scan_arguments`, `_analyze_target_result`, `create_scan_plan`~~ **DONE 2026-06-07** — `core/ai_engine.py` coverage ~96% | ✅ |
| ~~Add `tests/unit/test_config.py` covering YAML round-trip + the `field(default_factory=...)` fix in commit c61d49f (regression guard)~~ **DONE 2026-06-07** — YAML+JSON round-trip equal; instance-independence guard | ✅ |
| ~~Wire up CI: GitHub Actions running `pytest`, `black --check`, `flake8`, `mypy` on PRs~~ **DONE 2026-06-07** — `.github/workflows/ci.yml`: pytest gating on py3.9–3.12; **black/flake8/mypy advisory** (continue-on-error) pending the cleanup below | ✅ (lint advisory) |

**Lint/type debt (follow-up, slots into Phase 4):** the legacy tree is far from clean — `black --check` would reformat ~35 files and flake8 reports ~1400 issues, and mypy strict has not been run to green. CI runs these advisory today. Pay down per-area and flip each tool to gating in `ci.yml` as it reaches green. A `.flake8` already excludes the dead `nmap_ai/cli/commands` tree and `examples/`.

**Dead code (follow-up):** `nmap_ai/cli/commands/` is a parallel CLI implementation that imports the non-existent `Config` class and is wired into nothing (the real CLI is `nmap_ai/cli/main.py`). Likewise several `examples/` import `Config`. Decide: delete or repair.

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
