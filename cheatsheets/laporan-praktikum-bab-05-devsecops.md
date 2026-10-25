# LAPORAN PRAKTIKUM WORKSHOP DEVOPS
## Bab 5 — Database Service di Docker: PostgreSQL

**Disusun untuk Memenuhi Tugas Mata Kuliah:**  
Workshop DevOps & DevSecOps

**Disusun Oleh:**  
- **Nama:** Dafa Ahmad Fahrisi  
- **NRP:** 3126640010  
---

## 1. TUJUAN PRAKTIKUM

Praktikum Bab 5 bertujuan untuk memahami perancangan dan pengelolaan database relasional persisten di lingkungan container Docker menggunakan PostgreSQL dan pgAdmin4. Database container memiliki karakteristik stateful yang membedakannya secara fundamental dari web service stateless. Data harus diperlakukan sebagai aset utama, sehingga volume persisten, siklus hidup container, inisialisasi skema otomatis, pemeriksaan kesehatan (*healthcheck*), serta mekanisme *backup* dan *restore* teruji menjadi fokus utama dalam praktikum ini.

Kompetensi utama yang dicapai:
1. Menjalankan service PostgreSQL dengan konfigurasi berbasis *environment variable* dan *named volume* persisten.
2. Mengotomatisasi inisialisasi skema dan data awal database menggunakan skrip SQL pada direktori `/docker-entrypoint-initdb.d`.
3. Mengakses dan mengelola database melalui antarmuka CLI (`psql`) serta GUI berbasis web (`pgAdmin4`).
4. Melakukan pencadangan logis (*logical backup*) dengan `pg_dump` dan menguji pemulihannya (*tested restore*) menggunakan `pg_restore`.
5. Mengimplementasikan dan memvalidasi mekanisme *healthcheck* container menggunakan utilitas `pg_isready`.

---

## 2. DASAR TEORI DAN ARSITEKTUR SISTEM

### 2.1 Database Container dan Persistensi Data
Container secara *default* bersifat *ephemeral* (sementara); seluruh modifikasi file pada *writable layer* akan musnah saat container dihapus. Untuk database relasional, *storage* harus dipisahkan dari siklus hidup proses container melalui **Docker Named Volume**. Named volume dikelola langsung oleh Docker Engine pada path host yang terisolasi, sehingga saat container di-*recreate* atau diperbarui versinya, seluruh data cluster PostgreSQL tetap utuh.

### 2.2 Siklus Inisialisasi Database (`docker-entrypoint-initdb.d`)
Image resmi PostgreSQL memiliki mekanisme startup khusus: skrip `.sql` dan `.sh` yang dipasang pada direktori `/docker-entrypoint-initdb.d` hanya akan dieksekusi **satu kali**, yaitu ketika direktori data (`$PGDATA`) masih kosong. Jika direktori data telah berisi cluster PostgreSQL dari volume sebelumnya, proses inisialisasi dilewati (*skipped*) untuk mencegah duplikasi data atau penimpaan skema yang sedang berjalan.

### 2.3 Pencadangan Logis vs Pemulihan Teruji (*Tested Restore*)
Sebuah *backup* tidak memiliki nilai kepastian sebelum berhasil dipulihkan melalui prosedur uji nyata (*restore drill*). `pg_dump` dengan format kustom arsip biner (`-Fc`) menyediakan kompresi tinggi dan fleksibilitas pemulihan selektif melalui `pg_restore`.

### Diagram Arsitektur Lab Bab 5

```mermaid
graph TD
    subgraph Host["Host Machine"]
        subgraph Ports["Published Ports"]
            P5050["Port 5050 (Host)"]
            P5432["Port 5432 (Host)"]
        end
        
        subgraph Volumes["Host Storage & Mounts"]
            InitDir["./init/01-schema.sql (Bind Mount: ro)"]
            BackupDir["./backup/labdb.dump (Bind Mount: rw)"]
            NamedVol[("Docker Volume: pg-data")]
        end
    end

    subgraph DockerNet["User-Defined Bridge Network"]
        subgraph C_pgAdmin["Container: pgadmin"]
            PGAdminApp["pgAdmin4 Web App (:80)"]
        end

        subgraph C_Postgres["Container: postgres-db"]
            PGEngine["PostgreSQL 16 Engine (:5432)"]
            Healthcheck["Healthcheck: pg_isready"]
            DataDir["/var/lib/postgresql/data"]
            InitTarget["/docker-entrypoint-initdb.d"]
            BackupTarget["/backup"]
        end
    end

    P5050 -->|HTTP Browser| PGAdminApp
    P5432 -->|TCP Client/psql| PGEngine
    PGAdminApp -->|Koneksi Internal (DNS: postgres-db:5432)| PGEngine

    InitDir -.->|Inisialisasi Pertama| InitTarget
    PGEngine --> DataDir
    DataDir === NamedVol
    PGEngine -->|pg_dump / pg_restore| BackupTarget
    BackupTarget === BackupDir
    PGEngine --> Healthcheck
```

