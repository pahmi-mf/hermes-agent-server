# Pasang Ubuntu Server 26.04.1 LTS, sampai Hermes di mesin lain mendapat balasan

Panduan ini untuk dikerjakan di laptop, dengan komputer lain atau ponsel di samping untuk membaca langkah dan nanti masuk lewat SSH. Laptop ini menjadi server `llama-server` saja. Hermes Desktop / Agent tetap di mesin lain: tool, file, shell, dan memori ada di sana.

Angka di bawah sama dengan [setup_guide.md](setup_guide.md). Backup/restore lengkap, upgrade RAM 32 GB, dan lampiran default upstream ada di dokumen itu, bukan diulang di sini.

Yang terpasang di laptop ini pada akhir panduan:

- Ubuntu Server 26.04.1 LTS (Resolute Raccoon), user `fahmi`
- Satu proses `llama-server` (llama.cpp, Vulkan), model `Qwen3.5-4B-UD-Q8_K_XL.gguf`, alias `qwen3.5-4b`
- Dengar di IP LAN laptop, port **8080**
- Context **16384**

Tidak ada basis data, tidak ada Ollama, dan tidak ada Hermes di laptop LLM ini.

Contoh alamat `192.168.1.50` dan subnet `192.168.1.0/24` hanya contoh. Ganti dengan angka LAN kamu.

---

## Sebelum menyentuh installer

Memasang Ubuntu Server **menghapus disk internal** laptop ini, termasuk Ubuntu Desktop yang sedang berjalan dan `~/.hermes`. Jangan mulai bagian installer sebelum salinan di bagian 1 ada di disk yang lain dan disk itu sudah dicabut.

Siapkan:

- USB kosong, 8 GB cukup (ISO server sekitar 2,7 GB)
- Disk eksternal, atau mesin lain di LAN, untuk menyimpan GGUF
- Listrik tercolok. Laptop ini akan menjadi server
- Mesin kedua di jaringan yang sama, untuk SSH dan untuk Hermes

---

## 1. Selamatkan berkas, lalu cabut disk cadangan

Colok disk eksternal. Di Ubuntu Desktop, Files biasanya memasangnya di bawah `/media/fahmi/`. Cek nama mount-nya:

```bash
lsblk -o NAME,SIZE,TYPE,TRAN,MOUNTPOINT,MODEL
ls -lh /media/fahmi
```

Di perintah bawah, ganti `/media/fahmi/CADANGAN` dengan path yang benar-benar ada. Disk internal (NVMe) bukan tujuan salinan.

```bash
ls -lh ~/.hermes/models
mkdir -p /media/fahmi/CADANGAN/cadangan-llm
cp -n ~/.hermes/models/Qwen3.5-4B-UD-Q8_K_XL.gguf /media/fahmi/CADANGAN/cadangan-llm/
```

File itu wajib. Ukurannya di rencana ini sekitar 5,5 GB. Tanpa file ini server tidak punya model.

`Qwen3.5-9B-Q6_K.gguf` (sekitar 6,9 GB) boleh ikut sebagai cadangan offline. File 9B tidak di-load pada RAM 16 GB, dan nanti tidak ditaruh di path yang di-start layanan.

```bash
cp -n ~/.hermes/models/Qwen3.5-9B-Q6_K.gguf /media/fahmi/CADANGAN/cadangan-llm/
```

Kalau percakapan, skill, dan SOUL di laptop ini ingin pindah ke mesin Hermes, hentikan Hermes dulu supaya basis datanya tidak setengah tertulis. Lalu buat arsip state. Perintah ini sama dengan bagian 7.1 di [setup_guide.md](setup_guide.md). Arsip berisi `.env` dan `config.yaml` (rahasia): izin `600`, jangan dicetak, jangan di-commit, jangan diunggah. Daftar lengkap apa yang ikut dan cara restore ada di bagian 7 dokumen itu.

