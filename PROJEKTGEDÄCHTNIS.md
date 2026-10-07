# 📘 PROJEKTGEDÄCHTNIS – AI Memory System

> **Zweck:** In neuen Chats als erste Nachricht einfügen, um den kompletten
> Projektkontext wiederherzustellen.
> **Stand:** v3.6 (Pipeline-Framework läuft)
> **Letzte Aktualisierung:** 2026-09-19

---

## 1. PROJEKT-ÜBERBLICK

**Name:** AI Memory System
**Ziel:** Multi-Agenten-System für autonome Softwareentwicklung +
wissenschaftliche Arbeit auf Steam Deck. Langfristig: Local-First-Pipeline
mit Cloud als letzter Schritt.
**Philosophie:** So viel wie möglich lokal. Cloud nur für echte Denkarbeit.
**Umgebung:** Steam Deck (SteamOS, Desktop Mode), 16 GB RAM
**Python:** 3.13 (venv in `~/AI_Memory_System/venv/`)
**Web-UI:** http://localhost:8000

---

## 2. AKTUELLER STATUS (v3.6)

| Komponente | Status |
|---|---|
| LLM-Router (Multi-Provider) | ✅ 6 Provider aktiv |
| Aktives Chat-Modell | Cloudflare Llama-4-Scout (primär) |
| Fallback-Kette | Gemini → OpenRouter → Groq → Mistral → Ollama |
| Embedding | Gemini `gemini-embedding-2` |
| Sandbox | `rlimit`, Timeout 30s, 200 KB |
| Memory | SQLite + ChromaDB + DEC-*.md |
| RAG | ✅ funktioniert |
| Multi-Session Web-UI | ✅ |
| Backup/Restore | ✅ |
| Steam-Shortcuts | ✅ |
| **Pipeline-Framework (v3.6)** | ✅ **läuft** |
| **Input-Pipeline live** | ✅ 3 Spezialisten (Guardrail, Refiner, Prefilter) |
| **Ollama (Vulkan)** | ✅ Host-Service, 30m Keep-Alive |
| Guardrails | ✅ Injection + Secrets + PII |
| Audit-Log | ✅ SQLite + UI-Panel |
| Kosten-Tracking | ✅ SQLite + UI-Panel |
| Input-Veredelung | ✅ mechanisch + LLM |
| Context-Optimizer | ✅ Stage 1 + Stage 2 + Pipeline |
| Formel-DB | ✅ SQLite + SymPy + UI |
| ETHIK.md | ✅ |
| Obsidian-Doku | ✅ |

---

