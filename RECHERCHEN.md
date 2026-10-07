# 🔬 RECHERCHEN – R1 bis R6

> **Zweck:** Kompakte Zusammenfassung der Recherchen R1–R6 (Sept 2026).
> **Rohprotokolle:** in ~/Downloads/deepseek_markdown_*.md und ggf. weiteren Quellen.
> **Stand:** 2026-09-19

---

## Übersicht

| Nr | Thema | KI | Note | Entscheidung |
|----|-------|-----|------|--------------|
| R1 | GPU auf Steam Deck | Claude | 10/10 | Vulkan aktiv, LoRA nein |
| R2 | Cloud-GPU-Training | Claude | 10/10 | Kaggle primaer |
| R3 | HF Jobs / Unsloth | Grok | 9,5/10 | Bezahlt als Overflow |
| R4 | Tool-Gateway | Grok | 10/10 | Capability + bwrap + HITL |
| R5 | Dynamic Specialists | Perplexity | 10/10 | Mem2Evolve, Human Review Pflicht |
| R6 | Experience-Store | Grok + ChatGPT | 9,5/10 | SQLite + DuckDB + Parquet |

**Erkenntnis:** Grok ist auf Claude-Niveau (10/10 bei R4). Nutzen wir als Alternative bei Task-Limits.

---

## R1 – GPU auf Steam Deck für LLMs

**Kernergebnisse:**

| Metrik | Vulkan | CPU-only | Faktor |
|--------|--------|----------|--------|
| Prompt eval 1.5B | 92,13 t/s | 22,96 t/s | 4x |
| Prompt eval 3B | 34,94 t/s | 11,47 t/s | 3x |
| Token eval 1.5B | 22,73 t/s | 22,26 t/s | 1x |
| Token eval 3B | 10,88 t/s | 11,14 t/s | 1x |

**Erkenntnisse:**
- Vulkan laeuft auf Steam Deck (Ollama >= 0.30)
- Prompt-Verarbeitung 3-4x schneller (wichtig fuer RAG!)
- Token-Generierung bandbreiten-limitiert
- GPU-Peaks bis 100 Prozent waehrend Prompt-Verarbeitung

**LoRA-Training:**
- NICHT moeglich auf Steam Deck
- ROCm unterstuetzt gfx1033 (Van Gogh) offiziell nicht
- Unsloth gated gfx1033 auf CPU
- Vulkan-Compute fuer Training existiert nicht

**Kleine Modelle (<3B):**
- 1.5B halluziniert Woerter und Fakten
- Nur fuer Klassifikation/Routing nutzbar
- NICHT fuer Freitext-Generierung

**Setup:**
- Ollama 0.34.2 in Distrobox-Container
- Host-Systemd-Service mit Vulkan
- Keep-Alive 30 Min
- Modelle: qwen2.5:1.5b, llama3.2:3b

**Quellen:** Ollama 0.30 Release, ROCm Issues, llama.cpp Vulkan-Reviews.

---

## R2 – Cloud-GPU-Training ohne Kreditkarte

**Anbieter-Vergleich (Sept 2026):**

| Anbieter | Kostenlos | Karte | API | LoRA | Limit |
|----------|-----------|-------|-----|------|-------|
| Kaggle | Ja | Nein | Ja | Ja | 30h/Woche T4 |
| Lightning | teilweise | Nein (5 Cr) | Ja | Ja | 5-30 Credits |
| Colab | Ja (ungarantiert) | Nein | Nein | Ja | max 12h |
| HF Jobs | Nein | Ja | Ja | Ja | 0.40-2.50/h |
| Modal | 30 USD/Monat | Widerspruch | Ja | Ja | 30 USD/Monat |
| Together | Nein | Ja (5 USD) | Ja | Ja | 4 USD min |
| Fireworks | Nein | praktisch | Ja | Ja | 0.50/1M Tok |

**Kaggle ist Top-1:**
- 30h/Woche T4 kostenlos
- Keine Kreditkarte (nur Telefon-Verifizierung)
- CLI + Python-API
- Session max 9h

**Wichtig:**
- Kaggle P100 abgeschaltet 15.09.2026 -> nur noch T4
- Kaggle-Kernel-CLI: `kaggle kernels push -p DIR --accelerator NvidiaTeslaT4 -t 10800`
- Kein Cancel-Befehl
- Keine Live-Logs
- Kein T4x2 per API

**Trainings-Dauer (geschaetzt):**
| Modell | 1000 Beispiele | 5000 Beispiele |
|--------|----------------|----------------|
| 1.5B QLoRA | 15-25 Min | 1,2-1,9 h |
| 7B QLoRA | 1,2-1,8 h | 6-9 h (Grenze) |

**Adapter-Pattern (TrainBackend):**
```python
class TrainBackend(Protocol):
    name: str
    def submit(self, spec: TrainSpec) -> JobHandle: ...
    def status(self, h: JobHandle) -> str: ...
    def fetch(self, h: JobHandle, out_dir: Path) -> Path: ...
```

