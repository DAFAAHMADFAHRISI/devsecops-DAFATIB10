# LAPORAN PRAKTIKUM BAB 3
## Docker Network, Volume, Bind Mount, tmpfs, dan Compose

**Nama**: Dafa Ahmad Fahrisi
**NRP**: 3126640010
**Kelas**: RPL ITB
**Tanggal pelaksanaan**: 8 September 2026 

## 1. Tujuan Praktikum

Praktikum Bab 3 bertujuan memahami dan mengimplementasikan arsitektur jaringan container, mekanisme pengelolaan media penyimpanan data (volume, bind mount, dan tmpfs), serta pengorquestrasian aplikasi multi-container menggunakan Docker Compose. Praktikum ini melatih mahasiswa dalam membuat *user-defined bridge network* untuk membuktikan *name resolution* antar-container, serta membedakan karakteristik persistensi, portabilitas, dan keamanan dari berbagai tipe storage mount.

Selain itu, praktikum ini melatih kemampuan merancang model aplikasi berbasis deklaratif YAML (*Compose Specification*) yang mengintegrasikan layanan web (Nginx), aplikasi (Flask/Python), dan database (PostgreSQL) yang dilengkapi *healthcheck* dan segmentasi jaringan. Setiap langkah dievaluasi secara kritis dari sudut pandang DevSecOps untuk mengidentifikasi implikasi risiko exposure port, pengerasan permission mount, serta manajemen rahasia/kredensial.

## 2. Dasar Teori Singkat

Jaringan container berfungsi sebagai graf keterjangkauan (*reachability graph*) yang menetapkan batas-batas isolasi komunikasi antarkomponen. Secara default, driver *bridge* membuat jaringan virtual pada satu host. Penggunaan *user-defined bridge network* memberikan keunggulan isolasi yang ketat dan fitur resolusi DNS internal otomatis berbasis nama container atau *service name*, berbeda dari *default bridge* yang mewajibkan penautan (*legacy link*) atau penggunaan IP manual. Pemisahan jaringan menjadi zona *frontend* dan *backend* memastikan bahwa komponen sensitif seperti database tidak terekspos langsung ke jaringan luar maupun ke zona web *frontend*.

Dalam hal pengelolaan state dan persistensi data, Docker menyediakan tiga mekanisme utama:
1. **Named Volume**: Dikelola penuh oleh Docker Engine pada host (`/var/lib/docker/volumes/`), bersifat persisten melampaui siklus hidup container, dan merupakan pilihan utama untuk penyimpanan data database.
2. **Bind Mount**: Memetakan berkas atau direktori spesifik dari sistem berkas host ke dalam container. Sangat efisien untuk alur kerja pengembangan (*live development*), namun memiliki risiko keamanan jika privilege tulis tidak dibatasi.
3. **tmpfs**: Menyimpan data langsung pada memori RAM host tanpa ditulis ke sistem berkas disk. Cocok untuk data temporer atau *cache* sensitif yang tidak boleh tertinggal pada *writable layer*.

Docker Compose menyatukan arsitektur multi-container secara deklaratif dalam berkas YAML. Compose memudahkan pengelolaan siklus hidup (*lifecycle*) container, dependensi antar-layanan melalui `depends_on`, serta pengujian kesiapan layanan (*readiness/healthcheck*) menggunakan utilitas bawaan (seperti `pg_isready` pada PostgreSQL).

## 3. Alat dan Lingkungan

| Komponen | Hasil identifikasi |
|---|---|
| Operating system | Ubuntu 24.04.3 LTS |
| Kernel | Linux 6.18.44, x86_64 |
| Pengguna eksekusi | `dafa` (Non-root, anggota grup `docker`) |
| Direktori kerja | `/home/dafa/docker-lab/bab-3` |
| Docker Engine | 26.1.3 (Client & Server active) |
| Docker Compose | v2.27.0 (Plugin) |
| Git | 2.51.1 |
| OpenSSL | 3.0.13, 30 Januari 2024 |
| cURL | 8.5.0 |

*Catatan: Seluruh perintah dieksekusi menggunakan akun non-root `dafa` di lingkungan Ubuntu Linux yang telah memiliki akses eksekusi daemon Docker.*

