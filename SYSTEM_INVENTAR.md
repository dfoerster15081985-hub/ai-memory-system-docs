# SYSTEM_INVENTAR

> Zweck: Bestandsaufnahme des laufenden CachyOS-Systems (Lesson 13, Schritt 1).
> Stand: 2026-10-02
> Vorgaenger: SYSTEM_INVENTAR.md.pre_cachyos (SteamOS-Stand, historisch)

---

## 1. Betriebssystem

| Punkt | Wert |
|---|---|
| Distribution | CachyOS |
| Kernel | 7.2.8-2-cachyos |
| Root-FS | btrfs (Subvol /@, /@home, /@cache, ...) |
| Snapshot-System | limine-snapper-sync (pre/post bei pacman) |
| Rollback | sudo snapper rollback <ID> |

### Kernel-Cmdline (aktiv)

```
quiet nowatchdog splash rw rootflags=subvol=/@ root=UUID=59a151e5-6512-4a15-ae0a-f3c67d63a07d
```

**Keine Custom-Parameter aktiv.** Die SteamOS-Parameter (amdgpu.lockup_timeout,
ttm.pages_min) existieren hier nicht. GTT-Tuning ist eine offene Aufgabe.

---

## 2. Hardware

| Punkt | Wert |
|---|---|
| Geraet | Steam Deck LCD |
| CPU | 4x Zen2, 8 Threads |
| RAM | 14 GiB nutzbar |
| GPU | AMD VanGogh (RDNA2, Custom GPU 0405) |

### GTT

```
$ dmesg: amdgpu 0000:04:00.0: 7409M of GTT memory ready.
```

- GTT aktuell: **7409 MB** (Kernel-Default)
- SteamOS-Referenz: 8 GB via ttm.pages_min=2097152
- Ziel: 10 GB via ttm.pages_min=2621440 (offen)
- Hinweis: Auf Kernel 7.2 ist GTT-Verhalten anders — Wert ist dynamisch

---

## 3. Storage

### Intern

| Partition | Groesse | FS | Mount |
|---|---|---|---|
| nvme0n1p1 | 4 GB | vfat | /boot |
| nvme0n1p2 | 234,5 GB | btrfs | /, /home (Subvolumes) |

Belegung: 75 GB von 235 GB (32%).

### Extern (alle USB)

| Geraet | Label | FS | Mount | Chip | Status |
|---|---|---|---|---|---|
| sda1 (232,8G) | IMAGE | ext4 | /mnt/image | JMS578 | SteamOS-Abbild, Quelle |
| sda2 (9,3G) | RECOVERY | ext4 | — | JMS578 | Recovery |
| sda3 (689,4G) | BACKUP | ext4 | /mnt/backup | JMS578 | neu, Backup-Ziel |
| sdb1 (238,5G) | EXT256 | ext4 | /mnt/ext256 | JMS567 | DB-Kandidat |

### Performance-Messungen (02.10.2026)

| Laufwerk | Lesen | Schreiben (sustained) | Anmerkung |
|---|---|---|---|
| HDD (sda*) | 138 MB/s | 132 MB/s | SMR-Limit, kein Fix noetig |
| SSD (sdb) | 375 MB/s | 355 MB/s | JMS567-Fix aktiv |
| SSD (sdb, ohne Fix) | ~24 MB/s | ~24 MB/s | Referenzwert |

### fstab-Eintraege (neu, 02.10.2026)

```
UUID=41606b00-... (EXT256) /mnt/ext256 ext4 defaults,nombcache,noatime,data=writeback,nofail,x-systemd.device-timeout=5 0 2
UUID=d2a261ec-... (BACKUP) /mnt/backup ext4 defaults,noatime,nofail,x-systemd.device-timeout=5 0 2
```

---

## 4. Projekt-Umgebung

| Punkt | Wert |
|---|---|
| Projektpfad | /home/dominikf/AI_Memory_System |
| Groesse | 690 MB |
| Python | 3.14.7 (uv-managed) |
| Paketmanager | uv 0.12.22 |
| Umgebung | .venv/ (108 Pakete) |
| Manifest | pyproject.toml + uv.lock |
| Alte venv | venv.bak_py313/ (pip, historisch) |

### Provider (providers.toml)

