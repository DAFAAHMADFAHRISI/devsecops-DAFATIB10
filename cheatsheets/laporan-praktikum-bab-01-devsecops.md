# LAPORAN PRAKTIKUM BAB 1
## Fondasi Teoretis dan Kerangka Kerja DevSecOps

**Nama**: ........................................................  
**NIM**: ..........................................................  
**Kelas**: ........................................................  
**Tanggal pelaksanaan**: 15 September 2026  

## 1. Tujuan Praktikum

Praktikum Bab 1 bertujuan menetapkan baseline laboratorium DevSecOps dan memahami verifikasi awal yang diperlukan sebelum eksperimen berikutnya. Baseline mencakup struktur direktori kerja, versi perangkat, akses terhadap Docker Engine dan Docker Compose v2, serta informasi mekanisme keamanan yang tersedia pada host.

Praktikum ini juga bertujuan melatih cara membaca hasil verifikasi secara kritis. Keluaran perintah tidak langsung dianggap sebagai bukti keamanan menyeluruh. Setiap hasil harus dikaitkan dengan konteks, versi perangkat, keterbatasan lingkungan, dan risiko yang mungkin timbul.

## 2. Dasar Teori Singkat

DevSecOps adalah pendekatan sosio-teknis yang mengintegrasikan keamanan ke dalam seluruh siklus hidup perangkat lunak, mulai dari perencanaan, pengembangan, build, pengujian, rilis, deployment, sampai operasi dan monitoring. DevSecOps bukan sekadar menambahkan security scanner ke dalam pipeline CI/CD.

Prinsip *shift-left* menempatkan pemeriksaan keamanan sedini mungkin, misalnya melalui secure coding, secret scanning, SAST, SCA, dan threat modeling. Prinsip *shift-right* melengkapi pemeriksaan tersebut melalui monitoring, runtime detection, DAST, verifikasi deployment, dan pembelajaran pascainsiden. Keduanya diperlukan karena pemeriksaan statis tidak dapat menggambarkan seluruh kondisi runtime.

Baseline diperlukan agar eksperimen dapat direproduksi. Versi operating system, Git, Docker, Compose, OpenSSL, cURL, konfigurasi daemon, dan mekanisme keamanan host harus dicatat. Tanpa informasi tersebut, perbedaan hasil antarpraktikan sulit ditelusuri.

## 3. Alat dan Lingkungan

| Komponen | Hasil identifikasi |
|---|---|
| Operating system | Ubuntu 24.04.3 LTS |
| Kernel | Linux 6.18.44, x86_64 |
| Pengguna eksekusi | `root` |
| Direktori kerja | `/workspace/scratch/25f72c6b6644` |
| Git | 2.51.1 |
| OpenSSL | 3.0.13, 30 Januari 2024 |
| cURL | 8.5.0 |
| Docker Engine | Tidak tersedia (`docker: command not found`) |
| Docker Compose | Tidak dapat diverifikasi karena Docker tidak tersedia |

Catatan: penggunaan akun `root` hanya menggambarkan akun pada lingkungan pengujian ini. Untuk laboratorium nyata, penggunaan akun non-root lebih sesuai dengan prinsip *least privilege*.

## 4. Langkah Praktikum

### 4.1 Membuat Struktur Direktori

Perintah yang digunakan:

```bash
mkdir -p ~/devsecops-lab/{app,policy,reports,sbom,keys}
cd ~/devsecops-lab
```

Pada lingkungan pengujian ini, direktori dibuat pada lokasi kerja berikut:

```text
/workspace/scratch/25f72c6b6644/devsecops-lab/
├── app/
├── keys/
├── policy/
├── reports/
├── sbom/
```

### 4.2 Mencatat Versi Perangkat

Perintah verifikasi:

```bash
docker version
docker compose version
git --version
openssl version
curl --version
docker info --format '{{json .SecurityOptions}}'
```

### 4.3 Verifikasi Direktori dan Permission

Perintah tambahan yang digunakan:

```bash
find /workspace/scratch/25f72c6b6644/devsecops-lab \
  -maxdepth 1 -mindepth 1 -type d -printf '%f\n' | sort

stat -c '%A %n' \
  /workspace/scratch/25f72c6b6644/devsecops-lab/reports \
  /workspace/scratch/25f72c6b6644/devsecops-lab/sbom \
  /workspace/scratch/25f72c6b6644/devsecops-lab/keys
```

## 5. Hasil Pengujian

### 5.1 Hasil Perintah Utama

| Pemeriksaan | Hasil aktual | Status |
|---|---|---|
| Struktur direktori lab | `app`, `policy`, `reports`, `sbom`, dan `keys` berhasil dibuat | Terpenuhi |
| `docker version` | `docker: command not found` | Gagal/terblokir |
| `docker compose version` | Tidak dapat dijalankan karena Docker tidak tersedia | Gagal/terblokir |
| `git --version` | `git version 2.51.1` | Terpenuhi |
| `openssl version` | `OpenSSL 3.0.13 30 Jan 2024` | Terpenuhi |
| `curl --version` | `curl 8.5.0` | Terpenuhi |
| `docker info ... SecurityOptions` | `docker: command not found` | Gagal/terblokir |
| Direktori `reports`, `sbom`, `keys` tersedia | Ketiga direktori tersedia | Terpenuhi sebagian |
| Permission direktori | `drwxr-xr-x` | Teridentifikasi; perlu hardening |
| Direktori bukan web root | Berada di bawah workspace, tetapi konfigurasi web server tidak diperiksa | Belum terverifikasi |

### 5.2 Bukti Output

Output perangkat yang tersedia:

