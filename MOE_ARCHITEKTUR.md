# MOE_ARCHITEKTUR

> Zweck: Kondensierte MoE-Recherche fuer das AI Memory System.
> Quellen: 3 KI-Recherchen (2026-10-03) + RECHERCHEN R22.
> Status: Referenz. Kein Bauen ohne Freigabe.
> Verknuepfung: RECHERCHEN R22, BUILD_PLANS, ARCHITEKTUR §4.

---

## 1. Paradigma: Dense -> Classic -> Fine-Grained

MoE ersetzt die FFN im Transformer-Block. Attention, RMSNorm,
Embeddings und Shared Experts bleiben dicht.

```
Dense:          y = W_down * sigma(W_up * x)
Classic MoE:    y = sum_{i in TopK(g(x), K)} pi_i(x) * E_i(x)
Fine-Grained:   y = E_shared(x) + sum_{i in TopK} pi_i(x) * E_i(x)
```

**FLOPs skalieren mit K * |E|, Speicher mit N * |E|.** Das ist der Deal:
Kapazitaet kaufen, Compute nicht.

| Familie | Experten | Routing | Aktiv | Vertreter |
|---|---|---|---|---|
| Dense | 1 FFN | - | 100 % | Llama 3.1 70B |
| Switch / Top-1 | viele breite | K=1 | ~1/N | Switch-C (1.6T, 2048 Exp) |
| Classic MoE | 8-16 grosse | K=2-4 | 25-30 % | Mixtral 8x7B, Grok-1, DBRX |
| Fine-Grained + Shared | 64-512 kleine + 1-2 | K=4-10 | 3-10 % | DeepSeek-V2/V3/R1, Qwen3, Kimi K2 |

**Kombinationsraeume:**
- Mixtral 8x7B: binom(8,2) = 28
- DeepSeek-V3: binom(256,8) ~ 4.4 * 10^14

Fine-Grained gewinnt, weil jeder Experte ~7x schmaler ist
(Mixtral 14336 -> DeepSeek 2048), also billiger zu laden.

---

## 2. Total vs. Active Parameters

Active = Attention + Embeddings + dichte Layer + Shared + K*Expert.
FLOPs skalieren mit Active, Speicher mit Total.

| Modell | Total | Active | Ratio | Routing |
|---|---|---|---|---|
| Mixtral 8x7B | 46,7B | 12,9B | 27,6 % | Top-2/8 |
| Mixtral 8x22B | 141B | 39B | 27,6 % | Top-2/8 |
| Qwen1.5-MoE-A2.7B | 14,3B | 2,7B | 19 % | Top-4 + 4 shared / 60 |
| Qwen3-30B-A3B | 30,5B | 3,3B | 10,8 % | Top-8/128 |
| gpt-oss-20b | 21B | 3,6B | 17 % | Top-4/32, MXFP4 |
| gpt-oss-120b | 117B | 5,1B | 4,4 % | Top-4/128, MXFP4 |
| DeepSeek-V2 | 236B | 21B | 8,9 % | Top-6 + 2 shared / 160, MLA |
| DeepSeek-V3 / R1 | 671B | 37B | 5,5 % | Top-8 + 1 shared / 256, MLA |
| Llama 4 Scout | 109B | 17B | 16 % | Top-1 + 1 shared / 16 |
| Llama 4 Maverick | ~400B | 17B | 4,3 % | Top-1 + 1 shared / 128 |
| Kimi K2 | 1T | 32B | 3,2 % | Top-8 + 1 shared / 384, MLA |

**Decoder-Grundformel (Batch=1, bandbreitenlimitiert):**

```
tok/s ~= BW / (Active_Params * Bits/8)
```

Zwei Fallen:
1. Active bestimmt FLOPs, nicht Speicher. Alle Total-Params
   muessen adressierbar sein (HBM/DRAM/NVMe).