**Quellen:** Kaggle-CLI-Doku, Lightning Pricing, Modal Pricing, HF Jobs Pricing.


## Nachtrag zu R2 - NVIDIA NIM Status (19.09.2026)

**Update zu NVIDIA NIM / build.nvidia.com:**

Stand 19.09.2026:
- Verifizierungsproblem offiziell bestaetigt von NVIDIA.
- Fix in Arbeit (Engineering-Team).
- Noch kein allgemeiner Fix.
- Betroffene Nutzer: "Please contact support to verify your account".
- Offizieller Weg: help@build.nvidia.com mit registrierter
  E-Mail, Telefonnummer, Fehlermeldung, Screenshot.

**Konsequenz fuer Konzept:**
- NVIDIA bleibt optionaler Provider.
- Nicht als zuverlaessig verfuegbar einplanen.
- Provider-Gateway bleibt agnostisch.
- Aktivierung sobald Account freigeschaltet.

---

## R3 – HF Jobs / Unsloth

**Ergebnis:** Nur mit Guthaben nutzbar, nicht kostenlos.

**Preise (offiziell):**
- T4-small: 0.40 USD/h
- T4-medium: 0.60 USD/h
- L4: 0.80 USD/h
- A100-80GB: 2.50 USD/h
- H200: 5.00 USD/h

**Unsloth 2026.9.7:**
- QLoRA, LoRA, DPO, GRPO, GSPO, Full-FT, FP8, GGUF-Export
- Fertige UV-Scripts: uv-scripts/unsloth-jobs, unsloth/jobs
- `hf jobs uv run` fuer automatische Jobs

**Wichtige Korrekturen:**
- HF Jobs braucht positiven Credit-Balance
- Unsloth-Jobs-Programm (20.02.2026): einmalig, nicht verifizierbar ob noch aktiv
- Default-Timeout 30 Min -> killt Training ohne `--timeout`
- Job-Disk ephemeral -> push_to_hub vor Exit zwingend
- PRO allein reicht oft nicht

**Karte:** HF Compute braucht Kreditkarte (Stripe), Jobs brauchen Credits.

**Empfehlung:** Kaggle primaer, HF Jobs als bezahlter Overflow.

---

## R4 – Tool-Gateway (KRITISCH!)

**Wichtige Korrekturen:**

| Meine Annahme | Realitaet |
|---------------|-----------|
| Agent darf nach Capability fragen | Angriffsflaeche! Grant entscheidet Orchestrator beim Spawn |
| rlimit reicht | Nicht hinreichend -> bubblewrap |
| MCP = einfache Anbindung | Groesste Tool-Angriffsflaeche 2026 |
| CrewAI allow_code_execution | CVE-2026-2275 (RCE) -> deprecated |
| AutoGen | Maintenance -> Microsoft Agent Framework |
| Swarm | Deprecated -> OpenAI Agents SDK |

**Capability-Modell (nicht RBAC):**
```python
@dataclass(frozen=True)
class ToolConstraint:
    path_prefix: str = "/home/deck/agent-ws"
    max_bytes: int = 200_000
    max_calls: int = 8
    timeout_s: float = 30.0
    network: bool = False
    hitl: bool = False
```

**Kernprinzipien:**
- Grant beim Spawn, TTL = Run
- Pydantic `extra=forbid` gegen Extra-Args
- 3 Rate-Limit-Ebenen: Global x Agent x Tool-Agent
- Output wrappen: `<tool_result untrusted="true">`
- Max 6 Tools pro Agent im JSON-Schema
- HITL fuer create_file, run_file, pip_install

**Sandbox:**
- **bubblewrap** (`bwrap --unshare-all`)
- SteamOS hat bwrap (Flatpak-Dependency)
- Fallback: rlimit-only mit Logging-Warnung

**Bekannte CVEs (2025-2026):**
- CVE-2025-54136 (MCPoison)
- CVE-2025-54135 (CurXecute)
- CVE-2026-2275 (CrewAI RCE)
- CVE-2026-32979 (OpenClaw TOCTOU)
- arXiv:2609.18217 (Cross-channel fragmentation)

**Latenz:** Gateway in-process = Mikrosekunden. Sandbox-Spawn ~10-50 ms.

---

## R5 – Dynamic Specialist Creation

**Beste Referenz: Mem2Evolve (ACL 2026)**
- "Reuse first, Create on demand"
- Dual-Memory: Asset Memory + Experience Memory
- Self-Correction-Loop
- Erfahrungs-Guidance: +36 Prozent First-Pass-Validity
- 70,24 Prozent avg Pass@1 ueber 8 Benchmarks

**Wichtigste Erkenntnis:**
Der exakte Anwendungsfall (Telemetrie-Clustering -> neue Spezialisten) ist NICHT als geschlossenes System in der Literatur beschrieben. Bausteine existieren, Kombination nicht.

**Warnungen:**
- Darwin Goedel Machine hat "gecheated" -> Validierung muss manipulationssicher sein
- Human Review PFLICHT bei neuen Agenten
- Max 10-15 aktive Agenten operativ
- Dedup-Check (Similarity > 0.85 -> Block)
- Auto-Archiv bei Inaktivitaet > 30 Tage

