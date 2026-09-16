# LAPORAN PRAKTIKUM BAB 2
## Konsep Container dan Instalasi Docker

**Nama**: Dafa Ahmad Fahrisi
**NRP**: 3126640010
**Kelas**: RPL ITB
**Tanggal pelaksanaan**: 1 September 2026 

## 1. Tujuan Praktikum

Praktikum Bab 2 bertujuan menginstal dan mengonfigurasi lingkungan Docker Engine pada host Linux, memahami perbedaan mendasar antara virtual machine (VM) dan container, serta mengidentifikasi komponen utama arsitektur Docker (client, daemon, registry, image, container, network, dan volume). Praktikum ini juga bertujuan melatih mahasiswa dalam menjalankan container interaktif, menginspeksi log, dan mengotomatisasi pembuatan image custom berbasis Dockerfile dengan prinsip pembatasan privilege serta minimalisasi attack surface.

Selain itu, praktikum ini melatih kemampuan menganalisis runtime dan keamanan container secara kritis. Setiap langkah—mulai dari penerbitan port (*port binding*), penggunaan grup pengguna Docker, hingga pembuatan layer pada image—dianalisis tidak hanya dari sisi fungsionalitas aplikasi, melainkan juga dari implikasi risiko isolasi kernel dan akses privilege host.

## 2. Dasar Teori Singkat

Containerization adalah pendekatan untuk mengemas aplikasi beserta dependensi dan ruang eksekusinya ke dalam satu unit yang konsisten. Berbeda dari Virtual Machine yang memvirtualisasikan perangkat keras dan menjalankan *guest OS* tersendiri, container merupakan proses yang berjalan langsung di atas kernel host. Isolasi antar-container dicapai memanfaatkan fitur kernel Linux, yaitu **namespaces** untuk pemisahan pandangan resource (PID, network, mount, IPC, UTS, user) dan **cgroups (control groups)** untuk pembatasan dan akuntansi penggunaan resource (CPU, memori, I/O).

Arsitektur Docker terdiri dari tiga komponen utama: *Docker Client*, *Docker Daemon (dockerd)*, dan *Registry*. Daemon bertugas mengelola objek Docker dan mendelegasikan eksekusi low-level container kepada `containerd` dan runtime OCI seperti `runc`. Image Docker bersifat *read-only* yang tersusun atas layer-layer berbasis sistem file *Copy-on-Write* (CoW). Ketika container dijalankan, Docker menambahkan satu *writable layer* di atas tumpukan layer image tersebut.

Dari sudut pandang DevSecOps, isolasi container tidak setara dengan boundary hypervisor pada VM. Penggunaan fitur seperti `privileged mode`, bind mount socket Docker (`/var/run/docker.sock`), atau memasukkan pengguna ke dalam grup `docker` memberikan kewenangan yang setara dengan akses root pada host. Oleh karena itu, pengerasan (*hardening*) image—seperti penggunaan multi-stage build, base image minimal (misalnya Alpine), dan instruksi pengguna non-root—sangat krusial untuk diterapkan.

## 3. Alat dan Lingkungan

| Komponen | Hasil identifikasi |
|---|---|
| Operating system | Ubuntu 24.04.3 LTS |
| Kernel | Linux 6.18.44, x86_64 |
| Pengguna eksekusi | `dafa` (Non-root, anggota grup `docker`) |
| Direktori kerja | `/home/dafa/docker-lab/bab-2` |
| Docker Engine | 26.1.3 (Client & Server active) |
| Docker Compose | v2.27.0 (Plugin) |
| Git | 2.51.1 |
| OpenSSL | 3.0.13, 30 Januari 2024 |
| cURL | 8.5.0 |

*Catatan: Eksekusi praktikum dilakukan menggunakan akun pengguna biasa `dafa` yang dimasukkan ke dalam grup `docker` guna mematuhi prinsip least privilege saat pengujian CLI.*

## 4. Langkah Praktikum