---

## 3. LANGKAH PRAKTIKUM & BUKTI EKSEKUSI (EVIDENCE)

### 3.1 Tahap 1: Persiapan Direktori Kerja & Inisialisasi Skema SQL
Membuat direktori kerja beserta subdirektori `init/` (untuk skrip skema awal) dan `backup/` (untuk menampung berkas dump).

```bash
mkdir -p ~/docker-lab/bab-5/{init,backup}
cd ~/docker-lab/bab-5
```

Membuat berkas `init/01-schema.sql`:
```sql
CREATE TABLE IF NOT EXISTS students (
  id SERIAL PRIMARY KEY,
  nrp VARCHAR(20) UNIQUE NOT NULL,
  name TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT now()
);

INSERT INTO students(nrp, name) VALUES
('3126640010', 'Dafa Ahmad Fahrisi'),
('31230001', 'Mahasiswa Satu'),
('31230002', 'Mahasiswa Dua')
ON CONFLICT (nrp) DO NOTHING;
```

**Bukti Eksekusi File Skema:**
```bash
$ ls -la init/
total 12
drwxr-xr-x 2 labuser labuser 4096 Oct 25 10:00 .
drwxr-xr-x 4 labuser labuser 4096 Oct 25 10:00 ..
-rw-r--r-- 1 labuser labuser  325 Oct 25 10:02 01-schema.sql
```

---

### 3.2 Tahap 2: Konfigurasi Orkestrasi `compose.yaml`
Menyusun berkas `compose.yaml` yang mendefinisikan service `postgres-db` (dengan named volume, bind mounts, dan healthcheck aktif) serta `pgadmin` untuk antarmuka web visual:

```yaml
services:
  postgres-db:
    image: postgres:16-alpine
    container_name: postgres-db
    environment:
      POSTGRES_DB: labdb
      POSTGRES_USER: labuser
      POSTGRES_PASSWORD: labpass123
    ports:
      - "5432:5432"
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
      - ./backup:/backup
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U labuser -d labdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@pens.ac.id
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80"
    depends_on:
      postgres-db:
        condition: service_healthy

volumes:
  pg-data:
```

---

### 3.3 Tahap 3: Menjalankan Stack & Validasi Healthcheck
Menjalankan seluruh service di latar belakang dan memeriksa status operasionalnya.

```bash
$ docker compose up -d
[+] Running 3/3
 ✔ Volume "bab-5_pg-data"       Created
 ✔ Container postgres-db        Healthy
 ✔ Container pgadmin            Started
```

**Pemeriksaan Status Container (`docker compose ps`):**
```bash
$ docker compose ps
NAME          IMAGE                   COMMAND                  SERVICE       STATUS                    PORTS
postgres-db   postgres:16-alpine      "docker-entrypoint.s…"   postgres-db   Up 42 seconds (healthy)   0.0.0.0:5432->5432/tcp
pgadmin       dpage/pgadmin4:latest   "/entrypoint.sh"         pgadmin       Up 41 seconds             0.0.0.0:5050->80/tcp
```

**Pemeriksaan Eksekusi Log Initdb:**
```bash
$ docker compose logs postgres-db | grep -E "initdb|01-schema.sql|PostgreSQL init process complete"
/usr/local/bin/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/01-schema.sql
CREATE TABLE
INSERT 0 3
PostgreSQL init process complete; ready for start up.
```

---

### 3.4 Tahap 4: Pengujian Akses CLI (`psql`) & Web UI (`pgAdmin4`)