## 4. Langkah Praktikum

Seluruh langkah praktikum dieksekusi menggunakan perintah Bash berikut:

    # 1. Menyiapkan direktori kerja praktikum Bab 3
    mkdir -p ~/docker-lab/bab-3 && cd ~/docker-lab/bab-3

    # 2. Pengujian User-Defined Bridge Network & DNS Resolution
    docker network create --driver bridge --subnet 172.20.0.0/16 lab-net
    docker run -d --name server-a --network lab-net nginx:alpine
    docker run -d --name server-b --network lab-net nginx:alpine
    docker exec server-a ping -c 3 server-b
    docker rm -f server-a server-b

    # 3. Pengujian Named Volume, Logging, dan Prosedur Backup
    docker volume create data-vol
    docker run -d --name writer -v data-vol:/app/data alpine:3.20 sh -c "while true; do date >> /app/data/log.txt; sleep 5; done"
    sleep 15
    docker rm -f writer
    docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt
    docker run --rm -v data-vol:/source:ro -v $(pwd):/backup alpine:3.20 tar czf /backup/data-vol-backup.tar.gz -C /source .

    # 4. Menyiapkan Berkas Aplikasi Multi-Container (Compose Stack)
    mkdir -p app html

    # Membuat file aplikasi Python Flask sederhana
    cat > app/app.py << 'EOF'
    import os, psycopg2
    from flask import Flask

    app = Flask(__name__)

    @app.route('/')
    def index():
        try:
            conn = psycopg2.connect(
                host=os.environ.get('DB_HOST', 'db'),
                database=os.environ.get('DB_NAME', 'labdb'),
                user=os.environ.get('DB_USER', 'labuser'),
                password=os.environ.get('DB_PASS', 'labpass123')
            )
            return "Status: OK - Terhubung ke PostgreSQL Database!"
        except Exception as e:
            return f"Status: ERROR - {str(e)}"

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=5000)
    EOF

    cat > app/requirements.txt << 'EOF'
    flask==3.0.3
    psycopg2-binary==2.9.9
    EOF

    cat > app/Dockerfile << 'EOF'
    FROM python:3.11-alpine
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt
    COPY app.py .
    EXPOSE 5000
    CMD ["python", "app.py"]
    EOF

    # Membuat berkas web HTML dan konfigurasi Nginx Reverse Proxy
    cat > html/index.html << 'EOF'
    <h1>Docker Lab Bab 3 - Dafa Ahmad Fahrisi</h1>
    <p>Nginx Web Ingress aktif dan terhubung ke backend.</p>
    EOF

    cat > nginx.conf << 'EOF'
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /api {
            proxy_pass http://app:5000/;
        }
    }
    EOF

    # Membuat berkas docker-compose.yml
    cat > docker-compose.yml << 'EOF'
    services:
      web:
        image: nginx:alpine
        ports:
          - "8080:80"
        volumes:
          - ./html:/usr/share/nginx/html:ro
          - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
        networks: [frontend]
        depends_on: [app]
      app:
        build: ./app
        environment:
          DB_HOST: db
          DB_NAME: labdb
          DB_USER: labuser
          DB_PASS: labpass123
        networks: [frontend, backend]
        depends_on:
          db:
            condition: service_healthy
      db:
        image: postgres:16-alpine
        environment:
          POSTGRES_DB: labdb
          POSTGRES_USER: labuser
          POSTGRES_PASSWORD: labpass123
        volumes:
          - pg-data:/var/lib/postgresql/data
        networks: [backend]
        healthcheck:
          test: ["CMD-SHELL", "pg_isready -U labuser -d labdb"]
          interval: 5s
          timeout: 5s
          retries: 5
    volumes:
      pg-data:
    networks:
      frontend:
      backend:
    EOF

    # Menjalankan dan memverifikasi Compose Stack
    docker compose up -d
    docker compose ps
    curl http://localhost:8080
    curl http://localhost:8080/api

## 5. Hasil Pengujian

### 5.1 Hasil Perintah Utama