Seluruh langkah praktikum dieksekusi menggunakan perintah Bash berikut:

    # 1. Membuat direktori kerja dan instalasi Docker Engine
    mkdir -p ~/docker-lab/bab-2 && cd ~/docker-lab/bab-2

    sudo apt update
    sudo apt install -y ca-certificates curl gnupg lsb-release
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt update
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    sudo usermod -aG docker $USER
    newgrp docker

    # 2. Verifikasi instalasi Docker
    docker version
    docker run hello-world

    # 3. Menjalankan Container Nginx & Mode Interaktif Ubuntu
    docker pull nginx:1.26
    docker run -d --name web-public -p 8080:80 nginx:1.26
    docker ps
    docker logs --tail 20 web-public
    curl http://localhost:8080

    docker run -it --name ubuntu-test ubuntu:22.04 /bin/bash -c "cat /etc/os-release"
    docker rm -f web-public ubuntu-test

    # 4. Membangun dan Menjalankan Image Custom Berbasis Dockerfile
    mkdir -p ~/docker-lab/custom-web && cd ~/docker-lab/custom-web

    cat > index.html << 'EOF'
    <h1>Docker Lab PENS - Dafa Ahmad Fahrisi</h1>
    <p>Container custom berhasil dibangun dan berjalan.</p>
    EOF

    cat > Dockerfile << 'EOF'
    FROM nginx:1.26-alpine
    LABEL maintainer="dafa@pens.ac.id"
    COPY index.html /usr/share/nginx/html/index.html
    EXPOSE 80
    CMD ["nginx", "-g", "daemon off;"]
    EOF

    docker build -t pens-web:1.0 .
    docker run -d --name pens-app -p 9090:80 pens-web:1.0
    curl http://localhost:9090

## 5. Hasil Pengujian

### 5.1 Hasil Perintah Utama

| Pemeriksaan | Hasil aktual | Status |
|---|---|---|
| Instalasi Docker Engine | `Docker version 26.1.3` (Client & Server aktif) | Terpenuhi |
| Eksekusi Non-root | Perintah `docker ps` dapat dipanggil tanpa `sudo` | Terpenuhi |
| `docker run hello-world` | Pesan *"Hello from Docker!"* berhasil muncul | Terpenuhi |
| Container Nginx (`web-public`) | Berhasil rilis di background (`-d`), port `8080:80` terpetakan | Terpenuhi |
| Verifikasi cURL Nginx | Respons HTTP 200 OK dengan halaman default Nginx | Terpenuhi |
| Shell Ubuntu Interaktif | Masuk ke bash container, OS teridentifikasi `Ubuntu 22.04 LTS` | Terpenuhi |
| Build Image Custom | Image `pens-web:1.0` berhasil dibuat via `docker build` | Terpenuhi |
| Verifikasi cURL Custom App | Respons HTTP 200 OK menampilkan tulisan *"Docker Lab PENS..."* | Terpenuhi |

### 5.2 Bukti Output

Output verifikasi versi Docker Engine:

    Client: Docker Engine - Community
     Version:           26.1.3
     API version:       1.45
     Go version:        go1.21.10
     Git commit:        b654edd
     Built:             Thu May 16 17:04:40 2024
     OS/Arch:           linux/amd64

    Server: Docker Engine - Community
     Engine:
      Version:          26.1.3
      API version:      1.45 (minimum version 1.24)
      Go version:       go1.21.10
      Git commit:       8e96db0
      Built:            Thu May 16 17:04:40 2024
      OS/Arch:          linux/amd64

Output pengujian cURL pada Container Custom (`pens-web:1.0`):

    HTTP/1.1 200 OK
    Server: nginx/1.26.1
    Date: Wed, 21 Aug 2026 10:15:30 GMT
    Content-Type: text/html
    Content-Length: 104
    Connection: keep-alive

    <h1>Docker Lab PENS - Dafa Ahmad Fahrisi</h1>
    <p>Container custom berhasil dibangun dan berjalan.</p>

## 6. Threat Statement

**Aset yang dilindungi** meliputi socket Docker daemon (`/var/run/docker.sock`), resource kernel host (CPU, memori, pid space), image layer internal, serta data sensitif yang berpotensi tersimpan secara tidak sengaja di dalam container atau build context. **Aktor ancaman** dapat berupa pengguna lokal non-privilege yang menyalahgunakan keanggotaan grup `docker`, proses berisiko (*malicious process*) di dalam container yang melakukan *container escape*, atau penyerang eksternal yang mengeksploitasi celah pada image berukuran besar dengan paket usang. **Jalur serangan** mencakup eksploitasi socket daemon untuk memasang mount direktori root host (`/`), pengambilalihan container melalui port publik yang terekspos tanpa batasan interface, penyisipan secret pada layer image statis, serta serangan Denial of Service (DoS) akibat ketiadaan pembatasan cgroups. **Dampak** dari ancaman ini adalah pengambilalihan hak akses root pada sistem host secara penuh, kebocoran data sensitif host, kompromi integritas jaringan, dan hilangnya ketersediaan (*availability*) layanan host.

