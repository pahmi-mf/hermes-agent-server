# Roadmap & Panduan Setup Hermes Agent 24/7 (Multi-Hardware Option)

Dokumen ini berisi panduan langkah demi langkah untuk memindahkan dan mengonfigurasi **Hermes Agent** beserta **Ollama (Model Lokal)** ke laptop dedicated (Server Bot Telegram) running **Ubuntu Desktop/Server**.

Tujuan setup ini adalah menjadikan laptop server terdedikasi 24/7 yang aman, terisolasi, dan *fully local* tanpa membebankan daya MacBook Pro.

---

## 💻 Spesifikasi & Pemilihan Model (2 Opsi Hardware)

Pilih instruksi model di bawah ini sesuai laptop yang kamu pilih sebagai server:

| Parameter | Opsi A: Asus A456U | Opsi B: Lenovo ThinkPad T460 |
| :--- | :--- | :--- |
| **Prosesor** | Intel Core i5-7200U | Intel Core i5-6200U / i5-6300U |
| **RAM / SSD** | 12 GB / 1 TB SSD | **16 GB** / 256 SSD |
| **Model Utama (Rekomendasi)** | **`qwen2.5-coder:3b`** | **`qwen2.5-coder:7b`** |
| **Ukuran RAM Model** | $\approx 2.0\text{ GB}$ | $\approx 4.7\text{ GB}$ |
| **Karakteristik Performa** | Kecepatan respons lebih tinggi, sangat ringan di CPU. | Kecerdasan & pemahaman konteks kode jauh lebih tinggi, responsif untuk RAM 16GB. |
| **Perintah Download** | `ollama pull qwen2.5-coder:3b` | `ollama pull qwen2.5-coder:7b` |

> **Catatan ThinkPad T460:** Jika di ThinkPad T460 kamu merasa `qwen2.5-coder:7b` terlalu membebankan CPU saat pemrosesan panjang, kamu selalu bisa menurunkan opsi model ke `qwen2.5-coder:3b`.

---

## 🏗️ Overview Arsitektur Sistem

* **LLM Engine:** Ollama (Pemrosesan CPU Murni)
* **Agent Framework:** Hermes Agent (Gateway Mode)
* **Antarmuka Utama:** Telegram Bot
* **Keamanan:** Whitelist Telegram User ID & Firewall (UFW)
* **Jaringan:** Outbound Polling (Aman digunakan pada koneksi GSM / CGNAT)

---

## 📑 Daftar Langkah Eksekusi

```
Phase 1: Pembersihan di macOS (MacBook Pro)
  ├── 1.1 Hentikan Proses Hermes Gateway
  └── 1.2 Hapus Konfigurasi Lokal Mac

Phase 2: Persiapan Sistem Ubuntu di Laptop Server (Asus / ThinkPad)
  ├── 2.1 Konfigurasi Power Management (Prevent Lid Sleep)
  └── 2.2 Konfigurasi Keamanan Jaringan (UFW)

Phase 3: Instalasi Engine AI & Model Lokal
  ├── 3.1 Instalasi Ollama
  └── 3.2 Unduh Model Sesuai Hardware (3b atau 7b)

Phase 4: Instalasi & Konfigurasi Hermes Agent
  ├── 4.1 Instalasi Hermes CLI
  ├── 4.2 Pengaturan Akun & Konfigurasi Gateway
  └── 4.3 Pengaturan Whitelist Telegram ID (Security)

Phase 5: Pengujian & Pembuatan Auto-Start (Systemd)
  ├── 5.1 Pengujian Layanan via Telegram
  └── 5.2 Konfigurasi Auto-Start Booting (Systemd)
```

---

## 🛠️ Langkah-Langkah Detil Eksekusi

### PHASE 1: Pembersihan Konfigurasi di macOS (MacBook Pro)

Buka Terminal di MacBook Pro kamu dan jalankan perintah berikut untuk menghentikan proses gateway lama dan membersihkan cache:

```bash
# 1. Hentikan semua proses Hermes yang berjalan
pkill -f hermes

# 2. Hapus folder konfigurasi dan memori lokal di Mac
rm -rf ~/.hermes ~/.config/hermes
```

---

### PHASE 2: Persiapan Sistem Ubuntu di Laptop Server

Buka Terminal di Ubuntu laptop server (`Ctrl + Alt + T`).

#### 2.1 Mencegah Laptop Sleep Saat Layar Ditutup

Jalankan perintah berikut agar laptop tetap beroperasi saat layarnya ditutup:

```bash
sudo sed -i 's/#HandleLidSwitch=suspend/HandleLidSwitch=ignore/g' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
```

#### 2.2 Pengaturan Firewall Dasar (UFW)

Aktifkan firewall untuk memblokir koneksi masuk yang tidak dikenal:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

---

### PHASE 3: Instalasi Engine AI & Model Lokal

#### 3.1 Instalasi Ollama