| Prio | Name | Status |
|---|---|---|
| 0 | openrouter | ✓ getestet (Claude Opus 5.5) |
| 1 | cloudflare | ✓ |
| 2 | gemini | ✓ |
| 4 | groq | ✓ |
| 5 | mistral | ✓ |
| 99 | ollama | installiert; Vulkan, 100% GPU (qwen2.5:1.5b-local) |

### Modelle (~/models, 66 GB)

9 GGUF-Dateien: Qwen1.5-MoE-A2.7B (16 GB), Qwen3-30B-A3B, DeepSeek-V2-Lite,
Qwen2.5 (1.5B, 7B, 14B, coder).

### Web-UI

- FastAPI auf 127.0.0.1:8000
- Getestet: HTTP 200 ✓

---

## 5. Fehlende Komponenten (vs. SteamOS)

| Komponente | Rolle | Prioritaet |
|---|---|---|
| Ollama | lokaler Notanker (Prio 99) | installiert; Vulkan; Testmodell 100% GPU |
| llama.cpp / CachyLLama | Streaming-Test | hoch |
| GTT-Tuning | 14B freischalten | mittel |
| RTC-Wake-Units | Nightly-Jobs | niedrig |

---

## 6. Migration

Quelle: /mnt/image/home/deck/ (HDD sda1, Label IMAGE)
Ziel: /home/dominikf/

| Element | Groesse | Status |
|---|---|---|
| AI_Memory_System/ | 690 MB | ✓ |
| models/ | 66 GB | ✓ |
| Passwoerter | — | Google PW-Manager (bewusste Entscheidung) |

---

## 7. Verweis auf Dateien

- ARCHITEKTUR.md — 9 Ebenen
- LESSONS_LEARNED.md — 13 Lessons
- BUILD_PLANS.md — konkrete Bau-Kommandos
- KONZEPT_2026-09-21.md — Zukunftskonzept
- SYSTEM_INVENTAR.md.pre_cachyos — SteamOS-Referenz

---

## 9. Autonomie-Bausteine (neu 2026-10-02)

| Baustein | Datei | Rolle |
|---|---|---|
| Executor | core/executor.py | Shell-Ausfuehrung mit Guard + Snapshot |
| Guard | core/exec_guard.py | Command-Blacklist |
| Job-Queue | core/experience/jobs.py | persistente Queue in agent_job |
| Worker | core/experience/worker.py | autonomer Job-Verarbeiter |
| Migration 003 | migrations/003_agent_jobs.sql | Tabelle agent_job |
| Migration 004 | migrations/004_agent_job_recovery.sql | Worker-Lease-Felder |
| exec-Test | tests/test_exec_guard.py | 30 Testfaelle |
| job-Test | tests/test_jobs.py | 6 Testfaelle |

### Job-API (Web-UI)

- ```POST /jobs``` — Auftrag einreihen (task, session_id, priority)
- ```GET /jobs``` — Liste (status, limit)
- ```GET /jobs/{id}``` — Status + Ergebnis
- ```POST /jobs/{id}/cancel``` — Storno anfordern

### Sicherheitsebenen

1. **Snapper pre/post** vor jedem Befehl (root + home). Fail-closed:
   schlaegt Snapshot fehl, wird nicht ausgefuehrt.
2. **Guard Blacklist**: katastrophale Befehle werden blockiert
   (rm rekursiv auf Systempfade, mkfs, dd of=Blockgeraet, wipefs,
   sgdisk zap, parted/fdisk non-list, shred, badblocks -w,
   mv Systempfade, chmod/chown rekursiv auf Root, fork bomb).
3. **Keine Sandbox**: bewusste Entscheidung. rlimit war ohnehin schwach.
   Rollback statt Isolation.
4. **Lokale Bindung**: Web-UI nur 127.0.0.1; leerer auth_token.
5. **Fail-Safe Recovery**: verwaiste running-Jobs werden als error
   markiert, nicht automatisch wiederholt.

### Worker-Betrieb

- Manueller Start: ```uv run python -m core.experience.worker```
- Heartbeat-Intervall: 60 s, Timeout: 600 s
- Kein systemd-Dienst (offen)
- Ein Worker pro Maschine

---

**Ende SYSTEM_INVENTAR.md (CachyOS-Stand, 2026-10-02)**