**Pipeline:**
```
Telemetrie (10k+ Tasks)
  -> Feature-Extraction + Clustering (HDBSCAN)
  -> Bedarfs-Hypothese
  -> Agent-Spec-Generierung (LLM)
  -> Sandbox-Testing (Docker, Self-Correction)
  -> A/B-Test (50+ Runs, p<0.05)
  -> Human Review Gate
  -> Deployment (Canary -> Full)
```

**Wichtige Papers:**
- Mem2Evolve (ACL 2026)
- Darwin Goedel Machine (arXiv:2505.22954)
- ADAS (ICLR 2025)
- SwarmAgentic (arXiv:2506.15672)
- AFlow (MetaGPT-Community)
- AgentSquare (ICLR 2025)
- Self-Evolving Agents Survey (arXiv:2507.21046)

---

## R6 – Experience-Store

**Konsens Grok + ChatGPT:**

**8 neue SQLite-Tabellen:**
1. agent_run (Root: ein Agent-Lauf)
2. agent_span (LLM/Tool/Sub-Agent-Span)
3. tool_call (normalisierte Tool-Aufrufe)
4. reasoning_step (geordnete Denkschritte)
5. run_score (Evaluierungen)
6. run_adr (ADR-Verknuepfung)
7. experience (Few-Shot-Index)
8. run_daily (Tages-Rollups)

**Plus:**
- content_blob (Hash-basiert, inline/file)
- prompt_version (Versionierung)
- adr (Entscheidungen)

**Speicherung:**
- SQLite = Source of Truth (OLTP)
- DuckDB = Analytics (OLAP)
- Parquet = Archiv (zstd)
- ChromaDB = Semantische Suche
- JSONL = Training-Export

**Retention:**
- Hot: 30 Tage (SQLite, volle Blobs)
- Warm: 180 Tage (SQLite, ohne inline)
- Cold: unbegrenzt (Parquet)

**Archivierung statt Loeschen!**

**Design-Prinzipien:**
- Root: agent_run (nicht api_call)
- Tool-Calls als Kind-Zeilen (nicht JSON-Array)
- Reasoning als reasoning_step (nicht Text-Feld)
- Content per SHA256-Hash
- train_eligible=0 Default
- PII-Redaktion vor Speicherung (Presidio)

**Kein Langfuse/Helicone/Opik auf Deck:**
- Langfuse: braucht ClickHouse (min 8 GB RAM)
- Helicone: Maintenance seit 3. Maerz 2026
- Opik: braucht ClickHouse

**Phoenix 20.14.0 als optionale UI:**
- ELv2 Lizenz
- pip install arize-phoenix und phoenix serve
- Default SQLite
- ~300-800 MB RAM

**Bibliotheken:**
- SQLite 3.53.4
- DuckDB 1.5.5
- Polars 1.44.2
- Pandas 3.0.6
- Presidio 2.2.362
- TRL (aktuell)
- Unsloth 2026.9.7
- Phoenix 20.14.0

**Datenmenge (geschaetzt):**
| Inhalt pro Run | grob |
|----------------|------|
| Metadaten | 1-3 KB |
| Input | 1-8 KB |
| Output | 1-10 KB |
| 5 Tool Calls | 5-25 KB |
| Events/Evaluations | 2-10 KB |

1000 Runs: ~10-60 MB (konservativ) bis 500+ MB (volle CoT).

---

## Auswirkungen auf das Projekt

**Neue ADRs:**
- ADR-051: Kaggle primaere Trainings-Cloud
- ADR-052: HF Jobs als bezahlter Overflow
- ADR-053: TrainBackend-Adapter-Pattern
- ADR-054: Training nur mit validierten Daten
- ADR-055: Experience-Store als Lern-Basis
- ADR-056: Capability-Modell statt RBAC
- ADR-057: bubblewrap statt rlimit-only
- ADR-058: Agent-Registry als Governance-Pflicht
- ADR-059: Human Review Gate bei neuen Agenten
- ADR-060: Mem2Evolve-Paradigma
- ADR-061: ChatML-JSONL als SFT-Format
- ADR-062: Presidio fuer PII-Redaktion
- ADR-063: Phoenix als optionale UI
- ADR-064: Ollama als Host-Systemd-Service

**Neue Kategorien in TOOL_IDEEN.md:**
- AR: Pipeline-Framework
- AS: Input-Spezialisten
- AT: Output-Spezialisten
- AU: Router (Local/Cloud)
- AV: System Manager
- AW: Learning-Layer
- AX: Ollama-Setup
- AY: Tool-Gateway
- AZ: Agent-Registry
- BA: Experience-Store
- BB: Self-Evolution
- BC: Cloud-Training-Adapter
- BD: HF Jobs Detail
- BE: Phoenix Observability
- BF: Guardrails & HITL
- BG: Systemd-Services
- BH: Lokale LLM-Nutzung
- BI: Recherche-Tools
- BJ: Systemd-Watchdog
- BK: Nightly-Jobs
- BL: Archivierung
- BM: Datenanalyse
- BN: Vorlagen-Sammlung
- BO: Ideen-Parkplatz