```bash
cd "$HOME"
arsip="hermes-state-$(date +%F).tar.gz"
tar -C "$HOME" -czf "$arsip" \
  --exclude='.hermes/audio_cache' \
  --exclude='.hermes/image_cache' \
  --exclude='.hermes/cache' \
  --exclude='.hermes/logs' \
  --exclude='.hermes/locks' \
  --exclude='.hermes/telemetry' \
  --exclude='.hermes/runtime' \
  --exclude='.hermes/runtimes' \
  --exclude='.hermes/state' \
  --exclude='.hermes/models' \
  --exclude='.hermes/models_dev_cache.json' \
  --exclude='.hermes/models_dev_cache.etag' \
  --exclude='.hermes/tools' \
  --exclude='.hermes/hermes-agent' \
  --exclude='.hermes/installs' \
  --exclude='.hermes/source-checks' \
  --exclude='.hermes/plugin-update-checks' \
  --exclude='.hermes/.skills_prompt_snapshot.json' \
  --exclude='.hermes/gateway_state.json' \
  --exclude='.hermes/gateway-starts.log' \
  --exclude='.hermes/.clean_shutdown' \
  --exclude='*.lock' \
  --exclude='*.sock' \
  --exclude='*.pid' \
  --exclude='*.db-shm' \
  --exclude='*.db-wal' \
  .hermes
chmod 600 "$arsip"
cp -n "$arsip" /media/fahmi/CADANGAN/cadangan-llm/
```

Arsip state **tidak** berisi GGUF. Dua salinan itu terpisah. Keduanya harus sudah berada di luar disk laptop.

Kalau tidak ada disk eksternal, salin file yang sama ke mesin lain (`scp` atau `rsync`) sebelum USB installer ditulis. Salinan satu-satunya tidak boleh tinggal di disk yang akan dihapus.

Cek ukuran, paksa tulisan selesai, lalu lepas disk:

```bash
sync
ls -lh /media/fahmi/CADANGAN/cadangan-llm
```

GGUF 4B harus terlihat di situ. Lepas lewat eject di Files, atau `sudo umount` pada partisi itu, lalu **cabut** disk cadangan. Biarkan tercabut sampai Ubuntu Server selesai terpasang dan kamu login.

---

## 2. Unduh ISO dan cek checksum

Unduh di laptop ini selama Desktop masih ada, atau di mesin lain. Pakai ISO **server** 26.04.1 LTS, bukan Desktop, dan bukan 26.10.