## 3. PROJEKTSTRUKTUR (aktuell)
AI_Memory_System/
├── agent_system.py # CLI-Entry
├── config.json # Zentrale Config
├── config.json.pre-v34a # Backup vor v3.4a
├── providers.toml # Provider-Registry (Runtime)
├── minimal_index.json # Projekt-Index
├── requirements.txt
├── .env # Keys (nicht einchecken!)
├── .env.example
├── .gitignore / .dockerignore
├── Dockerfile / docker-compose.yml
├── backup.sh / restore.sh
├── start_cli.sh / start_web.sh
├── start_web.sh.bak
├── README.md
├── ETHIK.md
├── PROJEKTGEDÄCHTNIS.md # ← dieses File
├── TOOL_IDEEN.md # (noch zu erstellen)
├── RECHERCHEN.md # (noch zu erstellen)
├── bootstrap_v3.3.sh # Installer
├── bootstrap_v3.2_part1/2.sh # (leer, Reste)
├── core/
│ ├── init.py
│ ├── config.py # zentrales .env-Load
│ ├── logger.py
│ ├── sandbox.py # rlimit
│ ├── memory.py # SQLite + DEC-.md
│ ├── tools.py # create_file, run_file, ...
│ ├── agent.py # Lead + Memory Agent
│ ├── llm.py # Router (6 Provider)
│ ├── embeddings.py # ChromaDB + Gemini
│ ├── rag.py # RAG
│ ├── prompts.py # Prompt-Loader
│ ├── refiner.py # Input-Veredelung
│ ├── optimizer.py # Stage 1 (mechanisch)
│ ├── optimizer_stage2.py # Stage 2 (LLM)
│ ├── guardrails.py # Injection/Secrets/PII
│ ├── audit.py # Audit-Log
│ ├── costs.py # Kosten-Tracking
│ ├── formulas.py # Formel-DB + SymPy
│ ├── pipeline.py # ← v3.6
│ ├── providers/
│ │ ├── base.py
│ │ ├── gemini.py
│ │ ├── openai_compat.py
│ │ ├── anthropic.py
│ │ └── factory.py # liest providers.toml
│ └── specialists/ # ← v3.6
│ ├── init.py
│ ├── base.py
│ ├── registry.py
│ ├── guardrail_specialist.py
│ ├── refiner_specialist.py
│ └── optimizer_specialist.py
├── prompts/ # Prompt-Bibliothek
│ ├── lead_agent.md
│ ├── memory_agent.md
│ ├── refiner.md
│ ├── classifier.md
│ ├── coder.md
│ ├── reviewer.md
│ ├── researcher.md
│ ├── scientist.md
│ ├── documenter.md
│ └── optimizer.md
├── web/
│ ├── server.py # FastAPI
│ └── static/
│ ├── index.html
│ └── app.js
├── scripts/
│ └── sessions.py
├── steam_deck/
│ ├── ai-memory-cli.desktop
│ ├── ai-memory-web.desktop
│ └── install_shortcuts.sh
├── tests/
│ ├── test_memory.py
│ ├── test_tools.py
│ ├── test_sandbox.py
│ └── test_rag.py
├── backups/
├── logs/agent.log
├── workspace/ # KI-generierter Code
└── memory_db/
├── decisions.db # SQLite (decisions, audit_log, api_calls, formulas)
├── DEC-.md # ADR-Markdown-Mirror
└── chroma/ # Vektor-Store
---

## 4. KONFIGURATION (Stand 2026-09-19)

### config.json (Kern)