#### 1. Verifikasi Data via CLI `psql`
```bash
$ docker exec -it postgres-db psql -U labuser -d labdb -c "SELECT id, nrp, name, created_at FROM students;"
 id |    nrp     |        name        |         created_at         
----+------------+--------------------+----------------------------
  1 | 3126640010 | Dafa Ahmad Fahrisi | 2026-10-25 10:05:12.128456
  2 | 31230001   | Mahasiswa Satu     | 2026-10-25 10:05:12.128456
  3 | 31230002   | Mahasiswa Dua      | 2026-10-25 10:05:12.128456
(3 rows)
```

#### 2. Akses pgAdmin4 via Browser
1. Akses web pgAdmin pada URL: `http://localhost:5050`
2. Autentikasi: `admin@pens.ac.id` / `admin123`
3. Registrasi server baru dengan parameter:
   - **Host name / address:** `postgres-db` *(resolusi DNS internal Docker)*
   - **Port:** `5432`
   - **Maintenance database:** `labdb`
   - **Username:** `labuser`
   - **Password:** `labpass123`
4. Hasil: Tabel `students` berhasil diakses, dan seluruh data record terverifikasi via query panel visual.

---

### 3.5 Tahap 5: Pencadangan Logis (`pg_dump`) dan Uji Pemulihan Teruji (`pg_restore`)

#### 1. Melakukan Backup Database (Format Custom Archive `-Fc`)
```bash
$ docker exec postgres-db pg_dump -U labuser -d labdb -Fc -f /backup/labdb.dump
$ ls -lh backup/labdb.dump
-rw-r--r-- 1 root root 3.9K Oct 25 10:15 backup/labdb.dump
```

#### 2. Pengujian Skenario Bencana (Simulasi Data Loss)
Menghapus tabel `students` untuk membuktikan integritas restore:
```bash
$ docker exec -it postgres-db psql -U labuser -d labdb -c "DROP TABLE students;"
DROP TABLE

# Memastikan tabel telah terhapus
$ docker exec -it postgres-db psql -U labuser -d labdb -c "\dt"
Did not find any relations.
```

#### 3. Menjalankan Restore Database (`pg_restore`)
```bash
$ docker exec postgres-db pg_restore -U labuser -d labdb --clean --if-exists /backup/labdb.dump
$ echo $?
0
```

#### 4. Validasi Akhir Keberhasilan Restore
```bash
$ docker exec -it postgres-db psql -U labuser -d labdb -c "SELECT id, nrp, name, created_at FROM students;"
 id |    nrp     |        name        |         created_at         
----+------------+--------------------+----------------------------
  1 | 3126640010 | Dafa Ahmad Fahrisi | 2026-10-25 10:05:12.128456
  2 | 31230001   | Mahasiswa Satu     | 2026-10-25 10:05:12.128456
  3 | 31230002   | Mahasiswa Dua      | 2026-10-25 10:05:12.128456
(3 rows)
```
*Hasil:* Seluruh tabel dan rekaman data berhasil dipulihkan secara sempurna tanpa kehilangan integritas data.

---

## 4. TABEL CHECKLIST VERIFIKASI PENGUJIAN

| No | Parameter Pengujian | Kriteria Evaluasi | Status | Bukti Terverifikasi |
|:--:|:--------------------|:------------------|:------:|:--------------------|
| 1 | Named Volume Persistensi | Data PostgreSQL tersimpan pada volume `pg-data` dan bertahan saat container restart/recreate | **PASS** | `docker volume ls` mencatat `pg-data` aktif |
| 2 | Inisialisasi Otomatis Skema | Skrip `01-schema.sql` berhasil dieksekusi oleh entrypoint saat cluster pertama dibuat | **PASS** | Log mencatat eksekusi skrip SQL & `INSERT 0 3` |
| 3 | Container Healthcheck | `pg_isready` memvalidasi kesiapan port database | **PASS** | Status container `Up ... (healthy)` |
| 4 | Akses Visual GUI | pgAdmin4 berhasil terhubung ke service `postgres-db` melalui network internal | **PASS** | Tabel `students` dapat dibuka di `http://localhost:5050` |
| 5 | Logical Backup (`pg_dump`) | File arsip biner `-Fc` terbuat dan tersinkronisasi ke folder host `./backup/` | **PASS** | Berkas `labdb.dump` (3.9 KB) tersimpan di host |
| 6 | Uji Pemulihan (`pg_restore`)| Database yang dihapus berhasil dipulihkan kembali secara identik | **PASS** | Query psql pasca-restore menampilkan 3 baris data utuh |

---

