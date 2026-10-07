# 📜 ETHIK & VERANTWORTUNG – AI Memory System

> Selbstverpflichtung für den Betrieb eines autonomen KI-Systems.
> Stand: 2026-09-18

---

## 1. RISIKOKLASSE (EU AI Act)

**Einstufung:** Minimalrisiko (§ 3 Nr. 3 EU AI Act)

Begründung:
- Kein Einsatz in kritischen Bereichen (Medizin, Justiz, Personal)
- Kein Social Scoring
- Kein verdecktes Profiling
- Kein autonomer Handel ohne menschliche Aufsicht
- Persönliches Single-User-System

**Konsequenz:** Keine Konformitätsprüfung nötig. Transparenz trotzdem zugesichert.

---

## 2. TRANSPARENZ-ZUSAGEN

- **Modell-Nennung:** Jede Antwort kann mit Provider/Modell angezeigt werden (`/costs`)
- **Audit-Trail:** Alle Aktionen werden geloggt (`/audit`)
- **Quellen-Tagging:** Importierte Entscheidungen tragen `source=chatgpt/claude/...`
- **Human-in-the-Loop:** Bei kritischen Aktionen (Datei schreiben, ADR speichern) bleibt die Entscheidung beim Nutzer
- **Kein stiller Datenaustausch:** Externe KIs werden ausschließlich manuell via Clipboard-Bridge genutzt

---

## 3. DATENSCHUTZ

- **Local-First:** Alle Daten liegen auf dem eigenen Gerät
- **API-Keys:** Ausschließlich in `.env`, niemals in Git
- **Backup:** Standardmäßig ohne `.env`
- **Free-Tier-Klauseln:** Bekannt und akzeptiert (Gemini Free: Training möglich; Abhilfe via Prepay)
- **Kein KI-Training mit eigenen Daten:** Bei Cloudflare, Groq, Mistral (opt-out), OpenRouter (Free)

---

## 4. AUTONOMIE & KONTROLLE

- **Sandbox:** Jeder generierte Code läuft isoliert (rlimit)
- **Timeout:** Max. 30 Sekunden pro Skript
- **Datei-Limit:** Max. 200 KB
- **Pfad-Schutz:** Kein Ausbrechen aus `workspace/`
- **Stop-Möglichkeit:** Nutzer kann jederzeit `Strg+C` oder Server-Neustart erzwingen

---

## 5. VERANTWORTUNG

**Der Nutzer** ist verantwortlich für:
- Inhaltliche Korrektheit generierter Ergebnisse
- Einhaltung von Lizenzen importierter Inhalte
- Weitergabe von KI-generierten Texten

**Das System** übernimmt keine Haftung für:
- Falsche oder unvollständige Antworten
- Datenverlust (Backups liegen in Nutzerverantwortung)
- Kosten durch aktivierte Paid-Provider (Nutzer verwaltet Keys)

---

## 6. VERBOTENE NUTZUNG

Nicht erlaubt mit diesem System:
- Erzeugung schädlicher Inhalte (Malware, Waffen, Drogen)
- Umgehung von Sicherheitsmechanismen
- Automatisierte Web-UI-Nutzung fremder KIs (ToS-Verstoß)
- Multi-Account-Nutzung zur Limit-Umgehung
- Kommerzieller Vertrieb importierter KI-Inhalte ohne Rechteklärung

---

## 7. ZUKUNFT DER ARBEIT

Dieses System automatisiert Teile der Entwicklungs- und Recherchearbeit.
Bewusster Umgang bedeutet:
- **Keine blinde Delegation** – jede Entscheidung bleibt prüfbar
- **Keine Verdrängung** – Werkzeug zur Erweiterung, nicht zum Ersatz
- **Kontrolle behalten** – Nutzer ist Architekt, KI ist Assistent

---

## 8. SELBSTVERPFLICHTUNG

- Regelmäßige Prüfung des Audit-Logs
- Kosten im Blick behalten (Ziel: 0 € durch Free Tiers)
- Bei Weitergabe: diesen Text mitgeben
- Bei kommerzieller Nutzung: Risikoklasse neu bewerten

---

**Ende ETHIK.md**