2. "37B Active" bei V3 enthaelt bereits Attention + Shared + Embed,
   nicht nur 8 Experten.

**Sparsity-Trend 2023-2026:** 3,6x (Mixtral) -> 18-30x (V3, K2).

---

## 3. Shared + Routed + Aux-Loss-Free

**Routed Expert:** klassische FFN, nur aktiv wenn im Top-K.
**Shared Expert:** immer aktiv, faengt Common Knowledge ab.

Formel: y = E_shared(x) + sum pi_i * E_i(x)

**Warum Shared Experts wichtig sind:**
1. Kein Informationsloch bei Capacity-Overflow.
2. Pinned VRAM-Resident beim Offloading (Shared + MLA + Router).
3. Routed Experts duerfen schaerfer spezialisieren.

**Aux-Loss-Free Balancing (DeepSeek-V3):**

```
s_i = sigmoid(x^T * e_i)
Kandidat: s_i + b_i  (b_i = Bias, nur fuer Auswahl)
Gate:     s_i / sum(s_j) fuer j in TopK
Update:   b_i <- b_i + gamma * (f_mean - f_i)
```

Ueberladene Experten bekommen negatives b_i, unterladene positives.
Kein Extra-Loss, der gegen die LM-Objective arbeitet.

**Router-Stabilitaet:**
- Router in FP16/FP32, nie INT2.
- Route-scale (DeepSeek: ~2.5) skaliert Gate-Gewichte.
- Node-Limited Routing (V3: max 4 Nodes).

---

## 4. Offloading: PCIe vs. CPU-Compute

**Hierarchie:**

```
GPU HBM       ~1000 GB/s   Attention, Shared, heisse Experten, KV
DDR5 dual      ~90 GB/s    kalte Experten (rechnen oder parken)
PCIe 4.0 x16   ~32 GB/s    GPU<->RAM Transfers
NVMe Gen4      ~14 GB/s    Disk-MoE, lazy experts
```

**Bottleneck-Rechnung DeepSeek-V3 (Q4):**
- 8 Experten * ~50 MB = 400 MB/Schicht
- PCIe 4.0: ~12 ms/Schicht -> <3 tok/s Obergrenze
- CPU-AMX in-situ: ~1-3 ms/Schicht, Activation-Rueckweg 14 KB
- **Konsequenz: Compute zum Gewicht, nicht Gewicht zur GPU.**

**Drei Offloading-Pfade (llama.cpp / KTransformers):**

1. **Tensor-Override (llama.cpp):**

```
--n-cpu-moe N       # erste N MoE-Schichten auf CPU
-ot "exps=CPU"      # alle Experten-Tensoren auf CPU
--lazy-experts      # mmap ohne Prefetch, OS-Page-Cache als LRU
```

2. **KTransformers (CPU-AMX in-situ):**
GPU: MLA + Shared + KV. CPU: Routed Experts mit AMX/AVX-512.
Aktivierung (14 KB) statt Gewicht (50 MB) ueber PCIe.

3. **Slot-Pool + NVMe-Paging:**
GPU Expert-Slots (8-32), LRU/LFRU-Map, pread fehlender Experten.
Funktioniert auf M1 Pro 16 GB mit 8 Slots.

**Cache-Policy:** LFRU statt LRU. Score = freq / age.
LRU ist falsch, weil Hub-Experten selten aber regelmaessig kommen.

**Prefill-Problem:** Bei vielen Tokens wird fast jeder Experte aktiv.
Erwartet distinct: E * (1 - (1 - k/E)^B). Cache-Hit-Rate bricht ein.
Loesung: Prefill ueber CPU-AMX mit allen Experten im RAM.

---

## 5. Speculative Prefetching + Duo-Systeme

**Problem:** Router der Schicht L braucht h_L. h_L existiert erst
nach Attention L. Fuer Prefetch ist es zu spaet.

**Loesung:** Ein Mini-Agent sagt Experten voraus, bevor sie gebraucht werden.