Pasang Ollama di Ubuntu menggunakan *installer script* resmi:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

#### 3.2 Unduh Model Lokal (Pilih Sesuai Laptop)

* **JIKA MENGGUNAKAN ASUS A456U (RAM 12 GB):**
  ```bash
  ollama pull qwen2.5-coder:3b
  ```

* **JIKA MENGGUNAKAN THINKPAD T460 (RAM 16 GB):**
  ```bash
  ollama pull qwen2.5-coder:7b
  ```

Verifikasi instalasi model:

```bash
ollama list
```
*(Pastikan model yang di-pull ada di dalam daftar)*

---

### PHASE 4: Instalasi & Konfigurasi Hermes Agent

#### 4.1 Instalasi Hermes Agent CLI

Pasang paket Hermes Agent CLI:

```bash
curl -fsSL https://raw.githubusercontent.com/nousresearch/hermes-agent/main/install.sh | bash
```

#### 4.2 Inisialisasi Konfigurasi

Jalankan perintah konfigurasi Hermes:

```bash
hermes config
```

Isi parameter konfigurasi saat diminta:
* **LLM Provider:** `Ollama`
* **Base URL:** `http://localhost:11434`
* **Model Name:** `qwen2.5-coder:3b` *(jika Asus)* ATAU `qwen2.5-coder:7b` *(jika ThinkPad)*
* **Nous Portal API Key:** *(Boleh dimasukkan API key lama kamu sebagai fallback, atau kosongkan)*
* **Telegram Bot Token:** *(Masukkan Token dari BotFather)*

#### 4.3 Mengamankan Telegram Bot (Whitelist User ID)

Cari tahu Telegram User ID kamu melalui bot `@userinfobot` di Telegram (berupa deretan angka, misal `123456789`).

Edit file konfigurasi Hermes yang terbentuk di `~/.hermes/config.json`:

```json
{
  "llm": {
    "provider": "ollama",
    "model": "qwen2.5-coder:7b",
    "base_url": "http://localhost:11434"
  },
  "telegram": {
    "enabled": true,
    "bot_token": "YOUR_TELEGRAM_BOT_TOKEN_HERE",
    "allowed_users": [
      123456789
    ]
  }
}
```
*(Ganti model ke `qwen2.5-coder:3b` jika memakai Asus, dan ganti `123456789` dengan User ID Telegram kamu)*

---

### PHASE 5: Pengujian & Pembuatan Auto-Start (Systemd)

#### 5.1 Pengujian Manual (Smoke Test)

Jalankan gateway secara manual dari terminal laptop server:

```bash
hermes gateway
```

**Langkah Pengujian via Telegram:**
1. Buka aplikasi Telegram dari HP atau Mac kamu.
2. Kirim pesan `/start` ke bot.
3. Kirim pesan tes, misalnya: `"Halo, siapa kamu?"`
4. Kirim perintah pengaturan home channel: `/sethome`
5. Kirim instruksi memori bahasa: `/remember Selalu jawab dalam Bahasa Indonesia atau Inggris sesuai konteks pengguna.`

#### 5.2 Konfigurasi Auto-Start 24/7 (Systemd Service)

Agar Hermes Agent otomatis berjalan saat laptop dinyalakan atau setelah *restart*, buat file *service*:

```bash
sudo nano /etc/systemd/system/hermes.service
```

Isikan konfigurasi berikut (sesuaikan `YOUR_USERNAME` dengan username Ubuntu kamu):

```ini
[Unit]
Description=Hermes Agent Telegram Gateway
After=network.target ollama.service

[Service]
Type=simple
User=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME
ExecStart=/usr/local/bin/hermes gateway
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Simpan file (`Ctrl + O`, `Enter`, lalu `Ctrl + X`), kemudian aktifkan servicenya:

```bash
sudo systemctl daemon-reload
sudo systemctl enable hermes
sudo systemctl start hermes
```

Cek status layanan untuk memastikan berjalan lancar:

```bash
sudo systemctl status hermes
```

---

## 📌 Catatan Tambahan & Troubleshooting

* **Perbedaan Pengalaman 3b vs 7b:** Pada ThinkPad T460 (16GB RAM), model `7b` akan menghasilkan pemahaman koding dan tata bahasa yang lebih presisi, tetapi kecepatan pengetikan teks (*token generation speed*) mungkin sedikit lebih lambat daripada model `3b` di Asus.
* **Rate Limit Solved:** Karena pemrosesan menggunakan Ollama lokal, kamu tidak akan lagi menemui error `rate limited - retrying in 592s`.
* **Koneksi GSM:** Telegram Bot menggunakan *outbound long polling*, sehingga tidak terpengaruh oleh CGNAT/Dynamic IP dari modem/tethering seluler.
* **Troubleshooting Logs:** Jika bot tidak membalas, periksa log dengan perintah:
  ```bash
  journalctl -u hermes -f
  ```