```text
git version 2.51.1
OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024)
curl 8.5.0 (... OpenSSL/3.0.13 ...)
```

Output pemeriksaan Docker:

```text
/bin/bash: line 1: docker: command not found
```

Output permission direktori:

```text
drwxr-xr-x /workspace/scratch/25f72c6b6644/devsecops-lab/reports
drwxr-xr-x /workspace/scratch/25f72c6b6644/devsecops-lab/sbom
drwxr-xr-x /workspace/scratch/25f72c6b6644/devsecops-lab/keys
```

## 6. Threat Statement

**Aset yang dilindungi** adalah source code, konfigurasi pipeline, laporan hasil scan, SBOM, image container, dan material kunci kriptografi laboratorium. **Aktor ancaman** dapat berupa pengguna lokal yang tidak berwenang, proses berbahaya pada host, penyerang yang memperoleh akses ke repository, atau pihak yang menyalahgunakan kredensial pipeline. **Jalur serangan** meliputi pencurian secret, perubahan source code, penyisipan dependensi berbahaya, manipulasi image, penyalahgunaan hak akses Docker, dan publikasi tidak sengaja terhadap laporan atau kunci. **Dampak** yang mungkin terjadi adalah kebocoran kredensial, artefak yang tidak tepercaya, kegagalan deployment, kompromi host, dan hilangnya integritas evidence.

Risiko terbesar pada tahap baseline adalah belum tersedianya Docker, penggunaan akun `root`, serta permission direktori `755` yang masih memungkinkan pengguna lain membaca isi direktori apabila file di dalamnya tidak dikonfigurasi secara ketat. Untuk direktori `keys`, praktik yang lebih aman adalah membatasi akses sesuai kebutuhan, menyimpan secret di secret manager, dan tidak memasukkan kunci nyata ke repository.

## 7. Analisis

Struktur direktori berhasil dibuat sehingga organisasi awal artefak praktikum sudah tersedia. Pemisahan `reports`, `sbom`, dan `keys` membantu membedakan evidence, inventaris komponen, dan material sensitif. Namun, keberadaan direktori saja belum membuktikan bahwa data aman. Verifikasi berikutnya harus memastikan tidak ada web server yang mengekspos direktori tersebut, serta memastikan permission file dan mekanisme penyimpanan secret telah ditetapkan.

Git, OpenSSL, dan cURL dapat digunakan sebagai komponen pendukung praktikum. Git diperlukan untuk version control dan penelusuran perubahan. OpenSSL diperlukan untuk pemeriksaan atau pembuatan material kriptografi pada eksperimen yang relevan. cURL mendukung pengujian komunikasi HTTP/HTTPS secara terkontrol.

Docker Engine tidak tersedia sehingga tiga verifikasi penting belum dapat dilakukan: akses ke Docker daemon, penggunaan Compose v2, dan identifikasi `SecurityOptions`. Kondisi ini menghalangi pelaksanaan eksperimen container pada tahap berikutnya. Instalasi Docker perlu dilakukan melalui prosedur resmi pada host atau VM khusus laboratorium. Setelah instalasi, seluruh pemeriksaan harus diulang dan hasilnya dicatat bersama versi Docker Engine, versi Compose, konfigurasi rootless/rootful, serta opsi keamanan host.

Hasil `SecurityOptions`, apabila tersedia nanti, hanya menunjukkan mekanisme keamanan yang tersedia pada Docker daemon/host. Hasil tersebut bukan bukti bahwa semua container telah di-hardening. Hardening tetap memerlukan pemeriksaan image, user container, Linux capabilities, seccomp, AppArmor/SELinux, filesystem, secret, network, resource limit, dan akses terhadap Docker socket.

## 8. Tindak Lanjut

1. Menyediakan Linux host atau VM khusus laboratorium.
2. Menginstal Docker Engine dan Docker Compose plugin v2 dari repository resmi.
3. Mengulang `docker version`, `docker compose version`, `docker info`, dan `SecurityOptions`.
4. Menggunakan akun praktikum non-root dan menerapkan *least privilege*.
5. Menetapkan permission lebih ketat untuk direktori `keys` setelah kebutuhan akses ditentukan.
6. Membuktikan bahwa direktori laporan, SBOM, dan kunci tidak diekspos oleh web server.
7. Menulis hasil baseline ke repository Git dengan menghapus atau menyamarkan secret sensitif.

## 9. Kesimpulan

Praktikum Bab 1 berhasil menetapkan sebagian baseline laboratorium. Struktur direktori, versi Git, OpenSSL, dan cURL berhasil diverifikasi. Docker Engine belum tersedia sehingga verifikasi Docker, Compose v2, dan `SecurityOptions` belum lulus. Hasil ini menunjukkan bahwa verifikasi praktikum harus dilaporkan berdasarkan bukti aktual dan tidak boleh digantikan oleh output contoh.

Baseline dapat dinyatakan lengkap setelah Docker tersedia, pemeriksaan diulang, dan keamanan direktori serta penggunaan akun praktikum diperbaiki. Dengan baseline yang lengkap, eksperimen pada bab berikutnya dapat dilaksanakan secara lebih konsisten, aman, dan dapat direproduksi.

## 10. Referensi

1. Ferry Astika Saputra, “Bab 1 — Fondasi Teoretis dan Kerangka Kerja DevSecOps,” repository DevSecOps PENS, `bab-01.md`, diakses 15 September 2026: https://github.com/ferryas-pens/devsecops/blob/main/bab-01.md
2. NIST SP 800-218, *Secure Software Development Framework (SSDF) Version 1.1*.
3. OWASP, *DevSecOps Guideline*.