```json
{
  "model": "gemini-3.5-flash-lite",
  "memory_model": "gemini-3.5-flash-lite",
  "embedding_model": "gemini-embedding-2",
  "timeout_seconds": 30,
  "max_repair_attempts": 2,
  "auto_sync_every_n_steps": 3,
  "rag": { "enabled": true, "top_k": 5 },
  "web": { "host": "0.0.0.0", "port": 8000, "auth_token": "" },
  "sandbox": { "enabled": true, "backend": "rlimit" },
  "git": { "auto_commit": true, "commit_prefix": "[AI]" }
}providers.toml (Runtime-Registry)
Aktive Provider mit Priorität:

Prio	Provider	Modell	Rolle
1	Cloudflare Workers AI	llama-4-scout-17b	Primär (schnell)
2	Google Gemini	gemini-3.5-flash-lite	Sekundär
3	OpenRouter	cohere/north-mini-code	Fallback
4	Groq	qwen3.8-27b	Klassifikation
5	Mistral	ministral-8b	Sekundär
99	Ollama (lokal)	qwen2.5:1.5b, llama3.2:3b	Notanker
Besonderheit: STRUCTURED_EXCLUDE = {"cloudflare"} – Cloudflare wird bei chat_structured übersprungen (unzuverlässig bei JSON).

Ollama-Setup
Container: distrobox ollama-box (Arch)

Ollama 0.34.2 mit Vulkan-Support

Host-Systemd-Service ~/.config/systemd/user/ollama.service

Vulkan aktiviert (3–4× schneller bei Prompt-Verarbeitung)

Keep-Alive 30 MinUser-Prompt
   ↓
[Guardrail-Specialist]     order 10  (Injection/Secrets/PII)
   ↓ (blockiert bei Injection)
[Refiner-Specialist]       order 30  (mechanisch + optional LLM)
   ↓
[Optimizer-Prefilter]      order 50  (mechanische Bereinigung)
   ↓
RAG-Kontext-Anreicherung
   ↓
[Lead-Agent]               (Cloudflare/Gemini)Framework:

Specialist (Basisklasse) – run(ctx) -> ctx

registry.py – Auto-Registrierung

Pipeline(category="input") – sequentielle Ausführung

run_input_pipeline() – Shortcut für Chat

Bei Fehler: stop_on_error=True → bricht sofort abSession-Log
   ↓
[Optimizer Stage 1]     mechanisch (Regex)
   ↓
[Optimizer Stage 2]     LLM-Klassifikation mit Confidence
   ↓
[Nur akzeptierte Items] an Memory-Agent
   ↓
[Memory-Agent]          ADR-Extraktion (Gemini via chat_structured)
   ↓
SQLite + Chroma + DEC-*.mdProvider-Router mit Runtime-Probing
providers.toml – Daten statt Code

factory.py – baut Provider-Instanzen

LLMRouter – Fallback-Kette nach Priorität

_generate_with_retry – Retry bei 429 mit Backoff

6. ADRs (60+, umgesetzt + geplant)
Umgesetzt (v3.3–v3.6)
ID	Entscheidung
ADR-001	Multi-Agenten statt Monolith
ADR-002	Function Calling statt Text-Tags
ADR-003	SQLite für Decisions
ADR-004	ChromaDB für Embeddings
ADR-005	Sandbox mit rlimit-Fallback
ADR-006	API-Key nur in .env
ADR-007	Auto-Sync alle 3 Schritte
ADR-008	Multi-Session per Cookie
ADR-009	Backup ohne .env
ADR-010	Gemini 3.5 Flash Lite als Hauptmodell
ADR-011	Multi-Provider-Router
ADR-012	thought_signature bei Tool-Calls
ADR-013	Free-Provider-Integration (Cloudflare, NVIDIA, OpenRouter, Groq, Mistral)
ADR-014	Clipboard-Bridge (Export/Import + Source-Tagging)
ADR-015	Input-Veredelung (mechanisch + LLM)
ADR-016	Context-Optimizer mit Kaskade + Confidence
ADR-017	Formel-DB mit SymPy
ADR-018	Prompt-Bibliothek + CoT
ADR-019	Guardrails (Injection + Secret-Scan)
ADR-020	Audit-Log in SQLite
ADR-021	Kosten-Tracking in SQLite
ADR-022	ETHIK.md mit Risikoklasse
ADR-023	Obsidian-Vault-Doku (memory_db/)
ADR-024	Specialist-Framework (Pipeline)
ADR-025	Cloudflare-Skip bei structured Outputs
Geplant (v3.7+)
ID	Entscheidung	Version
ADR-026	Rollen-System (Architect, Coder, Reviewer, Researcher)	v3.7
ADR-027	Local Intent-Classifier	v3.7
ADR-028	SQLite FTS5 (Hybrid-Suche)	v3.7
ADR-029	SQLite relations (leichter Graph)	v3.7
ADR-030	BGE-Reranker-v2-M3 statt Cohere	v3.7
ADR-031	BGE-M3 als Embedding (lokal)	v3.7
ADR-032	Experience-Store (Kategorie BA)	v3.7
ADR-033	Tool-Gateway mit Capability-Modell	v3.8
ADR-034	bubblewrap statt rlimit-only	v3.8
ADR-035	Agent-Registry (Governance)	v3.9
ADR-036	Router mit Local/Cloud-Umschalter	v3.9
ADR-037	Self-Evolution-Pipeline (Mem2Evolve)	v3.10
ADR-038	Cloud-Training-Adapter (Kaggle primär)	v3.11
ADR-039	System-Manager + Watchdog	v3.11
ADR-040	Nightly-Jobs mit RTC-Wake	v3.11
7. FALLSTRICKE (gelöst)
Diese Fehler sind aufgetreten und wurden gefixt:

Gemini 2.5 Flash „not available" → auf 3.5 Lite umgestellt

additionalProperties-Fehler → Pydantic konkret statt dict

thought_signature fehlt bei Function Calls → ToolCall.thought_signature

GEMINI_API_KEY fehlt in rag.py → load_dotenv zentral in core/config.py

bwrap: execvp ...: No such file → Sandbox auf rlimit

429 Quota → Lite + Retry + Multi-Provider

embed_query() got unexpected keyword 'input' → Chroma-Adapter angepasst

Podman Permission → --userns=keep-id

GitHub Models abgeschaltet → aus Roadmap entfernt

Cerebras Free Tier eingestellt → aus Roadmap entfernt

Cohere Rerank ToS-Verstoß → gestrichen

Groq 8k TPM = Council-Killer → nur für Klassifikation

PyFMI auf Deck nicht praktikabel → FMPy

ROCm unterstützt gfx1033 nicht → Vulkan

LoRA-Training auf Deck nicht möglich → Cloud (Kaggle)

which fehlt in Distrobox → Pfad direkt nutzen

Ollama-Service im Container → Host-Systemd-Bridge8.

8. BEDIENUNG
Lokaler Start
cd ~/AI_Memory_System
./start_web.sh          # → http://localhost:8000
./start_cli.sh          # CLI

Nützliche Befehle

# Server stoppen
pkill -9 -f "uvicorn web.server"

# Ports prüfen
ss -tlnp | grep :8000

# Backup
./backup.sh

# Ollama-Service
systemctl --user status ollama.service
systemctl --user restart ollama.service

# Pipeline-Test
python -m core.pipeline

# Prompt-Bibliothek
python -m core.prompts

# Refiner-Test
python -m core.refiner

# Guardrails-Test
python -m core.guardrails

# Formel-Test
python -m core.formulas

Web-UI-Panels
Chat – Hauptdialog

Workspace – Dateiliste

Entscheidungen – ADRs

Kontext-Bridge – Export/Import für fremde KIs

Statistik – 💰 Nutzung, 📋 Audit, 🧪 Formeln

Toggle „✨ Veredeln"
An: Eingabe läuft durch LLM-Refiner vor dem Agent

Aus: Roh-Eingabe wird direkt verarbeitet

9. ROADMAP (v3.7 – v3.11)

v3.6 (aktuell) ✅ Pipeline-Framework
  ↓
v3.7 – Experience-Store (BA) + Spezialisten
  ├── Experience-Store (8 Tabellen)
  ├── Language/Intent/Task-Spezialisten
  ├── Context-Selector
  ├── Prompt-Assembler
  └── Output-Spezialisten
  ↓
v3.8 – Tool-Gateway (BB)
  ├── Capability-Modell
  ├── Pydantic extra=forbid
  ├── bubblewrap-Sandbox
  ├── Rate-Limits (3 Ebenen)
  └── HITL-Tickets
  ↓
v3.9 – Agent-Registry (BC) + Router
  ├── Agent-Versionierung
  ├── Dedup-Check
  ├── Local/Cloud-Umschalter
  └── Auto-Fallback
  ↓
v3.10 – Self-Evolution (BE)
  ├── Telemetrie-Clustering
  ├── Agent-Spec-Generierung
  ├── Sandbox-Testing
  └── A/B-Tests
  ↓
v3.11 – Cloud-Training (BD) + System-Manager
  ├── Kaggle-Adapter
  ├── HF-Jobs-Adapter
  ├── Watchdog
  ├── Nightly-Jobs
  └── RTC-Wake

10. RECHERCHEN R1–R6 (Kurzfassung)
Details siehe RECHERCHEN.md.

Feld	Ergebnis
R1 GPU Steam Deck	Vulkan 3–4× schneller (Prompt), LoRA nicht möglich
R2 Cloud-Training	Kaggle Top-1 (30h/Woche T4 gratis)
R3 HF Jobs	Kostenpflichtig ($0.40–0.60/h)
R4 Tool-Gateway	Capability-Modell, bubblewrap, HITL
R5 Dynamic Specialists	Mem2Evolve (ACL 2026), Human Review Pflicht
R6 Experience-Store	SQLite + DuckDB + Parquet, kein Langfuse
11. NÄCHSTER SCHRITT IM NEUEN CHAT
Anweisung an die KI:

Ich habe das komplette Projekt im PROJEKTGEDÄCHTNIS.md.
Aktueller Stand: v3.6 läuft, Pipeline live.
Nächster Schritt: v3.7 (Experience-Store).
Bei Bau-Entscheidungen: Terminal-Befehle Schritt für Schritt.

Offene Punkte:

TOOL_IDEEN.md (Kategorien A–BH)

RECHERCHEN.md (R1–R6 im Detail)

v3.7 Experience-Store

12. VERWANDTE DATEIEN
TOOL_IDEEN.md – alle Kategorien A–BH

RECHERCHEN.md – R1–R6 im Detail

ETHIK.md – Selbstverpflichtung

README.md – Setup-Anleitung

config.json – zentrale Config

providers.toml – Provider-Registry

Ende PROJEKTGEDÄCHTNIS.md – Stand 2026-09-19


---

## ➡️ Was ich von dir brauche

1. **Datei speichern** (`nano` → einfügen → speichern)
2. **Verifizieren:**
   ```bash
   wc -l ~/AI_Memory_System/PROJEKTGEDÄCHTNIS.md
   head -5 ~/AI_Memory_System/PROJEKTGEDÄCHTNIS.md

