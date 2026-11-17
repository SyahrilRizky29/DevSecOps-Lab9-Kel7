# Tugas 2 DevSecOps — Secure Dockerfile & CI/CD Vulnerability Gate

Aplikasi web Flask sederhana dengan container image yang sudah di-hardening dan terintegrasi dengan Jenkins + Trivy sebagai automated vulnerability gate.

## Struktur Proyek

- `app.py` — Aplikasi Flask (`/` dan `/health` pada port `8080`)
- `requirements.txt` — Dependensi Python dengan versi ter-pin
- `Dockerfile` — Multi-stage build berbasis `python:3.12-alpine` dengan non-root user (`appuser`)
- `.dockerignore` — Mengecualikan file sensitif dan file yang tidak diperlukan dari build context
- `Jenkinsfile` — Pipeline CI/CD untuk build image, `trivy config`, dan `trivy image` (`--exit-code 1 --severity CRITICAL,HIGH`)