**Naechster Baustein:** Experience-Store (Kategorie BA).

---

## Recherche-Methodik (fuer spaeter)

**Verfuegbare KIs (mit Websuche):**

| KI | Note technisch | Task-Limit |
|----|----------------|------------|
| Claude | 10/10 | begrenzt |
| Grok | 10/10 | begrenzt |
| ChatGPT | 9/10 | begrenzt |
| Perplexity | 9/10 | begrenzt |
| DuckDuckGo AI | verworfen | kein Websuche-Zugriff |

**Template-Muster:**
1. Kontext (Zielplattform, Stack)
2. Vorhandenes Wissen
3. Konkrete Fragen
4. Kritische Anweisungen (keine Allgemeinplaetze)
5. Ausgabeformat (Veraltet / Neu / Tools / Code / Warnungen / Quellen)

**Erwartete Antwort-Formate:**
- Was ist VERALTET?
- Was ist NEU?
- Konkrete Tools mit Version/Lizenz/Limit
- Code-Beispiele (Python)
- Empfehlung
- Warnungen
- Quellen mit Datum

## R7 - Modellkataloge fuer lokale KI-Agenten
**Datum:** 2026-09-20
**Quelle:** vorgefertigte_modelle_fuer_lokale_ki_agenten_2026-09-20.md

**Kernergebnisse:**
- Katalog nach FUNKTION statt Hersteller: General, Reasoning, Agentic, Coding, Math, Science, Multilingual, Vision-Language, Document, OCR, STT, TTS, Audio, Embedding, Reranker, Classification, Guard, Edge, Image-Gen, Image-Edit, Video, 3D, Domain.
- ~40 Modellfamilien gelistet: Llama, Qwen, Gemma, Mistral, DeepSeek, Phi, Granite, Nemotron, GLM, Cohere, OLMo, Falcon, DBRX, Jamba, Yi, InternLM, Baichuan, MiniMax, Kimi, Step, LongCat, Solar, Hermes, Zephyr.
- Modellgroesse != aktive Rechenlast (MoE: Gesamt- vs. aktive Parameter).
- Varianten-Explosion: FP32/FP16/BF16/FP8/INT8/INT4/GPTQ/AWQ/GGUF/EXL2/MLX/ONNX/TensorRT plus Instruct/Base/Chat/Thinking/Tool-Use/Distill/LoRA/Merge.

**Empfehlung:** Modellpool statt Einzelmodell. Klassifikation nach Funktion als Registry-Feld.
**Status:** Uebernommen als KATEGORIE BP.


## R8 - Kostenlose RAG-Kombination
**Datum:** 2026-09-20
**Quelle:** rag-kostenlos-kombination.md

**Kernergebnisse:**
- Empfehlung: Qdrant (self-hosted, Rust) + bge-m3 (Ollama) + Ollama-Chat + LangChain.
- bge-m3: MIT, 1024 Dim, 8K Kontext, 100+ Sprachen - beste Wahl fuer Deutsch.
- nomic-embed-text: 274 MB, 768 Dim, laeuft auf CPU; Praefixe search_query/search_document zwingend.
- Qdrant Cloud Free: 1 GB RAM, ~4 GB Disk, keine Karte.
- Dimensionen muessen exakt zur Collection passen, sonst Upsert-Fehler.

**Empfehlung:** Qdrant als v3.8-Testkandidat neben Chroma. Kein sofortiger Wechsel.
**Status:** Uebernommen als KATEGORIE BQ.


## R9 - Open Wikis
**Datum:** 2026-09-20
**Quelle:** open-wikis-recherche.md

**Kernergebnisse:**
- Self-Hosted Wiki-Software: Wiki.js (AGPL, Node.js, Git-Backend), BookStack (MIT, PHP), MediaWiki (GPL, PHP), Docmost (AGPL), DokuWiki (GPL, dateibasiert), Outline (BSL), XWiki (LGPL), TiddlyWiki (BSD).
- Fuer Solo-Nutzer: Obsidian (empfohlen, local-first, Markdown), TiddlyWiki (portable HTML), DokuWiki (kein DB-Setup), Wiki.js (Git-Sync).
- Datenschutz/Backup: bei lokalem Markdown -> Git/Syncthing.

**Empfehlung:** Optionale Ergaenzung zu Obsidian. Wiki.js oder DokuWiki als Self-Hosted-Option in v3.10.
**Status:** Uebernommen als KATEGORIE BX.


## R10 - Nicht-KI-Automatisierungsprogramme Linux
**Datum:** 2026-09-20
**Quelle:** nicht_ki_basierte_automatisierungsprogramme_linux_2026.md

