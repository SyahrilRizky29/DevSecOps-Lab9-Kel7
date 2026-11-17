# Log Pengerjaan — Hardening Container & CI/CD Vulnerability Gate

**Tanggal:** 17 November 2026  
**Direktori Proyek:** `/home/devsec/aril-praktikum9/tugas2-secure-dockerfile`  
**Repository GitHub:** `https://github.com/SyahrilRizky29/DevSecOps-Lab9-Kel7.git` (branch `main`)

---

## 1. Audit Hardening Container & Checklist

Dilakukan pemeriksaan terhadap `Dockerfile`, `.dockerignore`, `app.py`, dan `requirements.txt`, serta pemindaian konfigurasi menggunakan `trivy config`:

- **Hasil `trivy config`:** `27` pengujian (`26` lulus, `1` temuan `LOW`: `DS-0026 — Add HEALTHCHECK instruction in your Dockerfile`, `0` `CRITICAL`, `0` `HIGH`, `0` `MEDIUM`).
- **Status 10 Item Hardening Checklist:**

| No | Item Hardening Checklist | Status | Bukti Implementasi |
|---|---|---|---|
| 1 | Use Minimal Base Image | Diterapkan | `FROM python:3.12-alpine` pada `Dockerfile` |
| 2 | Run as Non-Root User | Diterapkan | `appgroup` (GID `1001`), `appuser` (UID `1001`), dan `USER appuser` pada `Dockerfile` |
| 3 | Implement Multi-Stage Build | Diterapkan | Pemisahan stage `builder` dan `runtime` pada `Dockerfile` |
| 4 | Clean Package Manager Cache | Diterapkan | `pip install --no-cache-dir` dan `apk upgrade --no-cache` pada `Dockerfile` |
| 5 | No Hardcoded Secrets/Credentials | Diterapkan | Tidak ada credential pada `ENV` (`PORT`, `PYTHONDONTWRITEBYTECODE`, `PYTHONUNBUFFERED`) maupun `app.py` |
| 6 | Prevent Sensitive Leaks via `.dockerignore` | Diterapkan | Mengecualikan `.env`, `*.env`, `.git`, `.gitignore`, `__pycache__/`, `*.pyc`, `Dockerfile*`, `README.md` |
| 7 | Set Explicit Working Directory | Diterapkan | `WORKDIR /build` (builder) dan `WORKDIR /app` (runtime) |
| 8 | Limit Exposed Ports | Diterapkan | Hanya `EXPOSE 8080` |
| 9 | Use Exec Form (`ENTRYPOINT` + `CMD`) | Diterapkan | `ENTRYPOINT ["python3"]` dan `CMD ["app.py"]` |
| 10 | Automated Vulnerability Gate in CI/CD | Diterapkan | Pipeline `Jenkinsfile` di Jenkins dengan `trivy config` dan `trivy image` (`--exit-code 1 --severity CRITICAL,HIGH`) |

---

## 2. Instalasi Jenkins + Trivy via Docker Compose

Untuk memenuhi poin 10, disiapkan instance Jenkins dengan Docker CLI dan Trivy di `/home/minkwaq/jenkins-docker`:

1. **`/home/minkwaq/jenkins-docker/Dockerfile`**
   - Base image: `jenkins/jenkins:lts-jdk21`
   - Menginstal `docker-cli` (`26.1.5+dfsg1`) dan `trivy` (`0.76.2`) di dalam image.
   - *(Catatan: sempat mengganti paket `docker.io` menjadi `docker-cli` karena pada Debian Trixie binary client `docker` tersedia pada paket `docker-cli`.)*
2. **`/home/minkwaq/jenkins-docker/docker-compose.yml`**
   - Container name: `jenkins-trivy`
   - Port mapping: `8081:8080` (port `8080` host sudah digunakan layanan lain) dan `50000:50000`
   - Akses Docker daemon: `group_add: ["988"]` dan volume `/var/run/docker.sock:/var/run/docker.sock`
   - Zona waktu WIB: `TZ=Asia/Jakarta` dan `JAVA_OPTS=-Duser.timezone=Asia/Jakarta -Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Jakarta`
   - Volume persisten: `jenkins_home:/var/jenkins_home`
3. **Verifikasi Container:**
   - `docker --version`, `trivy --version`, akses `/var/run/docker.sock`, dan waktu sistem `WIB` di dalam container `jenkins-trivy` terverifikasi berjalan normal.

---

## 3. Pembuatan `Jenkinsfile` & Integrasi GitHub SCM

1. **Pembuatan `Jenkinsfile`** di `/home/devsec/aril-praktikum9/tugas2-secure-dockerfile/Jenkinsfile` dengan alur:
   - **Stage `Build Docker Image`:** `docker build --network=host -t ${IMAGE_NAME}:${IMAGE_TAG} .` *(menggunakan `--network=host` setelah sebelumnya `apk upgrade --no-cache` mengalami hang pada jaringan bridge default).*
   - **Stage `Trivy Misconfiguration Scan`:** `trivy config --exit-code 1 --severity CRITICAL,HIGH .`
   - **Stage `Trivy Vulnerability Gate`:** `trivy image --exit-code 1 --severity CRITICAL,HIGH ${IMAGE_NAME}:${IMAGE_TAG}`
   - **Post `always`:** `docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true`
2. **Konfigurasi Job Jenkins (`tugas2-secure-dockerfile`):**
   - **Definition:** `Pipeline script from SCM` (`Git`)
   - **Repository URL:** `https://github.com/SyahrilRizky29/DevSecOps-Lab9-Kel7.git`
   - **Branch Specifier:** `*/main`
   - **Script Path:** `Jenkinsfile`
   - **Trigger:** `Poll SCM` dengan jadwal `* * * * *` (pengecekan commit baru setiap 1 menit)
3. **Dokumentasi Tambahan:**
   - Menambahkan `README.md` di `/home/devsec/aril-praktikum9/tugas2-secure-dockerfile/README.md` untuk dokumentasi proyek sekaligus pengujian pemicu otomatis setelah `git push`.

---

## 4. Ringkasan Alur Kerja Vulnerability Gate

- Setiap `git push` ke branch `main` di GitHub akan dideteksi oleh Jenkins melalui `Poll SCM`.
- Jenkins menarik kode terbaru, membangun Docker image, lalu menjalankan `trivy config` dan `trivy image` dengan `--exit-code 1 --severity CRITICAL,HIGH`.
- Jika ditemukan misconfiguration atau CVE berlevel `CRITICAL`/`HIGH`, pipeline otomatis berstatus `FAILURE` untuk mencegah image yang rentan lolos ke tahap distribusi/deploy.
