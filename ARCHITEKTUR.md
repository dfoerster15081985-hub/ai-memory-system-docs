# ARCHITEKTUR

> Zweck: Gesamtsystem in 9 Ebenen.
> Stand: 2026-09-30
> Verknuepfung: KONZEPT, TOOL_IDEEN, RESEARCH_IMPORT, RECHERCHEN

---

## Uebersicht

```
Ebene 9 — Arbeitsweise
          Backward Programming, Rollen, Qualitaetsfilter
              |
Ebene 8 — Protokoll
          io_uring, eBPF, AF_UNIX, varlink
              |
Ebene 7 — Persistenz
          SQLite, LanceDB, DuckDB, KuzuDB
              |
Ebene 6 — Routing
          Capability, Co-Activation, Graph
              |
Ebene 5 — Cache
          GPU / RAM / NVMe Hierarchie
              |
Ebene 4 — Modell
          MoE mit Streaming, Capability
              |
Ebene 3 — Compute
          Vulkan, llama.cpp
              |
Ebene 2 — Session
          Game Mode / Desktop Mode
              |
Ebene 1 — Hardware
          Van Gogh APU, 16 GB UMA, NVMe
```

---

## Ebene 1 — Hardware

### Steam Deck

- CPU: 4x Zen2, 8 Threads, 2,4-3,5 GHz
- GPU: RDNA2, 8 CUs, 1,6 GHz
- RAM: 16 GB LPDDR5 Unified (14 GB nutzbar)
- NVMe: PCIe Gen3 x4, 224 GB
- TDP: 4-15 W (bis 23 W mit OC)
- Thermik: Eine Zone, luefterlos bis 60 C

### Edge (zurueckgestellt)

- MCU: RP2350, ESP32-P4, i.MX RT1170
- Externe NVMe: ueber Bridge
- Protokoll: COBS + USB Bulk

**Referenz:** RECHERCHEN R23, TOOL_IDEEN CB

---

## Ebene 2 — Session

### Game Mode vs Desktop

| Modus | RAM frei | Anwendung |
|---|---|---|
| Desktop + KDE | ~10 GB | Bau-Arbeit, Recherche |
| Game Mode | ~11,5 GB | Deep Mode, Batch |

**Plan:** Game Mode wird Standard fuer produktiven Betrieb.

### Trigger

- Manuell (Steam-Button)
- Inaktivitaet (5 Min)
- RTC-Wake (03:00)

**Referenz:** KONZEPT §2, §16

---

## Ebene 3 — Compute

### Vulkan

- llama.cpp mit Vulkan-Backend
- RADV VANGOGH, Mesa 25.2.8
- `-ngl 999` = alle Layer auf GPU

### Benchmarks (R26)

| Modell | Test | Vulkan | CPU | Faktor |
|---|---|---|---|---|
| 1.5B Q4 | pp512 | 362 t/s | 318 t/s | 1,14x |
| 1.5B Q4 | tg128 | 54 t/s | 23 t/s | 2,31x |
| 7B Q4 | pp512 | 69 t/s | 59 t/s | 1,17x |
| 7B Q4 | tg128 | 11 t/s | 5,4 t/s | 2,05x |

**Referenz:** RECHERCHEN R26, Session 29.09

---

## Ebene 4 — Modell

### MoE-Streaming

**Kernprinzip:** Das Modell ist nicht im RAM. Das Modell
ist auf NVMe. Im RAM sind nur die Experten, die gerade
gebraucht werden.

- Aktive Experten im RAM (~2 GB bei Q4)
- Rest auf NVMe
- Streaming-Tools:
  - llama.cpp PR #25294 (offen)
  - CachyLLama (mmap-basiert)
  - PR #27861 (GPU-Cache)

- Aktive Experten im RAM (~2 GB bei Q4)
- Rest auf NVMe
- Streaming-Tools:
  - llama.cpp PR #25294 (offen)
  - CachyLLama (mmap-basiert)
  - PR #27861 (GPU-Cache)

### Kandidaten

| Modell | Quant | Groesse | Aktiv |
|---|---|---|---|
| OLMoE-1B-7B | Q4_K_M | 4,2 GB | 1,3B |
| Qwen1.5-MoE-A2.7B | Q4_K_M | 8,8 GB | 2,7B |
| DeepSeek-V2-Lite | Q4_K_M | 9,7 GB | 2,4B |
| GPT-OSS-20B | Q4_K_M | 11,6 GB | MoE |
| Qwen3-30B-A3B | Q2_K | 11,3 GB | 3,3B |

**Status:** Vulkan-Streaming auf gfx1033 ungetestet.