- Halaman unduh: [https://ubuntu.com/download/server](https://ubuntu.com/download/server) — pilih Ubuntu 26.04.1 LTS, Intel atau AMD 64-bit
- Berkas di direktori rilis: `ubuntu-26.04.1-live-server-amd64.iso`
- URL langsung: [https://releases.ubuntu.com/26.04/ubuntu-26.04.1-live-server-amd64.iso](https://releases.ubuntu.com/26.04/ubuntu-26.04.1-live-server-amd64.iso)

Daftar berkas di [https://releases.ubuntu.com/26.04/](https://releases.ubuntu.com/26.04/) menyebut image itu Ubuntu 26.04.1 LTS (Resolute Raccoon), sekitar 2,7 GB.

Checksum yang dipakai Canonical untuk image ini adalah SHA256, di [https://releases.ubuntu.com/26.04/SHA256SUMS](https://releases.ubuntu.com/26.04/SHA256SUMS). Berkas itu juga memuat hash 26.04 (bukan .1) dan hash Desktop. Yang dicocokkan hanya baris `ubuntu-26.04.1-live-server-amd64.iso`.

Pada 22 Oktober 2026 baris resminya:

```text
cc8a95cde20f6ced61a322420de00f10cc3c90ced545daa46cb9c1a117f1d927 *ubuntu-26.04.1-live-server-amd64.iso
```

Selalu cocokkan lagi ke berkas yang baru diunduh. Kalau Canonical mengganti image, angka di `SHA256SUMS` yang menang, bukan angka di dokumen ini.

```bash
cd "$HOME/Downloads"
wget -O SHA256SUMS https://releases.ubuntu.com/26.04/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
```

Baris yang diharapkan:

```text
ubuntu-26.04.1-live-server-amd64.iso: OK
```

Kalau bukan `OK`, jangan tulis USB. Unduh ulang dari halaman di atas. Direktori yang sama juga menyediakan `SHA256SUMS.gpg`. Pemeriksaan yang wajib di panduan ini adalah `sha256sum` terhadap `SHA256SUMS`.

---

## 3. Buat USB boot

Isi USB akan terhapus. Disk cadangan tetap tercabut, supaya tidak ikut tertulis.

### Startup Disk Creator

Di laptop ini, selama masih Ubuntu Desktop: buka menu aplikasi, cari **Startup Disk Creator**. Kalau tidak ada:

```bash
sudo apt install usb-creator-gtk
usb-creator-gtk
```

Pilih ISO `ubuntu-26.04.1-live-server-amd64.iso` (lewat Other / berkas lain kalau daftar otomatis hanya menampilkan image Desktop). Pilih USB, bukan disk internal. Jalankan penulisan. Kalau aplikasi menolak ISO server, pakai `dd` atau balenaEtcher di bawah.

### dd

`lsblk` dulu. Kolom `TRAN` bernilai `usb` adalah flashdisk. Disk internal bertipe `nvme`. **`of=` adalah seluruh disk USB, misalnya `/dev/sdb`, bukan partisi `/dev/sdb1`, dan bukan `/dev/nvme0n1`.** Salah `of=` menghapus disk yang disebut di situ.

```bash
lsblk -d -o NAME,SIZE,MODEL,TRAN,TYPE
```

Contoh setelah kamu yakin `sdX` adalah USB (ganti `sdX`, jangan menyalin huruf ini mentah-mentah):

```bash
sudo dd if="$HOME/Downloads/ubuntu-26.04.1-live-server-amd64.iso" of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

### balenaEtcher

Pilih ISO, pilih USB, flash. Di daftar perangkat, pilih flashdisk. Jangan pilih NVMe internal.

Setelah salah satu cara selesai, USB boleh dicabut dari mesin yang menuliskannya.

---

## 4. Boot dari USB

Colok listrik. Colok USB ke laptop yang akan dihapus. Disk cadangan tetap tercabut. Nyalakan, dan baca petunjuk tombol di layar pabrikan.

Dokumentasi Ubuntu menyebut menu boot satu kali lewat **Escape**, **F2**, **F10**, atau **F12**, tergantung mesin. Pakai tombol yang memunculkan **menu boot satu kali**, lalu pilih USB. Jangan mengubah urutan boot permanen kalau menu satu kali sudah cukup.

Secure Boot boleh tetap nyala untuk ISO Ubuntu resmi. Matikan hanya kalau USB tidak mau boot.

Kalau firmware menampilkan mode disk, pilih **AHCI**. Mode RAID / Intel RST sering menyembunyikan NVMe dari installer.

Beberapa detik kemudian installer berhenti di pilihan bahasa. Itu subiquity, installer Ubuntu Server.

Navigasi, dari dokumentasi installer: panah atas/bawah, Space atau Enter untuk memilih, Tab untuk pindah fokus, Esc kembali, F1 menu bantuan (pojok kanan atas).

Kalau layar laptop sulit dibaca: setelah jaringan hidup, menu bantuan punya petunjuk SSH **ke installer**. Ikuti pengguna dan kata sandi yang tertulis di layar. Itu bukan login ke sistem yang sudah terpasang.

---

## 5. Layar installer

Ikuti urutan di bawah. Tutorial resmi “Basic installation” memakai default untuk banyak layar; dua hal di rencana ini **tidak** mengikuti default itu: OpenSSH harus dipasang, dan LVM tidak dipakai.

1. **Bahasa.** Terserah. English memudahkan mencocokkan pesan dengan dokumentasi upstream.
2. **Pembaruan installer.** Kalau ditawarkan, terima. Installer mengunduh versi baru dan melanjutkan dari titik yang sama.
3. **Keyboard.** Pilih tata letak yang sama dengan papan ketik di depan kamu. Kalau ragu, pakai Identify keyboard.
4. **Jenis instalasi, kalau layar itu ada.** Pilih Ubuntu Server biasa, bukan yang bertuliskan minimized. Image minimized terlalu tipis untuk perkakas yang langkah berikut anggap ada. Kalau layar ini tidak muncul, lanjutkan. Kalau ada centang driver pihak ketiga, biarkan kosong: driver Vulkan Intel dipasang nanti lewat apt.
5. **Jaringan.** DHCP cukup. Installer mencoba kabel lewat DHCP. Kalau laptop memakai Wi-Fi, sambungkan di layar ini. Alamat tetap diatur setelah instalasi. Proxy tidak diisi. Mirror dibiarkan default. Jaringan boleh gagal; instalasi tetap bisa jalan, dan `apt upgrade` menyusul setelah boot asal jaringan sudah hidup.
6. **Ubuntu Pro, kalau layar itu ada.** Lewati. Tidak dipakai di rencana ini.
7. **Disk.** Berhenti dan baca daftar disk.
   - Disk cadangan tidak boleh ada di daftar. Kalau masih terlihat, matikan mesin, cabut, boot USB lagi.
   - Pilih **use an entire disk** pada **NVMe internal**, bukan pada USB installer.
   - LVM tidak dipakai. Kalau opsi LVM tercentang, hilangkan centangnya.
   - Enkripsi tidak wajib. Kalau diaktifkan, kata sandi diminta setiap boot, dan itu menghalangi laptop headless yang dinyalakan tanpa keyboard. Biarkan mati.
   - Di ringkasan, pilih **Done**, lalu konfirmasi penulisan. Konfirmasi itu menghapus disk yang dipilih. Setelah dikonfirmasi, pemasangan berjalan di latar; jangan mematikan mesin.
8. **Profil.** Nama kamu kalau diminta. Nama server (hostname) bebas; `llm` cukup. Username persis **`fahmi`**. Kata sandi yang kuat. User ini yang punya `sudo`, dan path di unit systemd mengarah ke `/home/fahmi`.
9. **SSH.** Centang pemasangan **OpenSSH server**, baru kemudian Done. Tanpa centang itu, tutorial resmi hanya memilih Done, dan laptop headless tidak bisa dimasuki dari LAN. Impor kunci SSH dari GitHub atau Launchpad bersifat pilihan. Kalau kamu mengimpor kunci, autentikasi sandi lewat SSH mati secara default; login di konsol laptop dengan sandi tetap bisa. Kalau belum punya kunci, lewati impor dan pakai sandi dari mesin lain.
10. **Snap.** Jangan centang apa pun, termasuk Docker atau MicroK8s kalau nama itu muncul. Pilih Done.
11. Tunggu sampai pemasangan selesai. Pilih restart. Cabut USB saat diminta. Kalau USB masih terpasang, mesin bisa masuk installer lagi; cabut dan boot dari disk internal.

---

## 6. Boot pertama

Login di konsol sebagai `fahmi`, atau dari mesin lain setelah kamu melihat alamat di konsol:

```bash
ip -br addr
```

```bash
ssh fahmi@<ip-yang-terlihat>
```

Lalu:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt-get install -y curl
sudo timedatectl set-timezone Asia/Jakarta
timedatectl
```

`timedatectl` harus menyebut `Asia/Jakarta`. Kalau upgrade memasang kernel baru, reboot menyusul di akhir bagian 7, setelah grup dan penutup layar diatur.

---

## 7. Grup GPU, tutup layar, IP, firewall

### Grup `render` dan `video`

```bash
sudo usermod -aG render,video fahmi
```

Grup baru aktif sesudah login ulang. Nanti dicek dengan `id` setelah reboot.

### Tetap menyala saat tutup

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/no-sleep.conf >/dev/null <<'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
IdleAction=ignore
EOF
sudo systemctl restart systemd-logind
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Listrik tetap tercolok. Laptop ini server.

### IP LAN yang tidak berubah

Pilih salah satu.

**Lebih sederhana:** reservasi DHCP di router, diikat ke MAC laptop.

```bash
ip link show
ip -br addr
```

Catat alamat `link/ether` dan alamat IPv4 yang didapat. Di router, ikat MAC itu ke satu alamat, misalnya `192.168.1.50`. Setelah reservasi disimpan, reboot laptop dan cek `ip -br addr` lagi.

**Di laptop sendiri**, netplan static. Nama interface dari `ip -br link`. Kabel Ethernet lebih stabil daripada Wi-Fi. Gateway yang sedang dipakai:

```bash
ip route show default
```

Contoh `/etc/netplan/01-lan.yaml`. Ganti nama interface, alamat, dan gateway. Menyimpan file ini memutus SSH lama; siapkan konsol, atau sambungkan lagi ke alamat yang baru.

```yaml
network:
  version: 2
  ethernets:
    enp0s31f7:
      dhcp4: false
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 1.1.1.1]
```

```bash
sudo netplan apply
ip -br addr
```

Alamat itulah yang nanti diisi ke `--host` dan ke base URL Hermes. Ikat ke **IP LAN laptop**. `127.0.0.1` hanya menjangkau laptop sendiri. `0.0.0.0` membuka semua interface, jadi tidak dipakai.

Reboot sekali supaya grup, penutup layar, dan IP sama-sama teruji:

```bash
sudo reboot
```

Masuk lagi, lalu:

```bash
id
ip -br addr
```

`id` harus memuat `render` dan `video`. Alamat LAN harus sudah yang kamu kunci.

Dari mesin lain, sebelum firewall dinyalakan:

```bash
ssh fahmi@192.168.1.50
```

### UFW

Ganti `192.168.1.0/24` dengan prefix LAN yang sebenarnya. Pada alamat `192.168.1.50/24`, prefix-nya `192.168.1.0/24`. SSH dibuka untuk subnet yang sama, supaya `ufw enable` tidak mengunci kamu di luar.

Kalau perintah `ufw` tidak ada: `sudo apt-get install -y ufw`.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8080 proto tcp
sudo ufw enable
sudo ufw status verbose
```

Tidak ada aturan yang membuka 8080 ke mana saja. Dari mesin lain, `ssh fahmi@<ip>` sekali lagi. Pertahankan sesi itu.

---

## 8. llama.cpp dengan Vulkan

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake git pkg-config \
  libvulkan-dev glslc spirv-headers mesa-vulkan-drivers vulkan-tools
vulkaninfo --summary
```

`vulkaninfo` harus melihat Intel Graphics sebelum compile dilanjutkan. Kalau tidak, lihat bagian gangguan di bawah. Jangan lanjut.

```bash
git clone https://github.com/ggml-org/llama.cpp.git "$HOME/llama.cpp"
cd "$HOME/llama.cpp"
cmake -B build -DGGML_VULKAN=1
cmake --build build --config Release
```

Build ini makan waktu. Listrik tetap tercolok.

Cek perangkat, tanpa memuat model:

```bash
"$HOME/llama.cpp/build/bin/llama-server" --list-devices
```

Yang diharapkan bentuknya seperti `Vulkan0: Intel(R) Graphics (ADL GT2)`. Nama di kolom kiri (`Vulkan0`) yang dipakai sebagai `--device`. Kalau nama di kolom kiri berbeda, pakai nama itu di `LLAMA_DEVICE` nanti.

---

## 9. Taruh GGUF 4B

Colok lagi disk cadangan. Di Server, disk tidak selalu terpasang sendiri. Lihat namanya dengan `lsblk`. Contoh untuk partisi pertama USB/disk eksternal (ganti `sdX1`; jangan pasang NVMe internal ke `/mnt`):

```bash
sudo mkdir -p /mnt/cadangan
sudo mount /dev/sdX1 /mnt/cadangan
ls -lh /mnt/cadangan/cadangan-llm
```

Kalau `mount` menolak karena filesystem-nya exfat atau ntfs:

```bash
sudo apt-get install -y exfatprogs ntfs-3g
sudo mount /dev/sdX1 /mnt/cadangan
```

```bash
mkdir -p "$HOME/llama-models"
cp -n /mnt/cadangan/cadangan-llm/Qwen3.5-4B-UD-Q8_K_XL.gguf "$HOME/llama-models/"
ls -lh "$HOME/llama-models"
sync
sudo umount /mnt/cadangan
```

Satu file yang dilayani. File 9B tidak ditaruh di `~/llama-models`. Biarkan di disk cadangan, atau salin ke direktori lain yang tidak disebut unit systemd.

Arsip `hermes-state-*.tar.gz` belum dipakai di laptop ini. Restore-nya di mesin Hermes, langkahnya di bagian 7.1 [setup_guide.md](setup_guide.md).

---

## 10. Uji `llama-server` di terminal

File `/etc/llama-server/llama.env`. Tidak ada token di file ini. Ganti `LLAMA_HOST` dengan IP LAN laptop yang sudah dicek lewat `ip -br addr`.

```bash
sudo mkdir -p /etc/llama-server
sudo tee /etc/llama-server/llama.env >/dev/null <<'EOF'
# Angka dan path yang memesan RAM / mengikat jaringan.
# Ganti LLAMA_HOST dengan IP LAN laptop. Bukan 127.0.0.1, bukan 0.0.0.0.
MODEL_PATH=/home/fahmi/llama-models/Qwen3.5-4B-UD-Q8_K_XL.gguf
LLAMA_ALIAS=qwen3.5-4b
LLAMA_HOST=192.168.1.50
LLAMA_PORT=8080
LLAMA_CTX=16384
LLAMA_DEVICE=Vulkan0
EOF
sudo chmod 644 /etc/llama-server/llama.env
```

Start di terminal, supaya bisa dibatalkan dengan Ctrl+C. Port upstream bawaan build yang dicek di rencana ini adalah **9931**. Flag di bawah mengunci **8080**. Context bawaan `0` memuat window model dan itu yang membekukan mesin; flag mengunci **16384**.

Perintah ini tidak memuat `--tools`, `--agent`, atau `--mcp` / `--mcp-servers-config`. Tool tetap di mesin Hermes.

```bash
set -a
source /etc/llama-server/llama.env
set +a
"$HOME/llama.cpp/build/bin/llama-server" \
  --model "$MODEL_PATH" \
  --alias "$LLAMA_ALIAS" \
  --host "$LLAMA_HOST" \
  --port "$LLAMA_PORT" \
  --ctx-size "$LLAMA_CTX" \
  --device "$LLAMA_DEVICE" \
  --n-gpu-layers all \
  --parallel 1 \
  --cache-ram 0 \
  --fit off \
  --jinja \
  --no-webui
```

Biarkan terminal itu terbuka. Di terminal lain, saat model dimuat, tonton RAM. Proses mendengarkan alamat di `--host`, jadi URL kesehatan memakai IP LAN itu, dari laptop maupun dari mesin lain:

```bash
free -h
curl -fsS "http://192.168.1.50:8080/health"
curl -fsS "http://192.168.1.50:8080/v1/models"
```

`/v1/models` harus menyebut `qwen3.5-4b`. Ganti `192.168.1.50` dengan IP LAN laptop. `127.0.0.1` tidak menjawab selama `LLAMA_HOST` berisi IP LAN.

Kalau swap langsung besar, proses mati, atau mesin hampir macet: Ctrl+C, ubah `LLAMA_CTX=8192` di `/etc/llama-server/llama.env`, start lagi dengan perintah yang sama. Tetap di model 4B. Context di Hermes nanti ikut 8192.

Kalau Vulkan menolak memuat padahal context sudah 8192, ganti `--n-gpu-layers all` menjadi `--n-gpu-layers 0` di perintah manual dan di unit systemd bagian 11, supaya hitungan jatuh ke CPU. Context tetap dibatasi. iGPU dan CPU berbagi RAM yang sama, jadi menurunkan context tetap langkah pertama.

Alasan tiap flag, dan default upstream yang mudah tertukar, ada di [setup_guide.md](setup_guide.md) bagian Fase 7 dan lampiran. Jangan mengganti angka di atas dari ingatan.

Ctrl+C setelah `/v1/models` lolos, supaya port 8080 kosong untuk systemd.

---

## 11. systemd

Baru setelah uji manual lolos.

```bash
sudo tee /etc/systemd/system/llama-server.service >/dev/null <<'EOF'
[Unit]
Description=llama.cpp server (satu GGUF, context tetap)
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=300
StartLimitBurst=3

[Service]
Type=simple
User=fahmi
Group=fahmi
SupplementaryGroups=render video
EnvironmentFile=/etc/llama-server/llama.env
ExecStart=/bin/bash -c 'exec /home/fahmi/llama.cpp/build/bin/llama-server --model "$MODEL_PATH" --alias "$LLAMA_ALIAS" --host "$LLAMA_HOST" --port "$LLAMA_PORT" --ctx-size "$LLAMA_CTX" --device "$LLAMA_DEVICE" --n-gpu-layers all --parallel 1 --cache-ram 0 --fit off --jinja --no-webui'
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now llama-server
systemctl status llama-server
curl -fsS "http://192.168.1.50:8080/health"
curl -fsS "http://192.168.1.50:8080/v1/models"
```

Kalau layanan jatuh tiga kali dalam lima menit, systemd berhenti mencoba. Baca `journalctl -u llama-server -e` sebelum start lagi.

Dari mesin lain, perintah yang sama dengan IP LAN:

```bash
curl -fsS "http://192.168.1.50:8080/v1/models"
curl -fsS "http://192.168.1.50:8080/v1/chat/completions" \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.5-4b","messages":[{"role":"user","content":"Balas dengan satu kata: siap"}],"max_tokens":16}'
```

Balasan JSON harus berisi teks. Kalau curl ini gagal, jangan buka Hermes dulu; betulkan jaringan dulu (bagian gangguan).

---

## 12. Hermes di mesin lain

Pasang Hermes Desktop atau Hermes Agent di mesin yang dipakai untuk mengobrol dan menjalankan tool. Bukan di laptop server.

Di `~/.hermes/config.yaml` (bagian `model`) atau di layar Settings:

| Pengaturan | Nilai |
| :--- | :--- |
| Provider | `custom` |
| `model.base_url` | `http://192.168.1.50:8080/v1` |
| `model.default` | `qwen3.5-4b` |
| API key | Kosong. Kalau aplikasi menolak field kosong, isi `sk-no-key-required` hanya di klien |
| Context | `model.context_length: 16384` |

Di UI, context window model utama diisi **16384**, angka yang sama dengan `--ctx-size`. Nilai **0** berarti window penuh model. Jangan simpan 0. Kalau server diturunkan ke 8192, angka di Hermes ikut 8192, lalu mulai chat baru.

Matikan runtime lokal, di `config.yaml`:

```yaml
local_runtime:
  enabled: false
```

Selama `enabled: true`, Desktop bisa menyalakan `llama-server` sendiri di mesin yang salah.

Jangan menyalakan `--api-key` di laptop. Server tidak memeriksa kunci. Placeholder `sk-no-key-required` hanya untuk klien yang menolak string kosong.

`config.yaml` dan `.env` berisi rahasia. Jangan dicetak dan jangan dimasukkan ke git. Izin yang dipakai rencana ini: direktori `~/.hermes` mode `700`, kedua file itu mode `600`.

Kalau kamu merestore arsip dari bagian 1, lakukan di mesin Hermes ini dengan langkah bagian 7.1 [setup_guide.md](setup_guide.md), lalu periksa lagi `base_url`, `context_length`, dan `local_runtime.enabled`. Arsip dari Desktop lama bisa masih menyimpan context 0 atau runtime lokal.

Uji dari mesin Hermes:

```bash
curl -fsS "http://192.168.1.50:8080/v1/models"
```

Lalu satu **chat baru** yang pendek. Balasan teks di Hermes adalah garis akhir panduan ini.

---

## Selesai ketika

- `curl -fsS http://<ip-laptop>:8080/health` mengembalikan jawaban, dari mesin lain
- `curl -fsS http://<ip-laptop>:8080/v1/models` menyebut `qwen3.5-4b`
- Context Hermes **16384**, bukan 0 (atau 8192 di kedua sisi, kalau server diturunkan)
- `local_runtime.enabled` false
- Chat baru di Hermes mengembalikan teks
- Tutup laptop terkunci, listrik tercolok, lalu dari mesin lain `ssh` dan `curl` health masih jalan

---

## Kalau macet

**USB tidak muncul di menu boot.** Coba port USB lain, matikan Secure Boot hanya untuk percobaan ini, dan masuk lagi lewat menu boot satu kali (Esc, F2, F10, atau F12, sesuai petunjuk di layar saat mesin menyala). Tulis ulang USB kalau menu boot melihat perangkat tetapi tidak mau mem-boot. Fast boot di firmware, kalau ada, kadang melewati USB; menu satu kali menghindari itu.

**Installer tidak melihat NVMe.** Disk cadangan yang tercabut bukan penyebabnya. Masuk firmware, ubah mode disk dari RAID / Intel RST ke AHCI, simpan, boot USB lagi. Jangan memilih USB sebagai disk tujuan instalasi.

**`vulkaninfo` tidak melihat Intel GPU.** `id` harus memuat `render` dan `video`; kalau belum, reboot lalu login lagi. Paket `mesa-vulkan-drivers` sudah terpasang dari perintah bagian 8. `vulkaninfo --summary` dijalankan sebagai user `fahmi`, bukan lewat `sudo` (root tidak berada di grup itu). Tanpa perangkat Intel di keluaran itu, jangan lanjut ke `cmake`.

**`llama-server` swap atau mesin membeku.** Ctrl+C atau `sudo systemctl stop llama-server` kalau layanan sudah enable. Set `LLAMA_CTX=8192`, start lagi, samakan `context_length` Hermes ke 8192, chat baru. Tetap di Qwen3.5-4B Q8. Kalau mesin tidak merespons: power-off, boot, turunkan `LLAMA_CTX` sebelum layanan naik lagi dengan 16384. Catatan recovery ada di Fase 8 [setup_guide.md](setup_guide.md).

**Hermes tidak tersambung.** Di laptop, `ss -lnt 'sport = :8080'` harus menampilkan IP LAN, misalnya `192.168.1.50:8080`. `127.0.0.1:8080` berarti `LLAMA_HOST` masih localhost; Hermes di mesin lain tidak bisa masuk. Samakan `LLAMA_HOST` dengan `ip -br addr`, restart layanan, dan samakan `model.base_url`. Cek `sudo ufw status verbose`: 8080 hanya dari subnet LAN, dan mesin Hermes berada di subnet itu. Cek IP yang dipakai Hermes sama dengan IP laptop saat ini, bukan alamat lama dari DHCP.

**Nama model tidak cocok.** `/v1/models` harus berisi `qwen3.5-4b`, sama dengan `model.default` di Hermes. Kalau yang muncul path file `.gguf`, alias tidak kepakai: cek `LLAMA_ALIAS` dan restart. Context 0 di Hermes tetap salah sekalipun nama model sudah benar.

---

## Lanjut di setup_guide.md

- Bagian 7: restore state Hermes, cadangan unit systemd, dan pindah mesin
- Bagian 2.2: upgrade RAM 32 GB dan kapan model lebih besar boleh dicoba
- Fase 7 dan lampiran: arti tiap flag dan default upstream (`--port` 9931, `--ctx-size` 0, `--cache-ram` 8192, `--fit on`, dan yang lain)