**Kernergebnisse:**
- Zentrale Unterscheidung: KI entscheidet WAS, Nicht-KI fuehrt AUS.
- Kategorien: systemd/cron (Basis), Workflow (Rundeck, Airflow, Temporal, Prefect, Dagster, Argo), Config-Management (Ansible, Puppet, Chef, Salt, CFEngine), IaC (OpenTofu, Terraform), CI/CD (Jenkins, GitLab, Drone, Woodpecker), Monitoring (Prometheus, Nagios, Zabbix, Icinga), HPC (Slurm, HTCondor, OpenPBS), Browser (Playwright, Selenium), GUI (xdotool, ydotool, wmctrl, AutoKey, Actiona, SikuliX), Events (Node-RED, Huginn, inotify).
- Tool Registry als YAML mit ai_required, deterministic, sandboxable.

**Empfehlung:** Deterministische Ausfuehrungsschicht als eigenstaendige Kategorie. KI-Agenten senden strukturierte Auftraege, Tools werden nicht improvisiert.
**Status:** Uebernommen als KATEGORIE BW.


## R11 - Lokale KI-Agenten mit vorgefertigten Modellen
**Datum:** 2026-09-20
**Quelle:** lokale_ki_agenten_vorgefertigte_modelle_recherche_2026-09-20.md

**Kernergebnisse:**
- Technisch bestaetigt: Lokale LLMs + Agent + Tools + Memory sind real (Hermes Agent, Open WebUI, Agent Zero, Ollama).
- "Lokal" != "offline" != "DSGVO-konform". Drei Stufen: local-first, offline, air-gapped.
- Autonomie-Level 0-4 (Antwort -> Tool-Auswahl -> Multi-Tool -> Fehlerkorrektur -> langfristige Planung).
- Spezialisierung ohne Fine-Tuning: Modell + Prompt + Tools + Knowledge + Memory + Workflow + Validator.
- Sicherheit: LLM -> Tool-Gateway -> Policy -> Sandbox -> Tool (nicht direkt).

**Empfehlung:** Bestaetigt bestehende Architektur. Intelligence, Memory und Tool-Ausfuehrung getrennt halten.
**Status:** Bereits konzeptionell abgedeckt (AY, AZ, BA).


## R12 - Linux-Automatisierungsprogramme (erweitert)
**Datum:** 2026-09-20
**Quelle:** linux_automatisierungsprogramme_und_anwendungen_2026.md

**Kernergebnisse:**
- Linux hat einen grossen Baukasten aus Automatisierungsebenen: Kernel/Shell, Zeit/Event, Datei, API, Browser, Desktop/GUI, Workflow, System/Server, verteilte Ausfuehrung, KI.
- Wayland vs X11: xdotool/wmctrl/AutoKey sind X11-orientiert; ydotool/dotool fuer Wayland.
- Tool-Kandidaten (Prioritaet hoch): systemd, Python, Bash, Playwright, Ansible, Rundeck, Temporal, Airflow, GNU Parallel, inotify, rsync, Slurm.
- Mittlere Prioritaet: Node-RED, Huginn, Selenium, Prometheus, Alertmanager, ydotool, SikuliX.

**Empfehlung:** Deterministische Werkzeuge als Capabilities registrieren, nicht fest verdrahten. Tool-Auswahl selbst optimierbar (Erfahrungsdaten).
**Status:** Ergaenzt KATEGORIE BW.


## R13 - Kostenlose HPC-Rechenzentren
**Datum:** 2026-09-20
**Quelle:** kostenlose_hpc_rechenzentren_2026-09-20.md

**Kernergebnisse:**
- Kostenlose HPC-Zugaenge: EuroHPC (Extreme Scale Call, Deadline 26.10.2026), NHR, JSC Test Projects, HLRS Stuttgart, bwUniCluster (BaWue), CSCS (Schweiz, User Lab), ARCHER2 (UK), MetaCentrum (CZ), FENIX, ECMWF.
- Fast alle brauchen institutionelle Anbindung - Ausnahme: JSC Test Projects (Eligible: "Everybody").
- Cloud Free Tiers (Oracle, AWS, Azure) sind KEIN HPC.
- Volunteer Computing (BOINC, Folding@home) ist KEIN HPC-Clusterzugang.
- Empfohlene Architektur: lokale Intelligenz + lokaler Memory + externer Compute + lokale Validierung.

**Empfehlung:** HPC Capability Database mit 30 Feldern pro Anbieter. JSC vorab pruefen.
**Status:** Uebernommen als KATEGORIE BT. JSC-Recherche offen.


## R14 - Kognitive Verarbeitungssoftware und Cloud-Kognitionszentren
**Datum:** 2026-09-20
**Quelle:** kognitive_verarbeitungssoftware_und_cloud_kognitionszentren_2026.md