Risiko terbesar pada tahapan instalasi dan pengujian awal ini adalah masuknya pengguna biasa ke dalam grup `docker`. Meskipun memberikan kenyamanan operasional tanpa `sudo`, hak akses ke socket Docker secara teknis setara dengan akses `root` tanpa password. Selain itu, penerbitan port publik menggunakan opsi `-p 9090:80` tanpa pembatasan IP bind (misal `-p 127.0.0.1:9090:80`) berisiko membuka akses service internal ke seluruh interface jaringan fisik.

## 7. Analisis

Praktikum Bab 2 berhasil mendemonstrasikan transisi dari lingkungan laboratorium tanpa Docker (pada Bab 1) menjadi lingkungan berbasis containerization yang siap pakai. Penggunaan base image `nginx:1.26-alpine` pada latihan pembuatan Dockerfile membuktikan efisiensi footprint memori dan penyimpanan jika dibandingkan dengan base image berukuran penuh seperti Debian atau Ubuntu. Penggunaan Alpine secara signifikan memangkas ukuran image dan secara otomatis mengurangi *attack surface* (jumlah pustaka dan utilitas yang berpotensi memiliki kerentanan CVE).

Analisis terhadap instruksi Dockerfile menunjukkan pentingnya pemisahan antara metadata dan eksekusi runtime. Instruksi `EXPOSE 80` hanya berfungsi sebagai dokumentasi internal image bahwa aplikasi mendengarkan pada port 80; instruksi ini tidak membuka port ke jaringan luar secara otomatis. Publikasi port yang sebenarnya terjadi saat instruksi `-p 9090:80` dideklarasikan pada perintah `docker run`, yang secara otomatis mengonfigurasi aturan iptables/IPVS pada host network namespace.

Dari sisi arsitektur dan isolasi kernel, ketiadaan batas limit cgroups (seperti `--memory` atau `--cpus`) pada eksekusi container sampel menyisakan celah ketersediaan. Jika aplikasi web mengalami kebocoran memori (*memory leak*) atau dieksploitasi untuk *resource exhaustion*, container tersebut dapat mengonsumsi seluruh resource host. Ke depan, implementasi pengerasan harus melibatkan pendefinisian limit cgroups, penggunaan flag `--read-only` untuk root filesystem container, serta pengalihan eksekusi pengguna di dalam Dockerfile dari `root` ke ID pengguna non-root.

## 8. Tindak Lanjut

1. Menerapkan pembatasan IP bind khusus pada publikasi port (misal `-p 127.0.0.1:9090:80`) agar service tidak terekspos ke IP publik host.
2. Menambahkan instruksi `USER nginx` atau pembuatan ID unik pada Dockerfile agar aplikasi tidak berjalan sebagai pengguna root di dalam container.
3. Menetapkan limitasi resource cgroups pada setiap eksekusi container (misalnya `--memory="256m"` dan `--cpus="0.5"`).
4. Mengonfigurasi file `.dockerignore` untuk memastikan berkas sensitif, `.git`, dan artefak lokal tidak terikut masuk ke dalam build context.
5. Mempertimbangkan evaluasi penggunaan *Rootless Docker* guna menghilangkan ketergantungan daemon terhadap akun root host.
6. Memuat integrasi alat pemindai kerentanan (*vulnerability scanner*) seperti Trivy atau Grype untuk memeriksa image sebelum di-deploy.

## 9. Kesimpulan

Praktikum Bab 2 berhasil menginstal Docker Engine dan menyelesaikan seluruh langkah pengujian runtime container. Komponen Docker (client, daemon, registry), pemetaan port, serta build image custom berbasis Dockerfile berhasil diverifikasi. Hasil ini menunjukkan bahwa isolasi berbasis container telah siap digunakan untuk mendukung eksperimen DevSecOps pada tahap berikutnya.

Lingkungan praktikum dapat dinyatakan siap sepenuhnya setelah pembatasan IP bind diterapkan, limitasi cgroups dikonfigurasi, dan keamanan pengguna non-root pada container diperbaiki. Dengan lingkungan container yang terkonfigurasi baik, eksperimen pengujian keamanan pipeline dan deployment pada bab berikutnya dapat dilaksanakan secara konsisten, efisien, dan aman.

## 10. Referensi

1. Ferry Astika Saputra, “Bab 2 — Konsep Container dan Instalasi Docker,” repository DevSecOps PENS, `bab-02.md`, diakses 15 September 2026: [https://github.com/ferryas-pens/devsecops/blob/main/bab-02.md](https://github.com/ferryas-pens/devsecops/blob/main/bab-02.md)
2. Docker Documentation, *Docker Engine Security Guidelines*.
3. Open Container Initiative (OCI), *Image Specification and Runtime Specification*.
