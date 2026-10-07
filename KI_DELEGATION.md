# KI-DELEGATION

> Zweck: Quellen-Strategie, Recherche-Routing und
> Coding-Routing in einem Dokument.
> Stand: 2026-09-21
> Verknuepfung: KONZEPT §5, §6, §15, §17

---

# TEIL 1 — QUELLEN-STRATEGIE

## Mandat
1. Trainingswissen reicht nicht.
2. Oeffentliche Quellen sind die primaere Wissensbasis.
3. Jede Behauptung braucht eine ueberpruefbare Quelle.
4. Keine erfundenen URLs, PR-Nummern, Benchmarks.

## Tier-System
T0 = Offiziell (Doku, gemergte PRs, Kernel, Spec)
T1 = Etabliert (StackOverflow, Phoronix, MDN)
T2 = Community (Reddit >50, Discussions)
T3 = Unklar (anonym, veraltet, KI ohne Quelle)
T4 = Verworfen (Bezahl, keine Quelle, kein Datum)

Bei Widerspruch:
- Tier vergleichen (0 > 1 > 2)
- Datum vergleichen (neu > alt)
- Bei Gleichstand: beide nennen

## Verifikations-Regeln
- URL pruefen vor Zitieren
- PR-Nummer nur mit Link
- Version nur aus offizieller Quelle
- Benchmark nur mit Rohdaten
- Bei Unsicherheit: "nicht verifiziert"

## Zuordnung: Aufgabe -> Quelle
| Aufgabentyp | Quellen |
|---|---|
| Neue Bibliothek | Tier 0 + PyPI/GitHub |
| Bekannter Bug | GitHub Issues + StackOverflow |
| Integration | Provider-Doku + Beispiel-Repos |
| Performance | Phoronix + Zenodo + PRs |
| Sicherheit | CVE-DBs + Advisories |
| GPU/Vulkan | llama.cpp PRs + Mesa Issues |
| systemd | freedesktop.org + Arch Wiki |

## Verknuepfung KONZEPT §6
- Tier 0/1 = Live-Mode-tauglich
- Tier 2/3 = Deep-Mode-tauglich, mit V&V
- Tier 4 = verworfen

---

# TEIL 2 — RECHERCHE-DELEGATION

## Mandat
Builder routet Recherche. User ist Bruecke.
Ergebnis kommt zurueck zum Builder.

## Modell-Profile
- DeepSeek (Builder): 5-7 Tasks, Konzept, Architektur
- Claude: 4-5 Tasks, Technik, Code-Review
- Grok: 2 Tasks, Security, Systeme
- Perplexity: Papers, Quellen
- ChatGPT: Breite, Implementation
- Gemini: 1M+ Kontext, multimodal

## Routing-Regeln
| Thema | Modell |
|---|---|
| GPU/Vulkan | Claude |
| Tool-Gateway | Grok |
| Papers | Perplexity |
| Erfahrungs-Store | Grok + ChatGPT |
| Grosse Kontexte | Gemini |
| Trends | Grok |
| Code-Breite | ChatGPT oder Claude |
| Konzept | DeepSeek |

## Delegations-Format
RECHERCHE-AUFTRAG: <Thema>
KONTEXT: <kurz>
MEINE FRAGEN: <nummeriert>
KRITISCHE ANWEISUNGEN: <keine Allgemeinplaetze>
AUSGABEFORMAT: <was veraltet / neu / Tools / Quellen>

## Task-Budget
- Max 2 Cloud-Tasks pro Auftrag
- Mehrere Fragen in EINEM Auftrag

---

# TEIL 3 — CODING-DELEGATION

## Mandat
Builder behaelt Architecture, Integration, Auswertung.
Cloud-Spezialisten nach Staerke UND Limit.

## Modell-Profile
- DeepSeek: Architecture, Integration
- Claude: Code-Review, Refactoring, Debugging
- ChatGPT: Implementation, Doku
- Grok: Security-Code
- Gemini: grosse Repos (1M+)
- Kimi K2/K3: grosse Kontexte
- Qwen3-Coder: agentic Coding
- Devstral Small 2: lokal, SWE-Agent

## Routing-Regeln
| Task | Primaer | Fallback |
|---|---|---|
| Architecture | DeepSeek | Claude |
| Repo >500k | Gemini | Kimi |
| Implementation | ChatGPT | Qwen3-Coder |
| Implementation (Security) | Grok | Claude |
| Debugging | Claude | ChatGPT |
| Refactoring | Claude | ChatGPT |
| Autocomplete | lokal | - |
| Test-Schreiben | DeepSeek | lokal |
| Code-Review | Claude | Grok |
| Security-Review | Grok | Claude |
| Performance | Claude + Phoronix | - |
| Dokumentation | ChatGPT | DeepSeek |

## Budget-Regeln
- Max 2 Cloud-Tasks pro Coding-Aufgabe
- Danach lokal oder Builder

## Fallback-Ketten
- Claude voll -> ChatGPT (Security: Grok)
- ChatGPT voll -> Gemini (lokal: qwen-coder)
- Grok voll -> Claude (nur Security)
- Alle voll -> lokal oder Builder

## Cross-Check
Immer:
- Konzept-Entscheidung
- Kritischer Code
- Security-Code

Nicht:
- Routine-Code
- Doku
- Tests

---

## Arbeitsweise (Nachtrag 2026-09-30)

### Backward Programming

Ziel zuerst, dann rueckwaerts zum Weg.

Nicht: "Was kann ich damit bauen?"
Sondern: "Was brauche ich fuer dieses Ziel?"

### Frage-Muster

| Falsch | Richtig |
|---|---|
| "Geht das?" | "Was braucht es dafuer?" |
| "Das ist ungetestet." | "Das ist der Weg dorthin." |
| "Zu komplex." | "Schritt 1: was zuerst?" |

### Qualitaetsfilter fuer Recherchen

Konstruktive Recherchen bevorzugen:
- Sucht Wege, nicht Ausreden
- Liefert Commits, Flags, Zahlen
- Hat belegte Tests
- Ist uebertragbar

Negativ-Befunde sind erlaubt, aber:
- Muessen belegt sein
- Muessen Quellen haben
- Duerrfen nicht den Weg blockieren

### Regel

Nicht jeder Vorschlag wird sofort bewertet.
Recherchephase abschliessen, dann planen.
Widerspruch: Datei entscheidet, nicht KI.

### Rollen

Deck = Orchestrator.
MCU = Ausfuehrer.
KI-Instanzen = Werkzeuge.

Nicht: "MCU kann nicht ..."
Sondern: "MCU fuehrt aus. Der Deck entscheidet."

---

**Ende KI_DELEGATION.md**