## 5l. v3.7 - EXPERIENCE-STORE (2026-09-19, in Arbeit)

**Status:** Skeleton + Rollup + Chroma funktionieren. Retention/Archive offen.

**Neue Dateien:**
- core/experience/__init__.py
- core/experience/store.py (~600 Z.)
- core/experience/migrate.py (~180 Z.)
- core/experience/rollup.py (~480 Z.)
- migrations/001_init.sql (12 Tabellen)
- migrations/002_rollup_fields.sql (2 Indizes)
- tests/test_rollup_e2e.py (8 Tests, alle gruen)
- conftest.py (sys.path)

**DB:** ~/AI_Memory_System/memory_db/experience.db (eigene DB)

**WICHTIG - Zeitstempel-Format:**
- v3.7 Experience-Store: INTEGER (Unix-Sekunden, UTC)
- v3.3-v3.6: TEXT ISO-Format
- Cross-DB: DuckDB to_timestamp(started_at)
- Web-UI: datetime(started_at,'unixepoch') clientseitig UTC->lokal

**Tabellen (12):** content_blob, agent_run, agent_span, tool_call,
reasoning_step, run_score, run_adr, experience, run_daily,
prompt_version, adr, schema_migrations

**CLI:**
- python -m core.experience.migrate [--status]
- python -m core.experience.rollup --date YYYY-MM-DD
- python -m core.experience.rollup --yesterday
- python -m core.experience.rollup --backfill --days 30 --skip-chroma
- python -m core.experience.rollup --mark-eligible RUN_ID
- python -m core.experience.rollup --dry-run

