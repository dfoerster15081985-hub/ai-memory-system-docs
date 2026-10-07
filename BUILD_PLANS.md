# BUILD_PLANS

> Zweck: Konkrete Bau-Kommandos fuer alle offenen Schritte.
> Stand: 2026-09-30
> Verknuepfung: RECHERCHEN R22-R26, ARCHITEKTUR.md

---

## 1. GTT-Tuning (14B freischalten)

Ziel: 8 GB -> 10 GB GTT fuer 14B Dense und
Streaming-Puffer.

### Schritt 1 — GRUB pruefen

```bash
grep GRUB_CMDLINE /etc/default/grub
```

Erwartet: Zeile mit ```ttm.pages_min=2097152```

### Schritt 2 — Backup

```bash
sudo cp /etc/default/grub /etc/default/grub.bak_$(date +%F)
cp /etc/default/grub /home/deck/backup-etc-boot/grub-$(date +%F)
```

### Schritt 3 — Parameter aendern

```bash
sudo nano /etc/default/grub
```

Aendern:

```
ttm.pages_min=2097152   ->   ttm.pages_min=2621440
```

(2097152 x 4096 = 8 GB, 2621440 x 4096 = 10 GB)

### Schritt 4 — GRUB neu bauen

```bash
sudo update-grub
```

### Schritt 5 — Reboot

```bash
sudo reboot
```

### Schritt 6 — Verifizieren

```bash
cat /proc/cmdline | tr ' ' '\n' | grep ttm.pages_min
cat /sys/class/drm/card0/device/mem_info_gtt_total
```

Erwartet:

- cmdline: ttm.pages_min=2621440
- GTT: 10737418240 (10 GB)

### Schritt 7 — 14B-Test

```bash
distrobox enter llama-vulkan-test
cd ~/llama.cpp
./build/bin/llama-bench \
  -m ~/models/qwen2.5-14b-instruct-q4_k_m-00001-of-00003.gguf \
  -ngl 999 -p 512 -n 128 -r 3
```

Erwartet: laeuft ohne "Not enough memory".

---

## 2. CachyLLama bauen

Ziel: MoE mit Expert-Residency (mmap-basiert).

### Schritt 1 — Klonen

```bash
cd ~
git clone https://github.com/fewtarius/CachyLLama.git
cd CachyLLama
git rev-parse HEAD > /tmp/cachyllama_commit.txt
cat /tmp/cachyllama_commit.txt
```

### Schritt 2 — Bauen im Container

```bash
distrobox enter llama-vulkan-test
cd ~/CachyLLama
cmake -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DGGML_VULKAN=ON \
  -DGGML_NATIVE=OFF
cmake --build build -j4
```

Dauer: 15-25 Min.

### Schritt 3 — Geraete pruefen

```bash
./build/bin/llama-cli --list-devices
```

Erwartet: RADV VANGOGH.

### Schritt 4 — Test mit 7B MoE

```bash
./build/bin/llama-cli \
  -m ~/models/qwen2.5-7b-instruct-q4_k_m-00001-of-00002.gguf \
  -ngl 999 \
  --moe-expert-residency \
  --moe-resident-per-layer 16 \
  --moe-prewarm-top-k 8 \
  --moe-residency-debug on \
  --moe-residency-debug-interval 64 \
  -c 2048 -n 128 \
  -p "Test"
```

Erwartet: laeuft, Debug-Ausgabe zeigt Hit-Rate.

---

## 3. PR #27861 bauen

Ziel: GPU-resident Expert Cache.

### Schritt 1 — Klonen

```bash
cd ~
git clone https://github.com/ggml-org/llama.cpp.git llama-27861
cd llama-27861
git fetch origin pull/27861/head:pr-27861
git checkout pr-27861
git rev-parse HEAD
```

### Schritt 2 — Bauen

```bash
distrobox enter llama-vulkan-test
cd ~/llama-27861
cmake -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DGGML_VULKAN=ON \
  -DGGML_NATIVE=OFF
cmake --build build -j4
```

### Schritt 3 — Cache-Test (klein anfangen)

```bash
./build/bin/llama-cli \
  -m ~/models/qwen2.5-7b-instruct-q4_k_m-00001-of-00002.gguf \
  -ngl 999 \
  --n-cpu-moe 4 \
  --moe-expert-cache 2 \
  --moe-expert-cache-inserts 1 \
  -c 2048 -n 128 \
  -p "Test"
```

Dann Cache-Groessen scannen:

```
cache = 0, 1, 2, 4, 8
inserts = 1, 2, 4
```

---