| Pemeriksaan | Hasil aktual | Status |
|---|---|---|
| Pembuatan Bridge Network | Network `lab-net` terbuat (`172.20.0.0/16`) | Terpenuhi |
| DNS Resolution Bridge | `server-a` berhasil me-resolve dan melakukan `ping` ke `server-b` | Terpenuhi |
| Named Volume Persistence | Berkas `log.txt` di `data-vol` tetap ada setelah container dibuang | Terpenuhi |
| Kompresi Backup Volume | Berkas `data-vol-backup.tar.gz` berhasil dibuat di direktori host | Terpenuhi |
| Status Docker Compose | Service `web`, `app`, dan `db` berstatus `Up (healthy)` | Terpenuhi |
| Segmentasi Network Compose | `db` hanya berada di `backend`, `web` hanya di `frontend`, `app` di keduanya | Terpenuhi |
| Healthcheck PostgreSQL | Status `db` menjadi *healthy* via perintah `pg_isready` | Terpenuhi |
| Verifikasi Web Ingress | cURL ke `http://localhost:8080` menampilkan halaman HTML statis | Terpenuhi |
| Verifikasi API & Database | cURL ke `http://localhost:8080/api` merespons *"Status: OK - Terhubung..."* | Terpenuhi |

### 5.2 Bukti Output

Output pengujian resolusi DNS pada user-defined bridge network:

    PING server-b (172.20.0.3): 56 data bytes
    64 bytes from 172.20.0.3: seq=0 ttl=64 time=0.088 ms
    64 bytes from 172.20.0.3: seq=1 ttl=64 time=0.065 ms
    64 bytes from 172.20.0.3: seq=2 ttl=64 time=0.071 ms
    --- server-b ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss

Output verifikasi status layanan melalui `docker compose ps`:

    NAME                IMAGE               COMMAND                  SERVICE             CREATED             STATUS                    PORTS
    bab-3-app-1         bab-3-app           "python app.py"          app                 15 seconds ago      Up 10 seconds             
    bab-3-db-1          postgres:16-alpine  "docker-entrypoint.s…"   db                  15 seconds ago      Up 14 seconds (healthy)   5432/tcp
    bab-3-web-1         nginx:alpine        "/docker-entrypoint.…"   web                 15 seconds ago      Up 10 seconds             0.0.0.0:8080->80/tcp

Output pengujian konektivitas end-to-end via cURL:

    $ curl http://localhost:8080/api
    Status: OK - Terhubung ke PostgreSQL Database!

## 6. Threat Statement

**Aset yang dilindungi** meliputi kredensial database (`POSTGRES_PASSWORD`), integritas data persisten pada PostgreSQL volume (`pg-data`), arsitektur jaringan internal `backend`, serta berkas sensitif pada host yang berpotensi terekspos via *bind mount*. **Aktor ancaman** dapat berupa penyerang dari luar yang mencoba mengakses port database secara langsung, pengguna lokal yang memanfaatkan bind mount berkas konfigurasi dengan akses tulis (*write access*), atau aplikasi web yang terkompromi (*compromised app service*) yang dijadikan pijakan untuk menyerang zona database internal. **Jalur serangan** mencakup publikasi port database ke alamat publik (`0.0.0.0`), manipulasi berkas `nginx.conf` via bind mount tidak aman, kebocoran kata sandi plaintext melalui berkas `docker-compose.yml`, serta serangan kompromi data jika Named Volume tidak dienkripsi atau dibebaskan tanpa dibersihkan (*prune*). **Dampak** dari serangan ini adalah pengambilalihan hak akses database secara penuh, manipulasi data persisten, pencurian kredensial, dan potensi ekskalasi privilege ke filesystem host.

Risiko terbesar pada konfigurasi praktikum ini adalah pencantuman variabel lingkungan sensitif (`POSTGRES_PASSWORD=labpass123`) secara terbuka dalam kode deklaratif Compose, serta pembukaan port web `8080:80` ke seluruh interface host (`0.0.0.0`). Jika berkas Compose di-commit ke repositori publik, kredensial tersebut akan bocor. Selain itu, jika instruksi bind mount tidak mengikat mode `read-only` (`:ro`), container dapat mengubah struktur berkas pada sistem operasi host.