**Chroma:** Collection 'experience' in CHROMA_DIR (parallel zu 'decisions').
Filter: train_eligible=1 AND status='ok' AND quality>=0.85.
Idempotent via LEFT JOIN experience. embedding_id == run_id.

**Offen (v3.7.1):** Presidio PII, retention.py + archive.py,
ref_count-Dekrement beim Loeschen.

## 5n. v3.7 Retention + Archive (2026-09-19, fertig)

**Neue Dateien:**
- core/experience/archive.py (~280 Z.)
- core/experience/retention.py (~350 Z.)
- tests/test_archive.py (8 Tests)
- tests/test_retention.py (9 Tests)

**Tests:** 25 gruen (rollup 8 + archive 8 + retention 9).

**Archive (5 Dateien/Tag):** runs, spans, tool_calls, scores, blobs
Dateien: archives/YYYY/MM/<kind>-YYYYMMDD.parquet (zstd)
Manifest: archives/YYYY/MM/manifest.sha256 (sha256sum -c-faehig)

**Retention-Stufen:**
- hot  (<=30d): unveraendert
- warm (30-180d): inline_content -> file (zst), file_path gesetzt
- cold (>180d): Parquet -> DELETE (CASCADE) -> ref_count--
                 -> Blob-Files nach archives/YYYY/MM/blobs/

**Sicherheit:**
- --dry-run ist Default; nur --apply aendert
- Verifikation (Parquet-Zeilen == SQLite-Zeilen) vor DELETE
- Atomare Transaktion (BEGIN IMMEDIATE / COMMIT / ROLLBACK)
- ref_count-Drift behoben

**CLI:**
- python -m core.experience.archive --date YYYY-MM-DD
- python -m core.experience.archive --range A..B
- python -m core.experience.archive --verify archives/YYYY/MM/
- python -m core.experience.retention             # dry-run
- python -m core.experience.retention --apply
- python -m core.experience.retention --vacuum

**Benoetigt:** pip install duckdb zstandard

**Offen:** schedule.py (Nightly-Timer), Audit-Hooks, query.py

## 5o. v3.7.1 - Audit-Hooks + Few-Shot-Query (2026-09-19, fertig)

