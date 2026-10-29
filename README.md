# APP-OCR-AI-OSMP

Aplikasi berbasis AI untuk membaca dan mengekstrak informasi dari dokumen **Purchase Order (PO)**, kemudian mengonversinya ke dalam format **Excel** yang digunakan oleh perusahaan. Aplikasi ini terintegrasi dengan **SAP OData** untuk mengambil data referensi serta memproses hasil ekstraksi menjadi data yang sesuai dengan kebutuhan dan standar perusahaan.

---

## System Workflow

```text
[User Upload PO (PDF/Img)] 
       │
       ▼
┌──────────────────────────────┐    1. Kirim Dokumen + Encrypted Key    ┌──────────────────────────────┐
│   myapp (Laravel 12 Web)     │ ─────────────────────────────────────► │   AI-processing (FastAPI)    │
│  • Livewire 4 & UniverJS     │                                        │  • REST API Endpoint (:8001) │
│  • SAP OData Sync            │ ◄───────────────────────────────────── │  • Celery + Redis Queue      │
│  • MySQL Database (:3306)    │    4. Webhook Callback (/ocr-callback) └──────────────┬───────────────┘
└──────────────────────────────┘                                                       │ 2. Dispatch Task
                                                                                       ▼
                                                                        ┌──────────────────────────────┐
                                                                        │   Celery Worker + Gemini AI  │
                                                                        │  • PyMuPDF / Poppler PDF     │
                                                                        │  •3. Ekstraksi JSON Terstruktur│
                                                                        └──────────────────────────────┘
```

## Struktur Proyek

Repository ini terdiri dari 2 direktori utama:

```text
APP-OCR-AI-OSMP/
├── myapp/           # Web Application (Laravel 12, Livewire 4, Tailwind CSS, UniverJS Spreadsheet)
├── AI-processing/   # AI OCR Microservice (Python FastAPI, Celery Worker, Google Gemini SDK)
└── README.md        # Dokumentasi instalasi & penggunaan lokal
```

> **Catatan Mengenai GitHub (`vendor`, `node_modules`, `venv`, `.env`):**  
> Saat kode di-push ke GitHub, folder **`myapp/vendor/`**, **`myapp/node_modules/`**, **`AI-processing/venv/`**, serta file kredensial **`.env`** sengaja **tidak ikut di-upload** karena sudah terdaftar di `.gitignore` (agar ukuran repository tetap ringan dan API key/password tetap aman).  
> Seluruh daftar library tetap tersimpan di `composer.json`, `package.json`, dan `requirements.txt`, sehingga ketika project di-clone/download dari GitHub, folder tersebut akan otomatis dibuat ulang melalui perintah instalasi di bawah ini.

---

## 📑 Daftar Isi (Contents)

