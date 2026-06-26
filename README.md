# 📚 42Hz Dev Docs

> MkDocs 기반 기술 문서와 GitHub Actions를 이용한 CI/CD 구축 프로젝트

## Overview

이 저장소는 **42Hz 기술 문서 사이트**를 관리하기 위한 Repository입니다.

문서는 **MkDocs(Material Theme)** 를 사용하여 작성하며, GitHub Actions를 통해 Build 및 Deploy를 자동화합니다.

---

## Tech Stack

- MkDocs
- Material for MkDocs
- GitHub Actions
- Nginx
- Ubuntu 22.04
- Cloudflare
- rsync
- SSH

---

## Architecture

```text
Git Push
    │
    ▼
GitHub Repository
    │
    ▼
GitHub Actions
    │
    ├── Build
    ├── Artifact
    └── Deploy
            │
            ▼
      SSH + rsync
            │
            ▼
Ubuntu Server
            │
            ▼
Nginx
            │
            ▼
Cloudflare
            │
            ▼
https://mkdocs.42hzs.com
```

---

## Features

- 📖 MkDocs 기반 기술 문서
- 🎨 Material Theme
- 🚀 GitHub Actions CI/CD
- 📦 Artifact 기반 배포
- 🔐 SSH Key 인증
- ⚡ rsync 증분 배포
- ☁️ Cloudflare 연동

---

## Project Structure

```text
.
├── docs/
├── .github/
│   └── workflows/
├── mkdocs.yml
└── README.md
```

---

## Local Development

```bash
pip install mkdocs-material
mkdocs serve
```

기본 주소

```
http://127.0.0.1:8000
```

---

## Build

```bash
mkdocs build --strict
```

---

## Deployment

배포는 **GitHub Actions**를 통해 자동으로 수행됩니다.

배포 과정

1. Build
2. Artifact 생성
3. SSH 인증
4. rsync 배포

---

## Documentation

모든 구축 과정은 MkDocs 문서에 기록됩니다.

- Infrastructure
- CI/CD
- Linux
- Docker
- Nginx
- Cloudflare
- Backend

---

## License

MIT License