**Kernergebnisse:**
- Kognitive Architekturen: Soar (BSD, regelbasiert), OpenCog/Hyperon (MeTTa, AtomSpace), ACT-R, LIDA, CLARION, Cognitiv (LLM-nah, MIT).
- Cloud-Kognitionszentren: Google AI Studio (Gemini), IBM watsonx (300k Token/Monat + 20 Compute Hours), HF Inference, GroqCloud, Cloudflare Workers AI, Cerebras, Lightning AI (80 free GPU-Stunden), Colab, Kaggle (30h/Woche T4), AWS, Azure.
- Konzept "Cognitive Router": Task -> Cognitive Task Analyzer -> Cognitive Router -> Processing Pool (FAST_REASONING, GENERAL_REASONING, SYMBOLIC, GPU_COMPUTE, SPECIALIZED).
- Optimierungseinheit wird der gesamte Verarbeitungspfad, nicht nur ein Modell.

**Empfehlung:** Cognitive Router als v3.9-Schicht. Soar/Hyperon als symbolische Spezialisten pruefen.
**Status:** Uebernommen als KATEGORIE BS.


## R15 - Hugging Face optimale Nutzung
**Datum:** 2026-09-20
**Quelle:** huggingface_optimale_nutzung_ai_memory_system.md

**Kernergebnisse:**
- HF als Evidence Layer: Vorselektion, nicht Entscheidung. Lokale Systeme entscheiden anhand eigener Benchmarks + Production Telemetry.
- Programmatische APIs: Models, Model Cards, Eval Results (expand=["evalResults"]), Benchmark Leaderboards (GET /api/datasets/{id}/leaderboard), Datasets, Papers.
- Vier lokale Datenbanken: Model Registry, Evaluation Registry, Agent Performance, Experience Store.
- Agent Genome als Optimierungseinheit: Modell + Prompt + Tools + Memory + RAG + Context + Validation.
- Multi-Objective Optimierung (Quality, Latency, RAM, VRAM, Tool Success, Validation Success).
- Fine-Tuning erst nach Modell-, Prompt-, Tool-, Memory-, RAG- und Workflow-Optimierung.
- Sicherheit: keine automatische Installation unbekannter Modelle, kein trust_remote_code blind ausfuehren.

**Empfehlung:** HF Collector als v3.7.2 (nach Gemma-Test). Vier Datenbanken in experience.db.
**Status:** Uebernommen als KATEGORIE BU. Start nach Gemma-Test.


## R16 - Fertige Wissensdatenbanken
**Datum:** 2026-09-20
**Quelle:** AI_Memory_System_Fertige_Wissensdatenbanken_2_Zyklen.md

**Kernergebnisse:**
- Fertige Datenbestaende: The Stack v2 (3 Mrd. Code-Dateien), Mathlib (formale Mathe, Lean), OpenWebMath (6,3 Mio. Docs / 14,7 Mrd. Token), OpenAlex (Wissensgraph), OpenCitations (Citation Graph), Wikidata (Knowledge Graph), CODATA/NIST (Physik-Konstanten), Materials Project (Materialwissenschaft), Wikilite (Wikipedia-SQLite mit FTS5), Common Crawl (Rohmaterial), HF Datasets.
- Tier-Klassifikation: TIER 0 (formal verifiziert) bis TIER 5 (General Web).
- Hybrid Retrieval: Keyword (FTS/BM25) + Semantic (Vektor) + Structured (SQL/Graph) -> Rank + Fuse.
- Provenance: jeder Eintrag behaelt Quelle, Version, retrieved_at, confidence.
- Autoritative Daten duerfen nicht ueberschrieben werden. Agenten erzeugen Hypothesen, nicht Fakten.

**Empfehlung:** Knowledge Fabric mit 6 Ebenen. The Stack v2 + Mathlib + Wikidata + OpenAlex als v3.8-Kandidaten.
**Status:** Uebernommen als KATEGORIE BV.

## R19 - Ollama Vulkan-Status auf Steam Deck (offen, v3.8)
**Datum:** 2026-09-20

**FAKT:** Ollama 0.34.2 im Distrobox-Container erkennt nur CPU
(`id=cpu` in Startup-Logs, `100% CPU` in `ollama ps`).

**FAKT:** Vulkan-Instanz im Container verfuegbar (1.4.357).
`/dev/dri/card0` und `/dev/dri/renderD128` sichtbar.

**FAKT:** `OLLAMA_VULKAN=1` in Service-Datei zeigt keine Wirkung.
`ollama serve --help` enthaelt keine Vulkan-Option.

**FAKT:** Service-Datei enthaelt `OLLAMA_VULKAN=1` (Stand 2026-09-20).

**HYPOTHESE:** Standard-`ollama`-Paket in Arch hat Vulkan-Backend
nicht kompiliert (GitHub Issue #16647 vermutet).

**HYPOTHESE:** Van Gogh (gfx1033) wird von ROCm nicht unterstuetzt;
Vulkan bleibt einziger GPU-Pfad auf dem Deck.

**Konsequenz:** Alle Benchmarks aus 2026-09-20 sind CPU-only valide.
Relative Rangfolge (qwen schnell, gemma langsam) bleibt stabil auch
mit Vulkan, weil Token-Generierung bandbreiten-limitiert ist.

**Vulkan-Vorteil laut R1:** Prompt-Verarbeitung 3-4x schneller.
Token-Generierung praktisch unveraendert.

**Optionen (Stand 2026-09-20, nicht entschieden):**
- A) `ollama-vulkan`-Paket (Arch, Version 0.33.3, Downgrade-Risiko)
- B) Custom AMD-APU-Image (ghcr.io/rjmalagon/ollama-linux-amd-apu,
     Existenz und Wartungsstand nicht verifiziert)