Drei Instanzen (aufsteigend nach Kosten):

1. **Router-Kopf selbst (0 Parameter):**
Quasi-hidden State q_L = Norm(x_L) + Default-Vektor.
Router von L+1 darauf anwenden.
~14 % TPOT-Reduktion, kein Extra-Modell.

2. **Shared LoRA-Adapter:**
Trainiert nur auf Next-Layer-Expert-IDs.
Native Router bleibt Wahrheit, Adapter triggert nur DMA.

3. **Draft-Modell (1-3B, aligned):**
Emittiert Expert-Spur fuer H Lookahead-Schichten.
4-bit-Draft erreicht >90 % Genauigkeit (MoE-SpeQ).

**Miss-Policy:**
- Hit: Copy-Stream hat Experten schon im Slot. 0 H2D.
- Miss, hohe Konfidenz: sync H2D oder CPU-AMX.
- Miss, niedrige Konfidenz: LoRE-Surrogate + Shared auf GPU,
  exakter Residual asynchron auf CPU.

**Adaptive Lookahead:** Wenn Konfidenz unter Schwelle, H reduzieren.
Kein Overfetch, der den Cache vergiftet.

**Was heute zuverlaessig funktioniert:**
- Gate-Lookahead (Schicht L+1 auf Hidden von L anwenden)
- Trace-basierte Praediktoren (MoE-Infinity)
- Speculative Decoding + MoE (Amortisierung)

---

## 6. Frameworks: Was nutzen wir wann

| Framework | Ansatz | Wann nutzen |
|---|---|---|
| llama.cpp / GGUF | --cpu-moe, -ot exps=CPU, --lazy-experts | Single-Node, GGUF-Modelle |
| KTransformers | CPU-AMX in-situ, GPU fuer MLA+Shared | Wenn AMX-CPU verfuegbar |
| vLLM | Expert Parallelism, Wide EP | Cluster, produktiv |
| DeepSpeed-Inference | EP + Expert-Slicing, alle resident | GPU-Cluster |
| SGLang | Paged Experts (K von E resident) | On-Demand Loading |
| Unsloth | MoE-Training + Dynamic Quant | Fine-Tuning |

**Fuer unser Deck (Van Gogh, RDNA2, 14 GB RAM):**
- llama.cpp mit GGUF ist der pragmatische Weg.
- KTransformers setzt AMX voraus (Intel) - auf AMD nicht direkt.
- vLLM/SGLang sind Cluster-orientiert - nicht relevant.

---

## 7. Asymmetrische Quantisierung

MoE ist nicht uniform quantisierbar. Der Router entscheidet
ueber alle Downstream-Qualitaet.

| Tensor-Klasse | Praezision | Warum |
|---|---|---|
| Router / Gate | F16 / F32, nie < Q8 | Top-K-Stabilitaet |
| Attention (Q/K/V/O, MLA) | Q6_K-Q8_0 oder FP8 | jedes Token, fehleramplifizierend |
| Shared Expert | Q5_K-Q8 / FP8 | always-on |
| Haeufige Routed Experts | Q4_K_M / IQ4_XS / MXFP4 | 80-95 % der Touches |
| Seltene Routed Experts | IQ2_XXS / IQ2_M / IQ1_M | fast nie im Decode-Pfad |
| Norms, Bias | F32 | billig, kritisch |
| Embed / LM-Head | Q5_K-Q8 | Token-Identitaet |

**INT2-Quantisierung des Routers:**
- Top-k-Uebereinstimmung faellt auf 56,8 % (~halbe Token fehlgeleitet).
- Router-Projektion macht ~21,6 MB aus (<0,04 % der Gewichte).
- Quantisierung bringt null Speichervorteil, kostet alles.

**Unsloth Dynamic / UD-Quants:** Layer- und tensorweise
unterschiedliche Bit-Breiten. MoE-Experten aggressiver als Attention.