## 4. NVMe-Messung (fio)

Ziel: Erwartungswerte kalibrieren.

```bash
mkdir -p ~/fio-test

fio --name=seq1m --filename=$HOME/fio-test/testfile \
  --rw=read --bs=1M --iodepth=16 --ioengine=io_uring \
  --direct=1 --size=4G --runtime=20 --time_based \
  --group_reporting

fio --name=rand2m --filename=$HOME/fio-test/testfile \
  --rw=randread --bs=2M --iodepth=8 --ioengine=io_uring \
  --direct=1 --size=4G --runtime=30 --time_based \
  --group_reporting

rm ~/fio-test/testfile
```

Erwartet:

- seq 1M: ~3 GB/s
- rand 2M: ~2,4-2,9 GB/s

---

## 5. Expert-Streaming debuggen

Wenn CachyLLama oder #27861 laeuft, aber langsam:

### Reads zaehlen

```bash
strace -f -e trace=read,pread64 -o /tmp/llama-trace.txt \
  ./build/bin/llama-cli -m model.gguf --moe-stream -p "Test"

grep pread64 /tmp/llama-trace.txt | wc -l
```

### Welche Dateien?

```bash
grep pread64 /tmp/llama-trace.txt | awk 'print $NF' | sort -u
```

### I/O-Last

```bash
iostat -x 1 5
```

### Syscalls live (bpftrace)

Beispiel siehe: ```man bpftrace```


---

## 6. Reihenfolge

| Schritt | Aufwand | Wann |
|---|---|---|
| GTT-Tuning | 15 Min + Reboot | jetzt |
| 14B-Test | 10 Min | nach GTT |
| CachyLLama bauen | 30 Min | danach |
| #27861 bauen | 30 Min | danach |
| NVMe-Messung | 10 Min | parallel |
| Debugging | nach Bedarf | bei Problemen |

---

## 7. Abbruchpunkte

- GTT nach Reboot nicht 10 GB -> zurueck zum Backup
- CachyLLama Build fehlerhaft -> Container saubermachen
- #27861 Flag nicht erkannt -> falscher Branch
- Vulkan-Devices fehlen -> /dev/dri im Container pruefen

---


---

## 8. CachyOS: JMS567-SSD dauerhaft mounten

Erst Geraet und UUID mit `lsblk -f` identifizieren. Die UUID nicht aus diesem Beispiel
uebernehmen, falls das Laufwerk gewechselt oder neu formatiert wurde.

fstab-Muster:
UUID=<EXT256-UUID> /mnt/ext256 ext4 defaults,nombcache,noatime,data=writeback,nofail,x-systemd.device-timeout=5 0 2

Danach:
1. `sudo systemctl daemon-reload`
2. `sudo umount /mnt/ext256`
3. `sudo mount -a`
4. `mount | grep ext256` kontrollieren
5. Nach Neustart erneut Mountoptionen und Durchsatz pruefen

Performance-Test mit Schreibcache-Flush:
`sudo dd if=/dev/zero of=/mnt/ext256/speedtest_tmp bs=1M count=1000 status=progress conv=fdatasync`
Danach Testdatei entfernen. `data=writeback` bedeutet: Stromausfall kann juengste Dateidaten
gefaehrden; wichtige Daten brauchen ein separates Backup.

---

## 9. CachyOS: Ollama Vulkan auf Van Gogh

Installieren:
`sudo pacman -S ollama-vulkan`

Drop-in unter `/etc/systemd/system/ollama.service.d/override.conf`:
[Service]
User=dominikf
Group=dominikf
WorkingDirectory=/home/dominikf
Environment="HOME=/home/dominikf"
Environment="OLLAMA_MODELS=/home/dominikf/.ollama/models"
Environment="OLLAMA_IGPU_ENABLE=1"
ProtectHome=no

Aktivieren/neuladen:
`sudo systemctl daemon-reload`
`sudo systemctl enable --now ollama`

Pruefen:
- `systemctl show ollama -p Environment --no-pager`
- `journalctl -u ollama -b --no-pager | grep -iE 'Vulkan|GPU|inference compute'`
- `ollama ps` waehrend ein Modell geladen ist

Erfolgsbefund auf Van Gogh: Vulkan/RADV VANGOGH wird erkannt; Ollama meldet
`OLLAMA_IGPU_ENABLE=1`; der geladene Test `qwen2.5:1.5b-local` zeigte `100% GPU`.

Sicherheit: Dienst standardmaessig lokal auf `127.0.0.1:11434` belassen.

**Ende BUILD_PLANS.md**