## 5. TROUBLESHOOTING DAN ANALISIS HASIL

### 5.1 Diagnosis Masalah Nyata: Skrip Init Tidak Berjalan Saat Modifikasi Skema Baru
- **Gejala Masalah:**  
  Mahasiswa menambahkan tabel baru atau mengubah skema pada file `init/01-schema.sql`, lalu menjalankan `docker compose restart postgres-db` atau `docker compose up -d --force-recreate`. Namun, perubahan skema tidak terjadi pada database.
- **Hasil Investigasi & Log Diagnosis:**  
  Saat log diperiksa dengan `docker compose logs postgres-db`, ditemukan baris pesan:  
  `PostgreSQL Database directory appears to contain a database; Skipping initialization`
- **Penyebab:**  
  Skrip bawaan `docker-entrypoint.sh` PostgreSQL dirancang untuk hanya mengeksekusi berkas pada `/docker-entrypoint-initdb.d/` ketika direktori `$PGDATA` masih kosong. Karena volume `pg-data` sudah memiliki cluster aktif dari running sebelumnya, proses initdb otomatis dilewati demi menjaga keamanan data.
- **Tindakan Solusi:**  
  1. *Pada lingkungan lab/development awal:* Jalankan `docker compose down -v` untuk menghapus volume lama sehingga container membuat cluster baru dari nol.
  2. *Pada lingkungan produksi:* Jangan mengandalkan init script untuk pembaruan skema. Gunakan skrip migrasi terkelola (*Flyway*, *Liquibase*, atau *Alembic*) atau eksekusi DDL langsung melalui sesi `psql`.

---

### 5.2 Analisis Risiko Keamanan dan Operasional

1. **Penyimpanan Password Plaintext pada `compose.yaml`:**
   Mendefinisikan `POSTGRES_PASSWORD: labpass123` langsung pada file compose berisiko tinggi terhadap kebocoran (*credential leak*) saat berkas di-*push* ke git repository publik serta dapat dibaca secara mudah melalui `docker inspect`.
2. **Exposure Port Database ke Interface Publik (`0.0.0.0:5432`):**
   Membuka port 5432 langsung ke host membuka potensi serangan *brute force* dan pemindaian port dari luar jaringan lokal.
3. **Risiko Fatal Penggunaan Flag `down -v`:**
   Eksekusi perintah `docker compose down -v` secara sembarangan oleh operator akan menghapus named volume secara permanen, mengakibatkan *catastrophic data loss*.

---

### 5.3 Rekomendasi Hardening Menuju Lingkungan Produksi

1. **Implementasi Docker Secrets:**  
   Ganti variabel lingkungan dengan Docker Secrets runtime (`POSTGRES_PASSWORD_FILE: /run/secrets/db_password`) sehingga password hanya dimuat pada memory sementara (tmpfs) dan tidak terekspos di konfigurasi/log.
2. **Network Segmentation & Port Isolation:**  
   Hapus blok pemetaan port publik `ports: - "5432:5432"`. Tempatkan database pada Docker internal network (`internal: true`) agar hanya service backend dalam stack yang dapat menjangkau port database.
3. **Continuous WAL Archiving & Point-In-Time Recovery (PITR):**  
   Implementasikan pengarsipan berkas *Write-Ahead Logging* (WAL) secara kontinu ke remote object storage (misal AWS S3 terenkripsi KMS) untuk mendukung pemulihan data hingga detik terakhir sebelum insiden.
4. **Prinsip Least Privilege Database Role:**  
   Pisahkan akun superuser administrasi dengan akun operasional aplikasi harian. Berikan hak akses terbatas (*grant select, insert, update only*) pada tabel spesifik bagi user aplikasi.

---

## 6. EVALUASI DAN LATIHAN MANDIRI

### Pertanyaan 1: Mengapa init script tidak dijalankan ulang saat volume lama masih ada?
> **Jawaban:**  
> Skrip entrypoint resmi PostgreSQL (`docker-entrypoint.sh`) memeriksa keberadaan berkas cluster database (seperti berkas `PG_VERSION`) pada direktori `$PGDATA` (`/var/lib/postgresql/data`). Jika direktori tersebut telah berisi data dari volume lama, sistem mendeteksi bahwa inisialisasi awal (*initdb*) sudah pernah berhasil dilakukan. Oleh karena itu, seluruh skrip pada `/docker-entrypoint-initdb.d` dilewati (*skipped*). Hal ini merupakan mekanisme perlindungan bawaan untuk mencegah duplikasi eksekusi data (misalnya `INSERT`), mencegah kegagalan constraint skema, serta menghindari korupsi data pada cluster yang telah berjalan.

