# AI Memory System v3.2 (Full Stack)

Multi-Agenten-System mit:
- Lead-Agent (Gemini Function Calling)
- Sandbox-Ausführung KI-generierten Codes (rlimit/firejail/bwrap)
- SQLite-Memory + Architektur-Entscheidungen (ADR)
- ChromaDB + Gemini Embeddings für semantisches Memory (RAG)
- FastAPI Web-UI mit Multi-Session
- Docker-Support
- Backup/Restore
- Steam-Deck-Shortcuts

## Setup (bereits erledigt durch bootstrap_v3.2_part1+2)

    nano .env                   # GEMINI_API_KEY eintragen

## Start – CLI

    ./start_cli.sh

Befehle: exit | sync | index | decisions | reindex | rag-on | rag-off

## Start – Web-UI

    ./start_web.sh
    # → http://localhost:8000

## Backup / Restore

    ./backup.sh                 # ohne .env
    ./backup.sh --with-env      # inkl. API-Key
    ./restore.sh backups/ai-memory-YYYYMMDD-HHMMSS.tar.zst

## Steam-Deck-Shortcuts

    ./steam_deck/install_shortcuts.sh

## Docker

    docker compose up -d --build
    docker compose logs -f ai-memory
    docker compose run --rm ai-memory-cli
    docker compose down

## Tests

    source venv/bin/activate
    pytest -q

## Sicherheit

- API-Key nur in .env
- Sandbox mit CPU/RAM/File/Prozess-Limits
- Pfad-Traversal-Schutz
- Git-Autocommit
- Web-Auth optional via Bearer-Token
- Backup schließt .env standardmäßig aus

---

## 📓 Als Obsidian-Vault öffnen

Das `memory_db/`-Verzeichnis enthält alle ADRs als Markdown-Dateien
(`DEC-*.md`). Du kannst es direkt als Obsidian-Vault öffnen:

1. Obsidian starten
2. **"Open folder as vault"**
3. Pfad wählen: `~/AI_Memory_System/memory_db/`
4. Fertig

**Vorteile:**
- Graph-Ansicht aller Entscheidungen
- Volltextsuche
- Tags (`logging`, `architecture`, ...)
- Wikilinks (falls später aktiviert)

**Dateiformat der ADRs:**

```markdown
# [DEC-20260917-141926-027c] JSON-Lines für Logging

- **Datum:** 2026-09-17
- **Kontext:** Logs sollen strukturiert sein
- **Entscheidung:** JSON-Lines in logs/events.jsonl
- **Begründung:** ...
- **Tags:** logging, architecture, jsonl
```

**Tipp:** Öffne `minimal_index.json` nicht in Obsidian – das ist Rohdaten-Format.