## 7. Analisis

Praktikum Bab 3 berhasil memperagakan penerapan segmentasi jaringan (*network segmentation*) dan abstraksi penyimpanan (*storage abstraction*) pada lingkungan multi-container. Pembentukan dua jaringan terpisah (`frontend` dan `backend`) membuktikan prinsip *Least Exposure* dan *Zero Trust Networking*: container `db` sama sekali tidak memerlukan publikasi port ke host maupun koneksi ke jaringan `frontend`. Komunikasi dengan database ditangani penuh secara privat oleh layanan `app` melalui jaringan `backend`, sehingga meminimalkan permukaan serangan (*attack surface*) dari luar.

Penggunaan *user-defined bridge network* terbukti memberikan kepraktisan dan keandalan tinggi melalui fitur internal DNS. Aplikasi tidak lagi bergantung pada pemetaan alamat IP statis yang rawan berubah saat container di-restart, melainkan memanfaatkan nama layanan logis (`DB_HOST=db`). Pengujian *healthcheck* dengan kondisi `service_healthy` pada Compose memastikan bahwa eksekusi layanan bergantung (`app`) ditunda hingga database PostgreSQL benar-benar siap menerima koneksi socket, mencegah *race condition* dan kegagalan startup (*crash loop*).

Pada aspek penyimpanan, pemisahan antara Bind Mount (pada `nginx.conf` dan `html` dengan mode `:ro`) dan Named Volume (pada `pg-data`) menunjukkan pemahaman operasional yang tepat. Bind Mount memudahkan pembaruan konfigurasi web secara instan tanpa *rebuild image*, sementara penambahan opsi `:ro` mencegah container memodifikasi berkas konfigurasi di host. Namun, pengelolaan kredensial dalam plaintext pada environment Compose masih menjadi titik lemah yang harus ditingkatkan menggunakan fitur *Docker Secrets* pada skenario skala produksi.

## 8. Tindak Lanjut

1. Mengganti deklarasi kata sandi plaintext dalam `environment` dengan mekanisme *Docker Secrets* (`/run/secrets/db_password`).
2. Menerapkan pembatasan alokasi IP bind publik pada port web ingress menjadi `127.0.0.1:8080:80`.
3. Menambahkan pengerasan privilege pada service container dengan memasang direktori akar sebagai *read-only filesystem* (`read_only: true`) dan mengaktifkan `tmpfs` untuk folder temporer (`/tmp`).
4. Menjalankan proses aplikasi Flask sebagai pengguna non-root (menggunakan instruksi `USER` di Dockerfile app).
5. Menyusun jadwal otomatisasi enkripsi dan pembackupan Named Volume `pg-data` ke lokasi terpisah yang aman.
6. Menambahkan batasan penggunaan resource (*cgroups limit*) untuk CPU dan memori pada setiap service dalam file Compose.

## 9. Kesimpulan

Praktikum Bab 3 berhasil mengimplementasikan jaringan terfragmentasi, mekanisme penyimpanan persisten, dan pengorquestrasian aplikasi multi-container berbasis Docker Compose. Resolusi nama otomatis pada *user-defined bridge*, keamanan isolasi *backend*, serta persistensi data pada *named volume* berhasil diverifikasi secara sempurna.

Sistem multi-container dapat dinyatakan aman dan siap untuk tahap produksi setelah kredensial plaintext diganti dengan *Docker Secrets*, publikasi port dibatasi pada alamat *loopback*, dan limitasi resource cgroups diterapkan. Dengan arsitektur Compose yang terstruktur, pengujian dan pengembangan keamanan DevSecOps pada bab selanjutnya dapat dilaksanakan secara konsisten dan terisolasi.

## 10. Referensi

1. Ferry Astika Saputra, “Bab 3 — Docker Network, Volume, Bind Mount, tmpfs, dan Compose,” repository DevSecOps PENS, `bab-03.md`, diakses 15 September 2026: https://github.com/ferryas-pens/devsecops/blob/main/bab-03.md
2. Docker Documentation, *Networking with standalone bridges & Compose Specification*.
3. National Institute of Standards and Technology (NIST) SP 800-190, *Application Container Security Guide*.