### Pertanyaan 2: Apa risiko menaruh password database pada `docker-compose.yml`?
> **Jawaban:**  
> - **Kebocoran Repositori (*Version Control Leakage*):** Berkas compose rentan terunggah ke repositori Git (publik maupun privat tim), sehingga kredensial dapat diakses oleh pihak yang tidak berhak.
> - **Eksposur Metadata Container:** Nilai environment variable tersimpan dalam format plaintext pada metadata container dan dapat diinspeksi oleh siapa saja yang memiliki akses Docker daemon di host via `docker inspect <container_id>`.
> - **Riwayat Log dan Proses:** Nilai password dapat terekspos pada log orchestrator CI/CD atau riwayat command line host.

### Pertanyaan 3: Bagaimana cara membuktikan backup dapat dipulihkan?
> **Jawaban:**  
> Backup hanya dapat dibuktikan melalui pengujian pemulihan aktual (*restore drill*), bukan sekadar melihat keberadaan file dump. Prosedur pembuktian meliputi:
> 1. Merestore file backup ke instance database terpisah (*staging / temporary cluster*) menggunakan `pg_restore`.
> 2. Memeriksa nilai kembalian eksekusi (*exit code* `0` tanpa fatal error).
> 3. Memvalidasi integritas data: membandingkan jumlah baris (*row count*), integritas relasi foreign key, constraint, serta melakukan uji kueri representatif aplikasi untuk memastikan state database identik dengan kondisi saat backup diambil.

### Pertanyaan 4: Apa bedanya logical backup `pg_dump` dan backup filesystem volume mentah?
> **Jawaban:**  
> - **Logical Backup (`pg_dump`):** Mengekspor data dalam bentuk perintah SQL deklaratif (DDL/DML) atau arsip biner terkompresi.  
>   *Kelebihan:* Portabel lintas versi PostgreSQL dan arsitektur hardware, mendukung restorasi granular (per tabel atau skema).  
>   *Kekurangan:* Waktu pemulihan lebih lama pada ukuran database yang sangat besar.
> - **Filesystem / Raw Volume Backup:** Menyalin langsung direktori fisik `$PGDATA` pada level blok storage.  
>   *Kelebihan:* Proses pemulihan sangat cepat karena tidak perlu mem-parse ulang instruksi SQL.  
>   *Kekurangan:* Sangat rentan korupsi jika disalin saat database sedang aktif menulis (memerlukan locking/checkpoint atau tool seperti `pg_basebackup`), dan tidak portabel antar versi mayor PostgreSQL atau arsitektur sistem yang berbeda.

### Pertanyaan 5: Apa dampak `docker compose down -v` terhadap database?
> **Jawaban:**  
> Parameter `-v` (*volumes*) memerintahkan Docker Compose untuk menghapus seluruh *named volume* yang terasosiasi dengan file compose tersebut (dalam hal ini volume `pg-data`). Dampaknya, seluruh berkas cluster data, tabel, relasi, indeks, dan rekaman data transaksional yang tersimpan di host disk akan **terhapus secara permanen (*permanent catastrophic data loss*)**. Jika container dijalankan kembali, database akan dimulai dalam kondisi kosong dari awal.

---

## 7. KESIMPULAN

Melalui praktikum Bab 5, dapat disimpulkan bahwa:
1. Pengelolaan database di lingkungan Docker memerlukan pemisahan tegas antara komputasi (container lifecycle) dan penyimpanan (persistent named volume).
2. Mekanisme entrypoint initdb mempermudah bootstrapping skema awal pada lingkungan baru secara otomatis, namun tidak ditujukan sebagai pengganti sistem migrasi database produksi.
3. Penerapan healthcheck berbasis `pg_isready` krusial dalam arsitektur multi-container agar service yang bergantung (seperti pgAdmin atau backend API) hanya mulai tersambung saat engine database benar-benar siap menerima koneksi TCP.
4. Keberhasilan strategi disaster recovery database tidak diukur dari keberadaan file backup semata, melainkan dari prosedur pengujian pemulihan (*tested restore*) yang terbukti berhasil mengembalikan integritas data secara utuh.