- C) CPU-only akzeptieren (aktueller Stand)
- D) A+B kombiniert (Image + Paket)

**Empfehlung:** C vorerst. Umbau auf v3.8 verschieben, wenn RAG
implementiert wird (dort ist der Vulkan-Vorteil direkt messbar).
Vor Umbau: 1,5 Std Recherche + Test, Risiko Downgrade.

**Status:** OFFEN, nicht dringend, v3.8-Kandidat.

---

## R22 - MoE-Architektur und dynamisches Laden

**Datum:** 2026-09-29
**Quelle:** Recherche-Session, mehrere Modelle

**KERNERKENNTNIS:**
MoE spart Rechnen, nicht Speicher. Nur die aktiven Experten
rechnen pro Token. Der Rest muss nicht resident sein, wenn
man Streaming nutzt.

**FAKTEN:**
- Qwen3-30B-A3B: 30B gesamt, 3B aktiv (10,8 %)
- 3B aktiv bei Q4 = ~2 GB
- Rest kann auf NVMe liegen
- Resident Fraction != Activated Fraction

**PR #25294 (llama.cpp):**
- Stream MoE routed experts from disk
- Flags: --moe-stream, --moe-stream-cache, --moe-stream-io-threads, --moe-stream-direct
- Status: OFFEN, nicht gemergt
- Validierung: CPU, Metal, CUDA. Nicht Vulkan

**CachyLLama:**
- Eigener Fork, mmap + madvise + mincore
- Flags: --moe-expert-residency, --moe-resident-per-layer, --moe-prewarm-top-k
- gfx103X/Van Gogh als Ziel genannt
- Residency-Test auf 7840U/780M, nicht gfx1033

**PR #27861 (csantiago78):**
- GPU-resident LRU Cache fuer host-offloaded Experts
- Branch: moe-expert-cache, Commit bccbacdb
- Vulkan-Test auf 2x RTX 3090

**Kernproblem Vulkan/AMD:**
- set_tensor_async() auf RADV war effektiv synchron
- Ohne SAM/direct-write: GPU-MoE 9 tok/s vs CPU-MoE 28 tok/s
- Mit SAM + GPU-Cache: 35,1 tok/s (RX 9070 XT)

**Status:** Vulkan-Streaming auf gfx1033 ungetestet.
**Naechster Schritt:** CachyLLama bauen und testen.

---

## R23 - Hardware-OC Steam Deck

**Datum:** 2026-09-29
**Quelle:** Recherche-Session, mehrere Modelle

**KERNERKENNTNIS:**
RAM-Bandbreite wichtiger als GPU-OC. UV wichtiger als OC.
Sustained Frequency wichtiger als Peak.

**FAKTEN:**
- Van Gogh: 4x Zen2, RDNA2 8 CU, 16 GB LPDDR5 UMA
- RAM-Bandbreite: 88 GB/s (LCD), 102,4 GB/s (OLED)
- GTT-Standard: 8 GB, erweiterbar auf 10 GB via GRUB
- TDP: 4-15 W, OC auf ~23 W moeglich
- RAM-OC (LCD): 5500 -> 6400 MT/s, +16 % Bandbreite

**REIHENFOLGE:**
1. UV (Undervolting)
2. RAM-OC (nur LCD)
3. Cooling
4. TDP
5. GPU-OC
6. CPU-OC

**THERMIK:**
- Eine Zone (thermal_zone0)
- Luefterlos bei <60 C
- Deck im Dock -> bessere Kuehlung als handheld

**Metriken:**
- Tokens/s / Watt
- Tokens/Joule
- Sustained Frequency

**Status:** Nicht durchgefuehrt. GRUB-Timeout auf 5 Sek gesetzt.

---

## R24 - CachyLLama Build-Details

**Datum:** 2026-09-30
**Quelle:** Recherche-Session

**FAKTEN:**
- Repo: github.com/fewtarius/CachyLLama
- Eigener Fork, kein #25294-basiert
- mmap + madvise(MADV_WILLNEED/MADV_COLD) + mincore()
- Builder nennt gfx103X/Van Gogh als Ziel
- Vulkan-Build: cmake -DGGML_VULKAN=ON (Standard)

**FLAGS:**
- --moe-expert-residency
- --moe-resident-per-layer N (Default: 32)
- --moe-prewarm-top-k N (Default: 16)
- --moe-residency-debug on/off
- --moe-residency-debug-interval N (Default: 64)
- --mmap zwingend erforderlich

**BUILD:**
git clone https://github.com/fewtarius/CachyLLama.git
cd CachyLLama
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=ON -DGGML_NATIVE=OFF
cmake --build build -j4

**STATUS:** Noch nicht gebaut. Kein gfx1033-Residency-Test gefunden.