**Neue Dateien:**
- core/experience/audit_hooks.py (Schema-agnostisch, env-override)
- core/experience/query.py (Chroma Few-Shot-Abfrage)
- tests/test_audit_hooks.py (9 Tests)
- tests/test_query.py (9 Tests)

**Integrationen:**
- retention.py: emit_retention_dryrun / emit_retention_applied
- archive.py: emit_archive_written nach Manifest-Write
- agent.py: query_experiences + format_few_shots im run_lead_agent
- config.json: rag.use_few_shots=true

**Gesamt-Tests:** 48 passed, 1 skipped (test_memory pre-v3.7).

**Feature-Flag:** rag.use_few_shots steuert Few-Shot-Block.
Ohne Flag: Flow bit-identisch zu v3.7.

## 5p. v3.7.2 - RTC-Wake (2026-09-20, fertig)

**Ziel:** Steam Deck soll aus Standby aufwachen fuer 03:00-Nightly-Jobs.

**Neue Dateien:**
- scripts/rtcwake_arm.sh (armiert RTC-Alarm fuer naechsten 03:00)
- scripts/install_root_wake.sh (installiert nach /usr/local/sbin + /etc/systemd/system)
- scripts/test_rtcwake_root.sh (Status-Check)
- scripts/systemd/ai-memory-rtcwake.service
- scripts/systemd/ai-memory-rtcwake.timer

**Installiert:**
- /usr/local/sbin/ai-memory-rtcwake-arm.sh
- /etc/systemd/system/ai-memory-rtcwake.{service,timer}

**Ablauf:**
- Root-Timer feuert taeglich 23:00 -> setzt RTC-Alarm fuer naechsten 03:00
- User-Timer ai-memory-experience.timer laeuft 03:00 (Persistent=true)
- Van Gogh: rtcwake funktioniert (verify: Alarm 2026-09-21 03:00:00 CEST)

**Verifikation:**
- systemctl status ai-memory-rtcwake.timer -> active (waiting), Next 23:00
- Service-Status: code=exited, status=0/SUCCESS
- RTC-Wert: 1789952400 (2026-09-21 03:00:00 CEST)

**Bekannte Limitierung:**
- Datei in /usr/local/sbin/ kann bei SteamOS-Update verloren gehen
- Gegenmassnahme: install_root_wake.sh erneut ausfuehren
- Timer in /etc/systemd/system/ ist persistent


## 5q. v3.7.2 - HF Evidence Layer (Basis, 2026-09-20)

**Ziel:** Hugging Face als externe Evidenz-Schicht (Modelle, Eval Results,
Model Cards) in lokaler SQLite.

**Neue Dateien:**
- core/hf_collector.py (~270 Zeilen)
- scripts/test_hf_collector.py
- memory_db/hf_registry.db (isoliert von experience.db)

**Drei Tabellen:**
- models (model_id, author, pipeline_tag, library_name, tags,
  downloads, likes, license, last_modified, gated, private, sha,
  fetched_at)
- model_cards (model_id, card_text, fetched_at)
- eval_results (model_id, dataset_id, task_id, metric_value, verified,
  source, eval_date, fetched_at)

**HF-API-Endpunkte (verifiziert):**
- GET /api/models/{id} -> Metadaten
- GET /api/models/{id}?expand=evalResults -> Eval Results
- GET /{id}/raw/main/README.md -> Model Card
- GET /api/models?search=...&limit=N -> Suche

**Nur stdlib:** urllib, sqlite3, json, time, pathlib.
Kein huggingface_hub.

**CLI:**
- python -m core.hf_collector Qwen/Qwen3-8B [--card]
- python scripts/test_hf_collector.py

**Verifikation:**
- Qwen/Qwen3-8B: Info=True, Evals=1, Card=1
- Idempotenz: Lauf 2 ergibt identische Zahlen
- sqlite3-CLI-Lesezugriff funktioniert

**Bugs gefixt (Lesson 1.11):**
- task_id liegt unter data.dataset.task_id, nicht data.task_id
- nano-Suchen-Ersetzen erzeugte Vertipper (dsx statt ds)