**Referenz:** RECHERCHEN R22, R24, R25

---

## Ebene 5 — Cache

### Hierarchie

| Level | Wo | Was |
|---|---|---|
| L0 | GPU-VRAM | aktive Experten |
| L1 | RAM hot | predicted |
| L2 | RAM cold | warm |
| L3 | NVMe | alle Experten |

### Cache-Score

Basis: frequency + recency + prediction + capability + load_cost.

**Referenz:** RECHERCHEN R22, R25

---

## Ebene 6 — Routing

### Capability-Router

- Kleines Modell (0,1-0,5B)
- Erzeugt Capability-Vektor
- Reduziert Kandidatenmenge
- Originaler MoE-Router entscheidet final

### Graph-Ebenen

| Graph | Zweck |
|---|---|
| Knowledge | Fakten + Relationen |
| Capability | Experten-Faehigkeiten |
| Runtime | Systemzustand |
| Performance | Hardware-Eignung |
| Co-Activation | Experten-Korrelation |

### Tools

- KuzuDB (embedded Graph, Cypher)
- LanceDB (Vektor-Index, mmap)

**Referenz:** KONZEPT §15 (Nachtrag), TOOL_IDEEN CA

---

## Ebene 7 — Persistenz

### Datenbanken

| DB | Zweck | Groesse |
|---|---|---|
| SQLite + sqlite-vec | State, Audit | bis 1 GB |
| LanceDB | Vektor-Index | 10-100 GB |
| DuckDB | Analytik | 1-10 GB |
| KuzuDB | Graph | 1-10 GB |
| chDB | Logs (optional) | bei Bedarf |

### Eigenschaften

- Alle embedded (kein Server)
- Arrow-Bridge: LanceDB zu DuckDB
- Chroma-Migration geplant

**Referenz:** RESEARCH_IMPORT, TOOL_IDEEN (Kategorie-Erweiterungen)

---

## Ebene 8 — Protokoll

### Lokal

| Protokoll | Zweck |
|---|---|
| io_uring | Ringbuffer, Experten-Streaming |
| io_uring_cmd | NVMe-Passthrough |
| netlink | Kernel-Events |
| AF_UNIX | MCU-Kommunikation |
| varlink | Daemon-Steuerung |

### Netzwerk (spaeter)

- eBPF / XDP
- RDMA / RoCE
- NVMe-oF
- QUIC

### Evolution

```
Klassisch:  App -> Syscall -> VFS -> Driver -> Hardware
Jetzt:      App -> Shared Ringbuffer -> Hardware
Next:       App/GPU -> Peer-to-Peer -> Hardware (Zero-CPU)
```

**Referenz:** TOOL_IDEEN CC

---

## Ebene 9 — Arbeitsweise

### Backward Programming

Ziel zuerst, dann rueckwaerts zum Weg.

### Qualitaetsfilter

Konstruktive Recherchen bevorzugen:
- Sucht Wege, nicht Ausreden
- Commits, Flags, Zahlen
- Belegte Tests
- Uebertragbar

### Rollen

| Rolle | Aufgabe |
|---|---|
| Deck | Orchestrator |
| MCU | Ausfuehrer |
| Builder-Instanz | Bauen |
| Kontroll-Instanz | Pruefen |
| Cloud-Spezialisten | Recherche |

**Referenz:** KI_DELEGATION (Nachtrag)

---

## Datenfluss

```
User Task
   |
   v
Capability-Router (klein)
   |
   v
Expert-Predictor (Markov)
   |
   v
Co-Activation-Graph (KuzuDB)
   |
   v
Cache-Check (L0-L3)
   |
   +-- Hit --> MoE Compute (Vulkan)
   |
   +-- Miss --> NVMe Read (io_uring)
                   |
                   v
              Cache Slot
                   |
                   v
              MoE Compute
```

---

## Status pro Ebene

| Ebene | Status |
|---|---|
| 1 — Hardware | Kartiert |
| 2 — Session | Konzept |
| 3 — Compute | Laeuft (Vulkan getestet) |
| 4 — Modell | Konzept (Streaming ungetestet) |
| 5 — Cache | Konzept |
| 6 — Routing | Konzept |
| 7 — Persistenz | Konzept |
| 8 — Protokoll | Konzept |
| 9 — Arbeitsweise | Aktiv |

---

## Naechste Schritte

1. GTT-Tuning (14B freischalten)
2. Expert-Streaming testen (CachyLLama)
3. DB-Stack aufsetzen
4. Co-Activation-Graph bauen
5. Prefetch-Predictor

**Referenz:** KONZEPT §19 (Bau-Reihenfolge)

---

**Ende ARCHITEKTUR.md**