- [Spesifikasi & Kebutuhan Sistem (Prerequisites)](#1-spesifikasi--kebutuhan-sistem-prerequisites)
  - [Kebutuhan Backend & Frontend (`myapp` - Laravel)](#a-kebutuhan-backend--frontend-myapp---laravel)
  - [Kebutuhan AI Microservice (`AI-processing` - FastAPI & Celery)](#b-kebutuhan-ai-microservice-ai-processing---fastapi--celery)
  - [Daftar Dependency Utama](#c-daftar-dependency-utama)
- [Panduan Setup Awal Setelah Download / Clone dari GitHub](#2-panduan-setup-awal-setelah-download--clone-dari-github)
  - [Clone Repository](#langkah-1--clone-repository)
  - [Setup Proyek Laravel (`myapp`)](#langkah-2--setup-proyek-laravel-myapp)
  - [Setup Proyek FastAPI & Celery (`AI-processing`)](#langkah-3--setup-proyek-fastapi--celery-ai-processing)
- [Cara Menjalankan Project di Lokal (Run Local)](#3-cara-menjalankan-project-di-lokal-run-local)
  - [Run Program Laravel (`myapp`)](#a-run-program-laravel-myapp)
  - [Run Program FastAPI & Celery (`AI-processing`)](#b-run-program-fastapi--celery-ai-processing)
- [Ringkasan Cepat Perintah (Quick Cheat Sheet)](#4-ringkasan-cepat-perintah-quick-cheat-sheet)

---

## 1. Spesifikasi & Kebutuhan Sistem (Prerequisites)

Pastikan perangkat lokal Anda sudah terinstal software dan versi berikut sebelum menjalankan project:

| Software / Layanan | Versi | Link Download Resmi | Cara Install Singkat |
| :--- | :--- | :--- | :--- |
| **Opsi Praktis (Windows): Laragon** *(Direkomendasikan)* | Full 64-bit | [Download Laragon](https://laragon.org/download) | Jalankan installer `.exe`, buka Laragon, klik **Menu > Tools > Path > Add Laragon to Path**. Sudah otomatis mencakup **PHP 8.2+, MySQL, Composer, Node.js, Redis, dan Git**. |
| **Opsi Alternatif: XAMPP** | PHP `8.2.x` | [Download XAMPP](https://www.apachefriends.org/download.html) | Pilih versi **8.2.x**, jalankan installer, lalu aktifkan modul **Apache** & **MySQL** dari XAMPP Control Panel. |
| **PHP** *(Jika install terpisah)* | `8.2+` | [PHP Downloads](https://www.php.net/downloads.php) / [PHP Windows](https://downloads.php.net/~windows/releases/) | **Windows:** Ekstrak file `.zip` ke `C:\php`, salin `php.ini-development` menjadi `php.ini`, lalu tambahkan `C:\php` ke *System Environment Variables (PATH)*. |
| **Composer** | `2.x` | [Download Composer](https://getcomposer.org/download/) | **Windows:** Unduh dan jalankan `Composer-Setup.exe` (arahkan ke lokasi `php.exe` Anda). **Linux/macOS:** Ikuti perintah terminal di halaman download resmi. |
| **Node.js & NPM** | `v18+ / v20+ / v22+` | [Download Node.js](https://nodejs.org/en/download) | Unduh installer **LTS** (`.msi` untuk Windows / `.pkg` untuk macOS), jalankan hingga selesai. Cek dengan `node -v` dan `npm -v`. |
| **MySQL / MariaDB** *(Jika tanpa Laragon/XAMPP)* | `8.0+` / `10.6+` | [MySQL Installer](https://dev.mysql.com/downloads/installer/) | Jalankan installer, pilih **Server Only** atau **Developer Default**, atur port `3306` dan password `root`. |
| **Python** | `3.11` *(atau `3.10+`)* | [Download Python](https://www.python.org/downloads/) | Jalankan installer `.exe` (Windows). **WAJIB centang kotak `"Add python.exe to PATH"`** di bagian bawah sebelum klik *Install Now*. |
| **Redis Server** | `7.x / 8.x` | **Windows:** [Memurai (Redis Native)](https://www.memurai.com/get-memurai)<br>**Linux/Docker:** [Redis Downloads](https://redis.io/downloads/) | **Windows:** Unduh & instal **Memurai Developer Edition** (otomatis berjalan sebagai Windows Service di port `6379`), atau aktifkan Redis bawaan **Laragon**. **Linux (Ubuntu/Debian):** `sudo apt install redis-server`. |
| **Poppler Utils** *(Untuk konversi PDF ke Image)* | Latest | **Windows:** [Poppler Windows GitHub](https://github.com/oschwartz10612/poppler-windows/releases/)<br>**Linux:** [Poppler Official](https://poppler.freedesktop.org/) | **Windows:** Unduh file `Release-*.zip`, ekstrak ke `C:\poppler`, lalu tambahkan `C:\poppler\Library\bin` ke *System PATH*. **Linux:** Jalankan `sudo apt install poppler-utils`. |

### A. Kebutuhan Backend & Frontend (`myapp` - Laravel)
* **PHP**: Versi **8.2+** (direkomendasikan PHP `8.2.x`)
* **Composer**: Versi **2.x**
* **Node.js & NPM**: Node.js **v18+ / v20+ / v22+** & NPM (untuk build asset Vite & UniverJS)
* **Database**: **MySQL / MariaDB** (port default `3306`)
* **Ekstensi PHP (`php.ini`)**:
  * Pastikan ekstensi standar Laravel & Excel aktif: `gd`, `zip`, `fileinfo`, `mbstring`, `xml`, `dom`, `curl`, `pdo_mysql`, `openssl`, `bcmath`, `intl`.
  * Aktifkan **OPcache** di `php.ini` untuk performa optimal:
    ```ini
    zend_extension=opcache
    opcache.enable=1
    opcache.enable_cli=1
    ```

### B. Kebutuhan AI Microservice (`AI-processing` - FastAPI & Celery)
* **Python**: Versi **3.11** (kompatibel juga dengan Python `3.10+`)
* **Redis Server**: Berjalan di `127.0.0.1:6379` (wajib sebagai message broker & result backend untuk **Celery Worker**)
* **Poppler Utils** *(Khusus pengguna Linux / jika menggunakan `pdf2image`)*:
  ```bash
  sudo apt install poppler-utils
  ```

### C. Daftar Dependency Utama

#### Dependency Laravel (`myapp/composer.json` & `myapp/package.json`)
* **PHP Composer**:
  * `laravel/framework: ^12.0`
  * `livewire/livewire: ^4.1` & `livewire/volt: ^1.7.0`
  * `maatwebsite/excel: ^3.1`
* **JavaScript NPM**:
  * `@univerjs/*` (`core`, `engine-render`, `presets`, `sheets`, `sheets-ui`, `ui` `^0.15.5`)
  * `vite: ^7.0.7`, `tailwindcss`, `axios`

#### Dependency Python (`AI-processing`)
* `fastapi` — Web framework REST API
* `uvicorn` — ASGI server untuk menjalankan FastAPI
* `python-multipart` — Parsing form-data & upload file PDF/Gambar
* `google-genai` — Official Google Gemini AI SDK
* `pydantic` — Validasi data & skema JSON
* `pymupdf` & `pdf2image` — Pembacaan dan konversi dokumen PDF
* `pillow` — Pemrosesan gambar (Image processing)
* `python-dotenv` — Pembacaan konfigurasi environment `.env`
* `celery[redis]` — Background worker antrean ekstraksi dokumen asinkron
* `requests` — Pengiriman HTTP webhook callback dari Celery ke Laravel

---

## 2. Panduan Setup Awal Setelah Download / Clone dari GitHub

Ikuti langkah-langkah berikut secara berurutan dari awal saat baru mengunduh atau melakukan `git clone` dari GitHub:

### Langkah 1 — Clone Repository
```bash
git clone https://github.com/AhmadSon/APP-OCR-AI-OSMP.git
cd APP-OCR-AI-OSMP
```

---

### Langkah 2 — Setup Proyek Laravel (`myapp`)

1. **Masuk ke folder `myapp`:**
   ```bash
   cd myapp
   ```

2. **Install dependency PHP (`vendor`) menggunakan Composer:**
   ```bash
   composer install
   ```

3. **Install dependency Frontend (`node_modules`) menggunakan NPM:**
   ```bash
   npm install
   ```

4. **Salin file `.env.example` menjadi `.env`:**
   * **Windows (CMD / PowerShell):**
     ```powershell
     copy .env.example .env
     ```
   * **Linux / macOS / Git Bash:**
     ```bash
     cp .env.example .env
     ```

5. **Generate Application Key (`APP_KEY`):**
   ```bash
   php artisan key:generate
   ```

6. **Sesuaikan konfigurasi di dalam `myapp/.env`:**
   Buka file `myapp/.env` dan sesuaikan bagian Database, URL aplikasi, serta koneksi ke FastAPI:
   ```dotenv
   APP_URL=http://127.0.0.1:8000

   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=nama_database_anda
   DB_USERNAME=root
   DB_PASSWORD=

   QUEUE_CONNECTION=database

   # FASTAPI CONNECTION
   FASTAPI_URL=http://127.0.0.1:8001
   WEBHOOK_SECRET=isi_secret_bebas_tapi_harus_sama_dengan_env_fastapi
   ```
   > **Penting:**
   > * Pastikan `APP_URL` sesuai dengan alamat tempat Laravel berjalan (misal `http://127.0.0.1:8000` saat memakai `php artisan serve`), karena Laravel mengirimkan `APP_URL + /api/ocr-callback` ke FastAPI sebagai alamat tujuan webhook.
   > * Nilai `WEBHOOK_SECRET` di `myapp/.env` **wajib sama persis** dengan `WEBHOOK_SECRET` di `AI-processing/.env`.

7. **Jalankan Migrasi Database & Seeder (Akun Superadmin Default):**
   Pastikan database sudah dibuat di MySQL, lalu jalankan:
   ```bash
   php artisan migrate --seed
   ```
   > Perintah `--seed` akan membuat akun **Super Admin** awal (`admin@gmail.com` / password default: `password`) beserta seluruh permission role.

8. **Buat Symbolic Link Storage:**
   ```bash
   php artisan storage:link
   ```

---

### Langkah 3 — Setup Proyek FastAPI & Celery (`AI-processing`)

1. **Masuk ke folder `AI-processing`:**
   ```bash
   cd ../AI-processing
   ```

2. **Buat Virtual Environment (`venv`) dengan Python 3.11:**
   ```bash
   python -m venv venv
   ```

3. **Aktifkan Virtual Environment (`venv`):**
   * **Windows (CMD / PowerShell):**
     ```powershell
     venv\Scripts\activate
     ```
   * **Linux / macOS:**
     ```bash
     source venv/bin/activate
     ```

4. **Install Dependency Python:**
   Anda dapat menginstal package inti secara langsung:
   ```bash
   pip install fastapi uvicorn python-multipart google-genai pydantic pdf2image pymupdf pillow python-dotenv "celery[redis]" requests
   ```
   *(Atau menggunakan file requirements: `pip install -r requirements.txt`)*

   > **Catatan Tambahan:**
   > * Jika perlu memperbarui SDK Gemini secara terpisah: `pip install google-genai`
   > * Untuk pengguna Linux, pastikan sudah menginstal Poppler: `sudo apt install poppler-utils`

5. **Salin file `.env.example` menjadi `.env` di folder `AI-processing`:**
   * **Windows (CMD / PowerShell):**
     ```powershell
     copy .env.example .env
     ```
   * **Linux / macOS / Git Bash:**
     ```bash
     cp .env.example .env
     ```

6. **Sesuaikan isi file `AI-processing/.env`:**
   ```dotenv
   WEBHOOK_SECRET=isi_secret_bebas_tapi_harus_sama_dengan_env_laravel
   LARAVEL_CALLBACK_URL=http://127.0.0.1:8000/api/ocr-callback
   REDIS_URL=redis://127.0.0.1:6379/0
   GEMINI_MAX_CONCURRENCY=3
   ```
   > **Catatan Keamanan API Key Gemini:**  
   > Jangan menaruh Gemini API Key di dalam `.env` FastAPI. API Key Gemini diatur langsung melalui halaman **AI Management** di dashboard web Laravel (disimpan terenkripsi di database) dan dikirimkan secara otomatis oleh Laravel ke FastAPI setiap kali ada proses OCR.

---

## 3. Cara Menjalankan Project di Lokal (Run Local)

Untuk menjalankan sistem secara penuh di lokal, pastikan **MySQL** dan **Redis Server** sudah aktif, kemudian buka terminal untuk menjalankan layanan **Laravel (`myapp`)** dan **FastAPI (`AI-processing`)**:

### A. Run Program Laravel (`myapp`)

Buka terminal di dalam direktori `myapp`:

1. **Jalankan Web Server Laravel (Terminal 1):**
   ```bash
   php artisan serve
   ```
   *(Berjalan di `http://127.0.0.1:8000`)*

2. **Jalankan Vite Frontend Dev Server (Terminal 2):**
   ```bash
   npm run dev
   ```

3. **Jalankan Queue Worker Laravel (Terminal 3):**
   ```bash
   php artisan queue:work --tries=3
   ```

4. **Sinkronisasi / Tarik Data Referensi dari SAP (Opsional / Sesuai Kebutuhan):**
   Setelah konfigurasi URL, Username, dan Password SAP OData diatur di menu **System Settings** pada aplikasi web, jalankan perintah berikut untuk menarik data SAP ke database lokal:
   ```bash
   php artisan sap:sync
   ```
   *(Gunakan `php artisan sap:sync --fresh` jika ingin sekaligus menghapus data lokal yang sudah tidak ada di SAP).*

---

### B. Run Program FastAPI & Celery (`AI-processing`)

Buka terminal di dalam direktori `AI-processing`:

1. **Jalankan Server FastAPI (Terminal 4):**
   ```powershell
   venv\Scripts\activate
   uvicorn app.main:app --reload --port 8001
   ```
   *(FastAPI berjalan di `http://127.0.0.1:8001` — Cek status di `http://127.0.0.1:8001/health`)*

2. **Jalankan Celery Worker untuk Antrean Ekstraksi AI (Terminal 5):**
   ```powershell
   .\venv\Scripts\celery -A app.celery_app worker --loglevel=info -P solo
   ```
   > Flag `-P solo` wajib digunakan saat menjalankan Celery di lingkungan **Windows**.

---

## 4. Ringkasan Cepat Perintah (Quick Cheat Sheet)

| Layanan | Direktori | Perintah |
| :--- | :--- | :--- |
| **Laravel Web Server** | `myapp` | `php artisan serve` |
| **Vite Asset Bundler** | `myapp` | `npm run dev` |
| **Laravel Queue Worker** | `myapp` | `php artisan queue:work --tries=3` |
| **Tarik Data SAP** | `myapp` | `php artisan sap:sync` |
| **FastAPI Server** | `AI-processing` | `venv\Scripts\activate` lalu `uvicorn app.main:app --reload --port 8001` |
| **Celery OCR Worker** | `AI-processing` | `.\venv\Scripts\celery -A app.celery_app worker --loglevel=info -P solo` |