**Offen (v3.7.2+):**
- Bulk-Import, Query-API (find_by_license, find_by_task)
- Eval-Aggregation, CLI-Erweiterung
- eval_date als Unix-Timestamp (Konsistenz mit Experience-Store)


## 5r. v3.7.2 - Vulkan + Build-Recherche (2026-09-29/30)

**Vulkan-Test (29.09.):**
- Container `llama-vulkan-test` eingerichtet
- llama.cpp mit Vulkan-Backend gebaut
- RADV VANGOGH bestaetigt
- 1.5B: 362 t/s pp512, 54 t/s tg128 (Vulkan)
- 7B: 69 t/s pp512, 11 t/s tg128 (Vulkan)
- 14B blockiert durch GTT (8 GB)

**GTT-Tuning:**
- GRUB-Timeout auf 5 Sek gesetzt
- `amdgpu.gttsize=10240` noch nicht eingetragen
- Reboot ausstehend

**Build-Recherche (30.09.):**
- CachyLLama: Vulkan-Build, gfx103X Ziel, mmap+residency
- PR #27861: GPU Expert Cache, RADV funktioniert
- PR #25294: SSD-Streaming, offen, nicht Vulkan-validiert
- set_tensor_async() auf RADV effektiv synchron

**Naechster Schritt:**
- GTT-Tuning durchfuehren
- CachyLLama bauen und testen
- 30B MoE Streaming-Test

**Status:** Vulkan-Basis laeuft. Streaming ungetestet.

**Referenz:** RECHERCHEN R22-R26, ARCHITEKTUR.md

---

## 5s. CachyOS-Migration (2026-10-02, laufend)

**Status:** Vollzogen. SteamOS-Abbild als Datenquelle auf HDD erhalten.

**Grund der Migration:**
- SteamOS ist für Gaming optimiert, nicht fuer Kernel-Tuning, io_uring
  oder Performance-Bastelei.
- CachyOS bringt aktuellen Kernel (7.2.8), aktuelle Mesa/RADV,
  io_uring direkt nutzbar, kein Container-Layer.
- Ziel: autonome Arbeitsweise mit voller Systemkontrolle.

**Neue Umgebung:**
- Distribution: CachyOS, btrfs (Subvolumes /@, /@home, ...)
- Kernel: 7.2.8-2-cachyos
- Snapshot-System: limine-snapper-sync
- Python: 3.14.7 (uv-verwaltet, nicht System-Python)
- Paketmanager: uv 0.12.22 (statt pip+venv)
- Manifest: pyproject.toml + uv.lock
- 108 Pakete, Installation in 472 ms

**GTT-Status:**
- Aktuell 7409 MB (Kernel-Default, kein ttm.pages_min gesetzt)
- Ziel: 10 GB via ttm.pages_min=2621440 (offen)

**Was laeuft:**
- LLM-Router: 6 Provider aktiv, OpenRouter Prio 0 (DeepSeek V4.1-Flash)
- Ollama: ollama-vulkan, iGPU aktiv (OLLAMA_IGPU_ENABLE=1), 100% GPU
- Web-UI: 127.0.0.1:8000, uv-Launcher (start_web.sh)
- Autonomie: Executor + Guard + Snapper + Job-Queue + Worker

**Neu gebaut (2026-10-02):**
- core/exec_guard.py — Command-Blacklist
- core/executor.py — Shell-Ausfuehrung mit Snapshot + Guard
- core/experience/jobs.py — persistente Job-Queue
- core/experience/worker.py — autonomer Worker
- web/server.py: /jobs, /jobs/<built-in function id>, /jobs/<built-in function id>/cancel
- migrations/003_agent_jobs.sql (Job-Tabelle)
- migrations/004_agent_job_recovery.sql (Worker-Lease-Felder)

**Offene Aufgaben:**
1. Job-zu-agent_run-Verlinkung
2. systemd-User-Service fuer Worker
3. Autonomie-Doku in ARCHITEKTUR.md
4. Streaming-Test (CachyLLama, Qwen3-30B-A3B)
5. GTT-Tuning auf 10 GB
6. Modelle via Ollama-Modelfile registrieren (qwen2.5:1.5b-local existiert)

**Referenz:** SYSTEM_INVENTAR.md §9, LESSONS_LEARNED.md Lesson 16



