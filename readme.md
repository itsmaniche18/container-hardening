# 🔒 Container Attack Surface Reduction
### A Measured CVE Reduction Study — PHP Laravel Base Image Hardening

![GitHub Actions](https://img.shields.io/badge/CI-GitHub_Actions-2088FF?logo=githubactions&logoColor=white)
![Trivy](https://img.shields.io/badge/Scanner-Trivy-00ADD8?logo=aqua&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Hardened-2496ED?logo=docker&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.2-777BB4?logo=php&logoColor=white)
![CVEs](https://img.shields.io/badge/Final_CVEs-0-brightgreen)

---

## 📋 Overview

This project investigates how **iterative Docker base image hardening** quantifiably reduces CVE exposure in a containerized PHP Laravel application — without modifying a single line of application source code.

Four Dockerfiles represent progressive hardening stages, each automatically built and scanned by **Trivy** via a **GitHub Actions** CI pipeline. Results are captured as downloadable artifacts on every push.

---

## 📊 Results

| Stage | Base Image | Total CVEs | Image Size | Change |
|-------|-----------|-----------|------------|--------|
| **Stage 1** — Baseline | `php:8.2-apache` (Debian 13.4) | **957** | 477.4 MB | — |
| **Stage 2** — Alpine Base | `php:8.2-fpm-alpine` | **2** | 80.8 MB | ↓ 99.8% CVEs |
| **Stage 3** — Multi-Stage Build | `php:8.2-fpm-alpine` | **2** | 80.8 MB | Build tools excluded |
| **Stage 4** — Hardened Final | `php:8.2-fpm-alpine` | **0** ✅ | 81.3 MB | 100% clean |

> All scans executed via GitHub Actions. Results stored as CI artifacts.  
> **957 CVEs → 0 CVEs | 477.4 MB → 81.3 MB**

---

## 🏗️ Repository Structure

```
container-attack-surface-reduction/
├── docker/
│   ├── Dockerfile.stage1        # Baseline: php:8.2-apache (Debian)
│   ├── Dockerfile.stage2        # Alpine Linux base
│   ├── Dockerfile.stage3        # Multi-stage build
│   └── Dockerfile.stage4        # Hardened: non-root, pinned packages
├── .github/
│   └── workflows/
│       └── scan.yml             # CI pipeline: build all 4 + Trivy scan
├── public/                      # Laravel web root
├── bootstrap/                   # Laravel bootstrap cache
├── storage/                     # Laravel storage (writable volume)
└── composer.json
```

---

## 🔄 CI/CD Pipeline

The GitHub Actions workflow automatically:

1. **Builds all 4 images in parallel** using a matrix strategy
2. **Scans each image** with Trivy (CRITICAL / HIGH / MEDIUM / LOW)
3. **Uploads scan reports** as downloadable artifacts (30-day retention)
4. **Reports image size** via `docker image inspect`

```yaml
strategy:
  fail-fast: false
  matrix:
    stage: [stage1, stage2, stage3, stage4]
```

Every push triggers a full scan across all stages. No manual steps required.

---

## 🛡️ Hardening Stages Explained

### Stage 1 — Baseline
Standard `php:8.2-apache` on Debian. Hundreds of pre-installed packages that are never used at runtime. **957 CVEs detected.**

### Stage 2 — Alpine Linux
Replaced Debian with `php:8.2-fpm-alpine`. Alpine Linux ships only the bare minimum, eliminating the vast majority of the OS vulnerability surface. **955 CVEs eliminated in a single step.**

### Stage 3 — Multi-Stage Build
Build-time tools (Composer, npm, gcc, git) are used in a separate builder stage and **never copied to the final image**. Only production artifacts reach the runtime layer. Architectural improvement that scales significantly with real dependency trees.

### Stage 4 — Hardened Runtime
- ✅ **Non-root user** — `appuser` (UID 1001), never runs as root
- ✅ **Pinned packages** — nginx locked to specific version (`1.28.3-r0`)
- ✅ **apk upgrade** — all available patches applied at build time
- ✅ **Volume declarations** — `storage/` and `bootstrap/cache/` declared as writable volumes, supporting `--read-only` at runtime
- ✅ **Unprivileged port** — serves on port 8080 (non-root cannot bind <1024)
- ✅ **0 CVEs detected**

---

## 🚀 Running Locally

### Prerequisites
- Docker
- [Trivy](https://aquasecurity.github.io/trivy/latest/getting-started/installation/)

### Build and scan any stage

```bash
# Clone the repo
git clone https://github.com/yourusername/container-attack-surface-reduction.git
cd container-attack-surface-reduction

# Build a specific stage
docker build -f docker/Dockerfile.stage1 -t laravel-stage1 .

# Scan with Trivy
trivy image --severity CRITICAL,HIGH,MEDIUM,LOW laravel-stage1

# Check image size
docker image inspect laravel-stage1 --format='{{.Size}}' | awk '{printf "%.1f MB\n", $1/1024/1024}'
```

### Run Stage 4 (hardened) with read-only filesystem

```bash
docker build -f docker/Dockerfile.stage4 -t laravel-stage4 .

docker run --read-only \
  -v laravel_storage:/var/www/html/storage \
  -v laravel_cache:/var/www/html/bootstrap/cache \
  -p 8080:8080 \
  laravel-stage4
```

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| [Trivy](https://github.com/aquasecurity/trivy) | Container vulnerability scanning |
| [Docker](https://docker.com) | Image builds |
| [GitHub Actions](https://github.com/features/actions) | CI/CD automation |
| PHP 8.2 / Laravel | Application under test |

---

## 📖 Key Takeaways

- **Base image choice is the highest-leverage security decision** in container hardening — switching from Debian to Alpine eliminated 99.8% of CVEs.
- **Multi-stage builds** are best practice for keeping build tools out of production images. Their CVE impact scales with the size of real dependency trees.
- **Package pinning** ensures reproducibility but requires active maintenance — Alpine drops old package versions when new ones are released.
- **Non-root containers** are a critical defence-in-depth measure; they limit the blast radius of any runtime exploit.
- **Automated scanning in CI** ensures vulnerabilities are caught at build time, not in production.

---

## 📄 Report

A full written project report (including methodology, results tables, and trade-off analysis) is available in the repository as [`Container_Attack_Surface_Report.docx`](./Container_Attack_Surface_Report.docx).

---

## 👤 Author

Built as a security research mini-project demonstrating practical DevSecOps techniques.  
Feel free to fork, adapt, and apply this methodology to your own container stacks.

---

*Scan results verified via GitHub Actions CI. All Dockerfiles are production-safe and can be adapted for real Laravel deployments.*
