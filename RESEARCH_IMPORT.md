# RESEARCH_IMPORT

> Zweck: Zielstruktur fuer Recherche-Import.
> Stand: 2026-09-30
> Verknuepfung: KONZEPT §10, Cluster D

---

## 1. Ziel

Recherchen werden in ein einheitliches Format konvertiert,
veredelt, in Schritte zerlegt und in eine Datenbank
eingepflegt.

Original bleibt erhalten (SD/Cloud).

---

## 2. Standard-Format

Markdown + YAML-Frontmatter.

```markdown
---
title: GPU_Vulkan_gfx1033_Status
date: 2026-09-20
category: GPU
tier: 2
source: RECHERCHEN.md
original: R19
status: unverified
import_status: ok
research_type: recursive
---
```

---

## 3. Import-Status

| Status | Bedeutung |
|---|---|
| ok | ohne Auffaelligkeiten |
| warnung | mit Hinweis |
| fehler | fehlgeschlagen |

---

## 4. Titel-Konvention

Format: Kategorie_Thema_Kurzform

Regeln:
- ASCII only
- Unterstrich als Trenner
- Max 60 Zeichen
- Kategorie zuerst
- Kein Datum im Titel

Feste Kategorien:
GPU, CPU, RAM, Modelle, DB, Agenten, Orchestrierung,
Voice, Sicherheit, Cloud, Hardware, Netzwerk, Meta,
Theorie, Sonstiges

---

## 5. Datenbanken (Cluster D, 2026-09-30)

| DB | Zweck | Groesse |
|---|---|---|
| SQLite + sqlite-vec | State, Audit, Metadaten | bis 1 GB |
| LanceDB | Vektor-Index | 10-100 GB |
| DuckDB | Analytik, Parquet, BM25 | 1-10 GB |
| KuzuDB | Graph, Cypher | 1-10 GB |
| chDB | Log-Analyse (optional) | bei Bedarf |

### Eigenschaften

- Alle embedded (kein Server)
- LanceDB: mmap, disk-backed, multimodal
- DuckDB: In-Process, Arrow-Bridge zu LanceDB
- KuzuDB: Disk-Paging, Multi-Hop-Traversal
- ChromaDB wird migriert (v3.8+)

### Tabelle research

| Spalte | Inhalt |
|---|---|
| id | Primary Key |
| title | Kategorie_Thema_Kurzform |
| title_original | Rohtitel |
| category | Kategorie |
| source_file | Pfad |
| source_hash | SHA256 |
| date_created | YYYY-MM-DD |
| tier | 0-4 |
| research_type | recursive/block/mixed/unknown |
| summary | Kurzfassung |
| status | unverified/verified |
| import_status | ok/warnung/fehler |

### Tabelle cycle

Nur bei recursive/mixed.

| Spalte | Inhalt |
|---|---|
| id | Primary Key |
| research_id | FK |
| sequence | 1, 2, 3, ... |
| title | Zyklus-Titel |
| focus | Fokus |
| summary | Ergebnis |
| status | offen/geschlossen |

### Tabelle step

Jede Recherche wird in Schritte zerlegt.

| Spalte | Inhalt |
|---|---|
| id | Primary Key |
| research_id | FK |
| cycle_id | FK (nullable) |
| sequence | Reihenfolge |
| step_type | frage/erkenntnis/synergia/hypothese/pruefung |
| text | Inhalt |
| confidence | 0.0-1.0 |
| reflexion_level | 0-5 |

### Tabelle source

| Spalte | Inhalt |
|---|---|
| id | Primary Key |
| step_id | FK |
| url | Quelle |
| tier | 0-4 |
| date | Datum |
| verified | 0/1 |

### Tabelle validation

| Spalte | Inhalt |
|---|---|
| id | Primary Key |
| step_id | FK |
| check_type | konsistenz/quelle/falsifikation/metamorphic |
| result | bestanden/warnung/fehler |
| details | Freitext |

### Tabelle link

| Spalte | Inhalt |
|---|---|
| id | Primary Key |
| from_step_id | FK |
| to_step_id | FK |
| link_type | verstaerkt/widerspricht/ergaenzt |
| confidence | 0.0-1.0 |

---

## 6. Schritt-fuer-Schritt-Import

Interaktiv. Jeder Schritt wird bestaetigt.

Vier Modi:
- Auto: einfache Recherchen
- Pruefend: Standard
- Tief: wichtige Recherchen
- Manual: kritische Inhalte

Ablauf pro Schritt:
1. System schlaegt vor
2. Nutzer waehlt: Bestaetigen / Aendern / Ablehnen / Ueberspringen
3. Nur bestaetigte Schritte in DB

---

## 7. Zugriff fuer KI-Modelle

### API-Modelle

HTTP-Endpoint (FastAPI):
- GET /research/list
- GET /research/{id}
- GET /research/search
- GET /research/semantic

### Non-API-Modelle

CLI-Export:
- python -m core.research list
- python -m core.research show ID
- python -m core.research search vulkan

Beide greifen auf dieselbe DB zu.

---

## 8. Was neu gebaut wird

| Stueck | Umfang |
|---|---|
| Parser (txt/md/pdf/docx/zip) | mittel |
| Struktur-Erkennung | mittel |
| Step-Extraktion | mittel |
| Interaktiver Import | mittel |
| Titel-Generator | klein |
| HTTP-Endpoints | klein |
| CLI-Export | klein |
| 7 Tabellen + Migration | mittel |

---

## 9. Offen

- Link-Erkennung: automatisch oder manuell?
- Chroma-Migration: wann?
- Cross-Model-Referenzierung: Format?

---

**Ende RESEARCH_IMPORT.md**