---

## R25 - PR #27861 GPU Expert Cache

**Datum:** 2026-09-30
**Quelle:** Recherche-Session

**FAKTEN:**
- PR: ggml-org/llama.cpp/pull/27861
- Branch: csantiago78:moe-expert-cache
- Commit: bccbacdb8945680f1cfc7e6bffd1e59014705750
- Veroeffentlicht: 28.08.2026
- Status: OFFEN, nicht gemergt

**FOLLOW-UP-PATCHES:**
- 08930e7f344890faf693a65dab58532e82d5074a (Cherry-Pick)
- Weitere Patches fuer MSVC, Small-Batch/MTP, Cache-Staging

**FLAGS:**
- --moe-expert-cache N
- --moe-expert-cache-inserts N
- --n-cpu-moe N

**KERNBEFUND (aus #20757):**
- set_tensor_async() auf AMD/RADV war effektiv synchron
- Staging-Buffer + ggml_vk_synchronize() = Blockade
- Mit SAM/direct-write: 27,9 -> 35,1 tok/s (RX 9070 XT)
- Ohne SAM: GPU-MoE-Cache 9 tok/s vs CPU-MoE 28 tok/s

**VULKAN-TEST:**
- Validiert auf 2x RTX 3090 (nicht gfx1033)
- Werte: ~39,55 - 44,01 tok/s bei ~3500 tokens

**BUILD:**
git fetch origin pull/27861/head:pr-27861
git checkout pr-27861
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=ON -DGGML_NATIVE=OFF
cmake --build build -j4

**STATUS:** Kein gfx1033-Test. Hauptproblem: Transferpfad.

---

## R26 - Vulkan-Baseline gfx1033 (praktisch gemessen)

**Datum:** 2026-09-29
**Quelle:** Eigene Messung auf Steam Deck

**FAKTEN:**
- Vulkan laeuft auf gfx1033 mit RADV VANGOGH
- Mesa 25.2.8, ACO, RADV
- Vulkan Instance 1.3.275

**BENCHMARKS (llama-bench, b11256):**

| Modell | Test | Vulkan (ngl=999) | CPU (ngl=0) | Faktor |
|---|---|---|---|---|
| Qwen2.5 1.5B Q4_K_M | pp512 | 362,79 t/s | 318,06 t/s | 1,14x |
| Qwen2.5 1.5B Q4_K_M | tg128 | 53,85 t/s | 23,27 t/s | 2,31x |
| Qwen2.5 7B Q4_K_M | pp512 | 68,97 t/s | 58,78 t/s | 1,17x |
| Qwen2.5 7B Q4_K_M | tg128 | 11,10 t/s | 5,41 t/s | 2,05x |

**LIVE-TEST 7B Q4_K_M:**
- Prompt: 54,8 t/s
- Generation: 13,0 t/s
- Antwortqualitaet: Korrekt (Python Primzahl-Funktion)

**GTT-LIMIT:**
- Standard: 8 GB (8589934592 Bytes)
- 14B Q4_K_M (9 GB) scheitert: "Not enough memory for command submission"
- 14B Q3_K_M noch nicht getestet

**STATUS:** Vulkan-Basis bestaetigt. GTT-Tuning offen.

---

## R27 - Linux-Protokolle (Kernel/Netzwerk/Storage)

**Datum:** 2026-09-30
**Quelle:** Recherche-Session

**KERNERKENNTNIS:**
Moderne Linux-Protokolle umgehen Syscalls und
Kernel-Stack durch Shared-Memory-Ringbuffer und
Kernel-Bypass.

**LOKALE PROTOKOLLE:**
- io_uring: Ringbuffer User/Kernel, kein Syscall pro I/O
- io_uring_cmd: NVMe-Passthrough (Block-Layer umgehen)
- netlink: Kernel-Events an User-Space
- AF_UNIX: lokale Sockets, schneller als TCP-Loopback
- varlink: JSON-basiert, strukturiert, ueber AF_UNIX

**NETZWERK- UND STORAGE:**
- eBPF / XDP: Bytecode im NIC-Treiber, Kernel-Bypass
- RDMA / RoCE: Host-to-Host direkt, CPU-Bypass
- NVMe-oF: Remote-SSD mit NVMe-Latenz
- QUIC: UDP-basiert, TLS 1.3, Multiplexing

**EVOLUTION:**
Klassisch: App -> Syscall -> VFS -> Driver -> Hardware
Jetzt: App -> Shared Ringbuffer -> Hardware
Next: App/GPU -> Peer-to-Peer -> Hardware (Zero-CPU)

**FUER DECK:**
- io_uring: relevant fuer Expert-Streaming (NVMe -> RAM)
- io_uring_cmd: direkter NVMe-Zugriff
- eBPF / bpf_struct_ops: Scheduler, Thermik
- varlink: Daemon-Steuerung

**STATUS:** Recherche. Nicht implementiert.
**Referenz:** TOOL_IDEEN CC

---

**Ende RECHERCHEN.md - Stand 2026-09-30 (R1-R27)**