**Per-Expert mixed Quant (llama-quantize):**

```
--tensor-type 'attn_.*=Q8_0'
--tensor-type 'ffn_gate_exps=IQ2_M'
--tensor-type 'ffn_down_exps=Q4_K'
--tensor-type 'ffn_gate_inp=F16'
```

---

## 8. Systemdesign fuer unser Deck

**Hardware:** Steam Deck LCD, Van Gogh APU, 14 GB RAM (UMA), RDNA2.
Kein separates VRAM. GTT shared mit System.

**Zielmodell:** Qwen3-30B-A3B (30,5B total, 3,3B aktiv, 10,8 %).
Q4_K_M: ~18 GB Datei. Aktiv pro Token: ~2 GB (Q4).

**Was geht (16 GB UMA realistisch):**

| Konfiguration | Geschwindigkeit | Aufwand |
|---|---|---|
| Voll GPU (Q4, ~18 GB > 14 GB) | passt nicht | - |
| --n-cpu-moe 12 | 15-25 tok/s | einfach |
| Q3_K_M (~15 GB) voll GPU | 30-50 tok/s | mittel |
| Q4 + KTransformers (AMD CPU) | 5-10 tok/s | aufwaendig |

**Empfehlung:** Starte mit llama.cpp und --n-cpu-moe 12.
Das legt die ersten 12 MoE-Schichten auf CPU, Rest auf GPU.
Qwen3-30B-A3B Q4_K_M sollte dann mit 15-25 tok/s laufen.

**Was NICHT geht:**
- DeepSeek-V3 (671B, 37B aktiv): braucht 256+ GB RAM oder NVMe-Paging.
- Kimi K2 (1T): braucht Cluster oder sehr viel RAM.
- 30B-A3B bei Q8 (35 GB): passt nicht.

**Streaming-Ziel (CachyLLama, PR #27861):**
- Aktive Experten ~2 GB im RAM
- Rest auf NVMe
- Speculative Prefetch (Gate-Lookahead) fuer naechste Schicht
- Ziel: 30B-A3B mit 2 GB RAM + 15 GB NVMe

**Status:** Vulkan-Streaming auf gfx1033 ungetestet.
Erst nach GTT-Tuning (10 GB GTT) sinnvoll.

---

### Bezug zur DB-Architektur (Nachtrag 2026-10-04)

Die DB-Architektur (siehe KONZEPT §10, §11, §22) laeuft
unabhaengig vom Modell-Streaming.

| System | Medium | Was |
|---|---|---|
| Modelle | NVMe | Qwen3-30B-A3B, Streaming-Puffer |
| DB Hot | Externe SSD | SQLite, KuzuDB, DuckDB |
| DB Warm | SD (Deck-Slot) | latente Daten, grosse Vektor-Sets |
| DB Cold | Externe HDD | archived, Backups, zstd-Dumps |

**Kein Konflikt:**
- NVMe traegt Modelle, nicht DB.
- SSD traegt DB, nicht Modelle.
- MoE-Streaming (NVMe + GTT) und DB-Tier-Logik sind getrennte Systeme.

**Konsequenz fuer GTT-Tuning:**
Das GTT-Tuning (10 GB, ttm.pages_min) betrifft nur das Modell-Streaming.
Die DB-Performance haengt nicht an GTT, sondern an USB-Bandbreite
(SSD/HDD) und microSD-Latenz (SD). Getrennt optimieren.

---

## 9. Verweise

- RECHERCHEN.md R22 - MoE-Grundlagen, GPU vs CPU
- BUILD_PLANS.md §2-3 - CachyLLama, PR #27861
- ARCHITEKTUR.md §4 - MoE-Streaming, Prefetch-Predictor
- LESSONS_LEARNED.md Lesson 14-17 - CachyOS-Setup

---

**Ende MOE_ARCHITEKTUR.md - Stand 2026-10-03**
