# LESSONS_LEARNED – AI Memory System

Stand: 2026-09-19
Zweck: Neue Dev-Instanz soll dieselben Fehler nicht wiederholen.

---

## 1. TOP-10 WIEDERKEHRENDE FEHLER

### 1.1 Backticks zerstören Pastes (häufigster Fehler)

Symptom: Heredoc bricht ab, Python-Code wird als bash interpretiert.
Ursache: Triple-Backticks im Paste.
Lösung:
    - Kein cat << EOF mit Codeblöcken, die ``` enthalten
    - Stattdessen: /tmp/write_x.py schreiben, das die Datei erzeugt
    - Oder: 4-Leerzeichen-Einrückung statt Codefences
    - Oder: chr(96) * 3 als Platzhalter

### 1.2 Heredoc-Anker nicht gefunden

Symptom: "Pattern nicht gefunden" in Patch-Scripts.
Lösung:
    - Vor Patch: grep -n "anchor" file.py
    - Anker 1:1 aus grep-Output übernehmen
    - Nach Patch: grep -n "new_string" file.py

### 1.3 Fehlende Module nach Patch

Symptom: No module named core.experience.migrate.
Ursache: Datei wurde nie geschrieben.
Lösung:
    - Nach jedem Schreiben: ls -la + python -m py_compile
    - wc -l mit erwarteter Zeilenzahl vergleichen

### 1.4 Config ignoriert neue Spalten

Symptom: source bleibt "local" statt "chatgpt".
Lösung: Bei ALTER TABLE alle INSERT/SELECT prüfen.

### 1.5 Modell veraltet

Symptom: 429/404 bei gemini-2.5-flash.
Lösung:
    - Modell beim Start proben, nicht hart kodieren
    - Fallback-Kette: 3.5 Lite -> 3.1 Lite -> flash-latest -> 3.6
    - Retry mit retryDelay aus Fehlermeldung

### 1.6 Cloudflare ungueltiges JSON

Symptom: Unterminated string bei chat_structured.
Lösung: STRUCTURED_EXCLUDE = {"cloudflare"} in core/llm.py.

### 1.7 Pydantic dict nicht erlaubt

Symptom: additionalProperties nur in Enterprise.
Lösung: Konkrete Pydantic-Modelle statt dict.

### 1.8 SymPy reservierte Symbole

Symptom: U = R * I -> "Cannot convert complex to float".
Ursache: I ist imaginäre Einheit.
Lösung: Reserved-Map {"I": "I_var", ...} vor sympify.

### 1.9 pytest findet core nicht

Symptom: ModuleNotFoundError: No module named 'core'.
Lösung: conftest.py im Projektroot mit sys.path.insert.

### 1.10 Test-Fixtures zu dünn

Symptom: Test scheitert wegen fehlender Spalte.
Lösung: Bei schema-agnostischen Modulen alle Zielspalten anlegen.


### 1.11 API-Feldname geraten statt verifiziert

Symptom: Upsert-Funktion überspringt alle Einträge kommentarlos.
Kein Fehler, keine Warnung, Rückgabe = 0.

Ursache: Feldname im Code geraten (aus Doku oder eigener Notiz),
aber API liefert andere Struktur.

Konkret (2026-09-20):
- Erwartet: data.task_id
- Tatsächlich: data.dataset.task_id
- Folge: Prüfung "if not (dataset_id and task_id): continue"
  schlug für alle Einträge an.

Lösung:
1. Bei unbekannter API-Struktur: erst Rohantwort zeigen.
2. Feldpfad aus dem JSON ableiten, nicht aus Doku/Notiz.
3. Test-Skript: EINEN Datensatz durch komplette Pipeline schicken,
   an jeder Stufe die Zwischensumme ausgeben.
4. Bei Upsert-Funktionen: Rückgabewert (count) explizit prüfen.

Regel: Eine plausible Annahme über eine API-Struktur ist kein Fakt.

Zusatz: Bei Edit via nano-Suchen-Ersetzen: nach dem Edit
grep auf alte Variable - Vertipper (dsx statt ds) sofort finden.

---

### 10. Unabhängige Operationen bündeln

Symptom: Viele nano + Paste + python-Runden für eine Aufgabe.
Ursache: Sequenzielles Vorgehen ohne Prüfung auf Unabhängigkeit.
Lösung:
    - Schreiben bündeln: ein Batch-Writer für alle unabhängigen Dateien.
    - Verifizieren trennen: py_compile + Test sequenziell,
      Fehlerisolierung bleibt erhalten.
    - Ausgabe pro Datei (Pfad, Zeilenzahl, SHA) zur Diagnose.
    - Kein Heredoc mit Codefences (Lesson 1.1).
Regel: Parallelisieren wo unabhängig, isolieren wo abhängig.

Beispiel 2026-09-20:
    - 3 Writer-Skripte (R7-R16, BP-BX, 5p+5q) -> 3 nano-Runden.
      Besser: ein Batch-Writer mit allen 3 Datei-Aenderungen.
    - 5 RTC-Wake-Dateien korrekt als 1 Writer gebuendelt.
    - Reads (grep, ls, cat, wc) einzeln getippt.
      Besser: { cmd1; echo; cmd2; } 2>&1 | tee /tmp/diag.log

Was NICHT in einen Batch gehoert:
    - Schema-Aenderungen (Rollback komplexer)
    - Systemd-Unit-Installation (sudo, persistente Wirkung)
    - Container-Aktionen (getrennte Umgebung)
    - Modell-Loads parallel (RAM-Konkurrenz)

Verhaeltnis zu Lesson 9:
    Kein Widerspruch. Saubere Parallelisierung ist sorgfaeltiger
    als wiederholte Einzelrunden. Sorgfalt gewinnt nur, wenn
    Parallelisierung Fehlerisolierung gefaehrdet.

## 2. ARCHITEKTUR-LESSONS

- Kein Provider blind vertrauen. Modelle werden abgeschaltet.
- Strukturierte Ausgaben sind fragil. Skip-Liste im Router.
- Quota-Killer ist TPM, nicht RPM (Groq 8k TPM).
- Zeitstempel-Format konsistent halten.
  v3.3–v3.6: TEXT ISO. v3.7: INTEGER Unix-Sekunden.
  Cross-DB: DuckDB to_timestamp(started_at).
- Migrationen idempotent (IF NOT EXISTS + Checksummen).
- Kein rm -rf auf Daten: Hot -> Warm -> Cold.
  ref_count in content_blob immer dekrementieren.
  Fail-safe vor DELETE: Zeilenzahl-Check.

---

## 3. INTEGRATION-LESSONS

- Cross-DB nicht per FK (decisions.db vs experience.db).
- _resolve_collection Fallback: rollup._get_exp_collection -> eigener Client.
- is_legacy als property definieren (nicht Variable).
- Neue Features mit Config-Flag gaten.
- run_daily.day ist UTC YYYY-MM-DD. Frontend konvertiert.

---

## 4. TEST-LESSONS

- Test-Reihenfolge: Unit -> Integration -> Real.
- Nur Collection experience_test in tmp_path berühren.
- pytest.importorskip("duckdb") / ("chromadb") für optionale Deps.
- Modul-Skip korrekt: pytest.skip("...", allow_module_level=True).
- Idempotenz testen (zweiter Lauf -> skip).
- Dry-Run testen (DB-Stand vor == nach).

---

## 5. COMMUNICATION-LESSONS

- Max ~100 Zeilen pro Terminal-Paste.
- Bei längeren Blöcken: Datei-basiert via Python-Script.
- Heredoc mit einem Wort (kein Punkt im Marker).
- Nach jedem Patch: py_compile + grep + Test.
- Vor Patch: cp file.py file.py.bak.

---

## 6. KONKRETE BEFEHLE

Datei sicher schreiben:

    # /tmp/write_x.py anlegen
    from pathlib import Path
    content = """...code..."""
    p = Path.home() / "AI_Memory_System" / "core" / "x.py"
    p.write_text(content, "utf-8")
    print("OK:", p, len(content), "Zeichen")

    # Ausführen
    python /tmp/write_x.py && rm /tmp/write_x.py

Anker-Patch:

    # grep -n "anchor" file.py zur Prüfung
    from pathlib import Path
    src = Path("file.py").read_text()
    assert "old_string" in src, "Anker fehlt"
    src = src.replace(old, new, 1)
    Path("file.py").write_text(src)

Modell-Probe:

    from google import genai
    client = genai.Client()
    for m in ["gemini-3.5-flash-lite", "gemini-3.1-flash-lite"]:
        try:
            client.models.generate_content(model=m, contents="OK")
            print(m, "OK")
        except Exception as e:
            print(m, "FAIL:", str(e)[:80])

---

## 7. BESTEHENDE SCHUTZMECHANISMEN (nicht entfernen)

| Mechanismus | Datei |
|---|---|
| Retry-Backoff bei 429 | core/llm.py |
| STRUCTURED_EXCLUDE | core/llm.py |
| Pydantic-Schemas statt dict | core/agent.py |
| Sandbox mit rlimit | core/sandbox.py |
| Pfad-Traversal-Schutz | core/tools.py |
| Idempotente Migrationen | core/experience/migrate.py |
| ref_count-Dekrement | core/experience/retention.py |
| Verify vor DELETE | core/experience/retention.py |
| Feature-Flag Few-Shots | config.json + core/agent.py |
| conftest.py sys.path | conftest.py |

---

## 8. CHECKLISTE VOR JEDEM PATCH

1. grep -n "anchor" file.py
2. cp file.py file.py.bak
3. Patch via Python-Script (nicht Heredoc mit Codefences)
4. python -m py_compile file.py
5. grep -n "new_string" file.py
6. pytest tests/test_x.py -q
7. Bei OK: file.py.bak entfernen
8. Bei Fehler: file.py.bak zurückspielen

---

## 9. KONFLIKT-REGEL

Bei Konflikt zwischen Effizienz und Sorgfalt:
Sorgfalt gewinnt.

---

## Lesson 11 - Qualitaetsfilter fuer Recherchen

**Datum:** 2026-09-30
**Kontext:** Einige KIs machen es sich zu leicht und
sagen von vornherein "geht nicht". Andere suchen
konstruktive Wege.

**Regel:**
Konstruktive Recherchen bevorzugen.

**Kriterien:**
1. Sucht Wege, nicht Ausreden
2. Liefert Commits, Flags, Zahlen
3. Hat belegte Tests
4. Ist uebertragbar auf unsere Hardware

**Negativ-Befunde sind erlaubt, aber:**
- Muessen belegt sein
- Muessen Quellen haben
- Duerrfen nicht den Weg blockieren

**Anwendung:**
- Cluster-Bildung beim Aufraeumen
- KI-Delegation: konstruktive Quellen zuerst
- Bei Widerspruch: Datei entscheidet

**Beispiel:**
- Negativ: "Vulkan + Streaming ist ungetestet auf gfx1033"
- Positiv: "CachyLLama hat gfx103X als Ziel. Konkrete Flags: ..."

---

## Lesson 12 - Backward Programming

**Datum:** 2026-09-30
**Kontext:** Forward-Denken ("Was kann ich damit
bauen?") endet oft in "geht nicht". Backward-Denken
("Was brauche ich fuer dieses Ziel?") findet Wege.

**Regel:**
Ziel zuerst, dann rueckwaerts zum Weg.

**Frage-Muster:**

| Forward (vermeiden) | Backward (nutzen) |
|---|---|
| "Geht das?" | "Was braucht es dafuer?" |
| "Das ist ungetestet." | "Das ist der Weg dorthin." |
| "Zu komplex." | "Schritt 1: was zuerst?" |

**Bei Recherchephasen:**
- Nicht sofort bewerten
- Recherche abschliessen lassen
- Dann gemeinsam konsolidieren

**Bei Widerspruch:**
Datei entscheidet, nicht KI.

**Beispiel aus der Session:**
- Forward: "30B passt nicht in 16 GB RAM"
- Backward: "Damit 30B laeuft, brauchen wir 2 GB
  RAM fuer aktive Experten + NVMe-Streaming"
- Ergebnis: 30B ist moeglich.

---

## Lesson 13 - Reverse Engineering und kontrollierte Weiterentwicklung

**Datum:** 2026-10-02
**Kontext:** Neue verbindliche Arbeitsweise ab Session 02.10.2026.
**Beziehung zu Lesson 12:** Lesson 12 = Ziel zuerst, rueckwaerts zum Weg.
Lesson 13 = Auf diesem Weg wird Bestehendes zuerst analysiert und
dokumentiert, bevor es veraendert wird.

**Kernregel:**
Bestehende Systeme zuerst erfassen, analysieren und dokumentieren -
dann kontrolliert weiterentwickeln. Nie gruenwuechig bauen, was
schon existiert.

**Fuenf Schritte:**
1. Bestandsaufnahme: Komponenten, Konfigurationen, Abhaengigkeiten
   erfassen und in einer Datei festhalten (z.B. SYSTEM_INVENTAR.md).
2. Struktur und Datenfluesse dokumentieren, bevor etwas geaendert wird.
3. Aenderungen klein und kontrolliert: Abhaengigkeiten vorher pruefen,
   Rollback vorher definieren.
4. Nach jeder Aenderung: Funktionstest UND Regressionstest
   (bestehende Funktionen muessen unberuehrt bleiben).
5. Dokumentation aktualisieren, sobald die Aenderung steht.

**Anwendung im Projekt:**
- CachyOS-Migration: erst SYSTEM_INVENTAR.md (Partitionen, Kernel-
  Parameter, Mounts, Backup-Verifikation), dann partitionieren,
  dann testen, dann Doku-Update.
- Neue Features (DB-Stack, Web-UI, Provider): erst existierenden
  Code und Schnittstellen analysieren, dann integrieren.
- Weiterhin gueltig: Datei entscheidet, nicht KI.

---


---

## Lesson 14 - JMS567-Bridge und EXT4-Mountoptionen

**Datum:** 2026-10-02
**Kontext:** Die externe ICYBOX-SSD mit JMicron JMS567 blieb am Steam Deck zunächst
bei ungefaehr 24 MB/s. Ein Kernel-Log zeigte eine auffaellige `optimal_io_size`-Warnung.
Eine vorgeschlagene udev-Regel gegen diesen Wert brachte keine Verbesserung.

**Verifizierter Workaround:**
Die SSD mit `nombcache,noatime,data=writeback` mounten. Nach manuellem Mount
stieg der 1-GB-Schreibtest von etwa 24 MB/s auf 224-248 MB/s. Nach fstab-Eintrag
und Kaltstart wurden 355 MB/s Schreiben und etwa 375 MB/s Lesen gemessen.

**Dauerhafter CachyOS-Eintrag (UUID muss zum Geraet passen):**
UUID=41606b00-4e47-4d83-b49d-5b0e16191fb2 /mnt/ext256 ext4 defaults,nombcache,noatime,data=writeback,nofail,x-systemd.device-timeout=5 0 2

**Wichtige Einordnung:**
- Der Workaround ist empirisch verifiziert; die genaue Kernel-Ursache ist nicht abschliessend bewiesen.
- Die udev-Regel fuer `optimal_io_size` allein war wirkungslos.
- `data=writeback` schwaecht die Reihenfolgegarantien fuer Dateidaten bei Stromausfall.
  Laufwerk sicher aushängen; wichtige Daten separat sichern.
- Nicht pauschal auf andere ext4-Laufwerke anwenden: Die Seagate-HDD mit JMS578
  erreichte ohne diesen Fix etwa 132 MB/s nachhaltig.

**Regel:** USB-Geraet, Bridge-Chip, Mountoptionen und Messmethode getrennt erfassen.
`lsusb -t` zeigt Linkrate, beweist aber allein keinen Anwendungsdurchsatz.

---

## Lesson 15 - Ollama Vulkan auf Van Gogh

**Datum:** 2026-10-02
**Kontext:** CachyOS auf Steam Deck LCD, AMD Van Gogh / RDNA2 (RADV VANGOGH).
ROCm ist fuer diesen GPU-Pfad nicht der verwendete Backend-Weg; Ollama Vulkan ist
installiert.

**Verifizierter Aufbau:**
- Paket: `ollama-vulkan`
- Ollama: Version 0.35.0, systemd-Systemdienst
- Systemd-Drop-in: `/etc/systemd/system/ollama.service.d/override.conf`
- Dienst laeuft als `dominikf`, `OLLAMA_MODELS=/home/dominikf/.ollama/models`
- `ProtectHome=no`, da der Dienst auf GGUF-Dateien im Home zugreifen muss
- `OLLAMA_IGPU_ENABLE=1` erforderlich: ohne diese Variable erkannte Ollama die Vulkan-iGPU,
  verwarf sie aber als integrierte GPU
- Ollama bindet standardmaessig an `127.0.0.1:11434`; nicht ohne Authentifizierung ins Netz oeffnen

**Modellimport statt erneutem Download:**
Ein vorhandenes GGUF kann per Modelfile registriert werden:
FROM /home/dominikf/models/qwen2.5-1.5b-instruct-q4_k_m.gguf

Danach: `ollama create qwen2.5:1.5b-local -f Modelfile-qwen15b`

**Verifikation:** `ollama run qwen2.5:1.5b-local` antwortete erfolgreich.
`ollama ps` zeigte `100% GPU`; Journal identifizierte Vulkan / RADV VANGOGH,
8.2 GiB Gesamt-VRAM und 7.6 GiB verfuegbar.

**Regel:** `ollama list` ist leer, auch wenn GGUF-Dateien in `~/models` liegen.
Ollama importiert vorhandene Dateien nicht automatisch. Erst den lokalen Modellbestand
pruefen, dann bei Bedarf mit Modelfile registrieren; nicht blind doppelt herunterladen.

---

## Lesson 16 - Autonomie-Architektur (Executor, Guard, Snapshot, Worker)

**Datum:** 2026-10-02
**Kontext:** Autonome Ausfuehrung soll auf CachyOS nativ laufen. Sandbox
ist fuer autonome Installationen hinderlich. Rollback statt Isolation.

**Kernregel:**
Rollback schlaegt Isolation. Nicht verhindern - ruecknehmbar machen.

**Vier Sicherheitsebenen:**

1. **Snapper pre/post** vor jedem Befehl (root + home). Fail-closed.
2. **Command-Blacklist** (core/exec_guard.py): nur katastrophale
   Befehle blockieren, nicht alles. Konkret: rekursives rm auf
   Systempfade, mkfs, dd of=Blockgeraet, wipefs, sgdisk --zap-all,
   parted/fdisk im Non-List-Modus, shred, badblocks -w, mv von
   Systemverzeichnissen, chmod/chown rekursiv auf Root, fork bombs.
3. **Lokale Bindung**: Web-UI nur 127.0.0.1, leerer auth_token.
4. **Fail-Safe Recovery**: verwaiste running-Jobs werden auf error
   gesetzt, nicht automatisch wiederholt. Manuelle Pruefung.

**Was NICHT blockieren:**
- rm eines einzelnen Ordners im Projekt
- sudo pacman, systemctl, git
- Alles innerhalb /home/<user>/*

**Warum keine Sandbox:**
- rlimit war schwach: kein Namespace, keine Netz-Isolation.
- Ziel ist autonome Systeminstallation. Sandbox verhindert das.
- Snapper deckt Rollback ab, auch wenn der Agent Systempakete installiert.

**Job-Architektur (persistent):**
- agent_job-Tabelle in experience.db
- Status: queued, running, ok, error, cancelled
- Worker-Lease per worker_id + heartbeat_at
- Blob-Referenzen (SHA256) statt Klartext in DB

**Lesson 16a - Persistenz in Blobs, nicht in Text:**
Job-Eingaben und -Ergebnisse in content_blob speichern,
nicht als Klartext in agent_job. Nutzt vorhandenen SHA256-Dedup-Store.
Haelt die Job-Tabelle schlank und konsistent mit agent_run/agent_span.

---

## Lesson 17 - uv als Paketmanager

**Datum:** 2026-10-02
**Kontext:** Migration SteamOS -> CachyOS. Alte venv (pip+python3.13)
nicht nutzbar auf python3.14.

**Regel:**
Bei Neuaufbau: uv statt pip+venv, wenn verfuegbar.

**Vorteile:**
- 108 Pakete in 472 ms (statt Minuten)
- pyproject.toml + uv.lock: reproduzierbar
- Managed Python: kein Versionsdrama
- uv run: automatische venv-Aktivierung

**Migration alter Projekte:**
- requirements.txt als Basis
- Imports der Projektdateien pruefen: ```grep -rhoE '^(import|from)\s+\w+'```
- Echte Abhaengigkeiten identifizieren (nicht raten)
- requirements.txt ist oft unvollstaendig: direkte Imports
  aus Code ableiten, nicht aus Doku.

**CachyOS-Detail:**
- CachyOS-Repo hat uv aktuell (0.12.22)
- Installation: ```sudo pacman -S uv```
- Snapper zieht pre/post-Snapshots bei pacman-Aufruf

---

Ende LESSONS_LEARNED.md
