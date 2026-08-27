# BOT ATF MINER
==================

---

Panduan langkah demi langkah untuk menginstal dan menjalankan bot otomatis ATF Miner di Ubuntu Server:

## Langkah 1: Mengambil Data Autentikasi Akun

Sebelum mengonfigurasi server, ambil kredensial akun dari browser komputer:

- Buka Telegram Web di browser (Chrome/Brave) dan login ke akun Anda.

- Tekan `F12` untuk membuka Developer Tools, pilih tab `Network`, lalu klik filter `Fetch/XHR`.

- Buka Mini App ATF Miner di Telegram.

- Lakukan aksi di dalam aplikasi (misalnya klik tombol Claim, Start/Mine, atau buka menu utama).

- Cari request bernama index.php?action=login... di panel sebelah kiri.

-Klik tab Payload di panel sebelah kanan, lalu salin data berikut:

device_id (contoh: dev-79c2189d-...)

initData (teks panjang yang diawali query_id=... atau user=...)

tg_id (angka ID Telegram Anda, contoh: 5154985260)

username (username Telegram Anda)

ref_code (kode referral bawaan, contoh: 1535785495)


## Langkah 2: Persiapan Sistem di Ubuntu Server

Buka terminal Ubuntu Server Anda dan jalankan perintah berikut untuk menginstal dependensi dan menyiapkan folder kerja:

```Bash

# 1. Update repositori sistem
sudo apt update

# 2. Install Python dan modul HTTP requests
sudo apt install python3 python3-pip python3-requests -y

# 3. Buat folder baru dan masuk ke dalamnya
mkdir atf-bot
cd atf-bot
```

## Langkah 3: Membuat File Script Bot (main.py)

Salin dan tempel seluruh blok perintah di bawah ini ke terminal. Sesuaikan isi variabel DEVICE_ID, INIT_DATA, TG_ID, USERNAME, dan REF_CODE dengan data yang Anda salin pada Langkah 1:

```Bash

cat << 'EOF' > main.py
import time
import random
import uuid
import requests
from datetime import datetime

BASE_URL = "https://atfminers.asloni.online/miner/index.php"

# Ganti variabel di bawah ini sesuai akun Anda
DEVICE_ID = "dev-79c2189d-a38c-4a8d-8466-02d61742515e"
INIT_DATA = "query_id=AAEs1UIz........38a7941117"
TG_ID = 5154985260
USERNAME = "indogems2"
REF_CODE = "1535785495"

HEADERS = {
    "Accept": "application/json, text/plain, */*",
    "Content-Type": "application/json",
    "Origin": "https://atfminers.asloni.online",
    "Referer": "https://atfminers.asloni.online/miner/",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36"
}

session = requests.Session()
session.headers.update(HEADERS)

def log(msg):
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print(f"[{now}] {msg}")

def login():
    t = int(time.time() * 1000)
    url = f"{BASE_URL}?action=login&t={t}"
    payload = {
        "device_id": DEVICE_ID,
        "initData": INIT_DATA,
        "ref_code": REF_CODE,
        "request_id": str(uuid.uuid4()),
        "tg_id": TG_ID,
        "username": USERNAME
    }
    
    try:
        res = session.post(url, json=payload, timeout=15)
        if res.status_code == 200:
            log(f"[LOGIN] Sukses login akun: {USERNAME}")
            return True
        else:
            log(f"[LOGIN] Gagal: HTTP {res.status_code} - {res.text}")
            return False
    except Exception as e:
        log(f"[LOGIN] Request error: {e}")
        return False

def execute_claim():
    t = int(time.time() * 1000)
    url = f"{BASE_URL}?action=claim&t={t}"
    payload = {
        "device_id": DEVICE_ID,
        "initData": INIT_DATA,
        "request_id": str(uuid.uuid4()),
        "tg_id": TG_ID,
        "username": USERNAME
    }
    
    try:
        res = session.post(url, json=payload, timeout=15)
        if res.status_code == 200:
            data = res.json()
            earned = data.get("claimed_amount", 0)
            balance = data.get("new_pool_balance", "N/A")
            level = data.get("new_level", "N/A")
            log(f"[CLAIM] Berhasil | Dapat: +{earned} ATF | Saldo: {balance} ATF (Level {level})")
            
            # Hitung waktu cooldown siklus penambangan
            if "mining_freezes_at" in data and "server_now" in data:
                cooldown = max(60, data["mining_freezes_at"] - data["server_now"])
            else:
                cooldown = 7200
            return cooldown
            
        elif res.status_code == 409:
            log("[CLAIM] Siklus mining masih berjalan / request duplikat. Cek ulang 15 menit lagi...")
            return 900
        elif res.status_code == 401:
            log("[AUTH] Sesi expired. Mencoba login ulang...")
            if login():
                return 5
            return 3600
        else:
            log(f"[CLAIM] Gagal: HTTP {res.status_code} - {res.text}")
            return 600
    except Exception as e:
        log(f"[CLAIM] Request error: {e}")
        return 300

def main():
    print("==========================================")
    print("       ATF Miner Auto Claim Bot           ")
    print("==========================================")
    
    login()
    
    while True:
        cooldown_seconds = execute_claim()
        
        jitter = random.randint(30, 120)
        total_wait = cooldown_seconds + jitter
        
        hours = total_wait // 3600
        minutes = (total_wait % 3600) // 60
        log(f"[SLEEP] Menunggu {hours} jam {minutes} menit ({total_wait} detik) ke siklus berikutnya...\n")
        
        time.sleep(total_wait)

if __name__ == "__main__":
    main()
EOF
```

### Jalankan bot dengan perintah:

```Bash
python3 main.py
```

---

## Langkah 4: Menjalankan Bot di Background (24/7)

Jalankan perintah ini agar bot tetap aktif terus-menerus meskipun terminal atau koneksi server ditutup:

```Bash
nohup python3 main.py > atf.log 2>&1 &
```


## Langkah 5: Perintah Pemantauan & Manajemen

Melihat aktivitas log secara real-time:

```Bash
tail -f atf.log
```
(Tekan Ctrl + C untuk keluar dari tampilan log tanpa mematikan proses bot).

### Mengecek apakah proses bot masih aktif:

```Bash
ps aux | grep "python3 main.py"
```

Menghentikan bot:

```Bash
kill <NOMOR_PID>
```

(Ganti <NOMOR_PID> dengan angka ID proses yang muncul dari perintah ps aux).
