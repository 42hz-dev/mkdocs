# CI/CD 구축 가이드 (GitHub Actions + MkDocs)

## 개요

본 문서는 MkDocs 프로젝트를 GitHub Actions를 이용하여 자동으로 배포하기 위한 CI/CD 구축 과정을 정리한 문서이다.

단순히 자동 배포를 구현하는 것이 아니라, 실제 운영 환경에서 사용하는 방식을 기준으로 설계하였다.

---

# 시스템 구성도

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
    ├── Artifact 생성
    └── Deploy
            │
            ▼
      SSH + rsync
            │
            ▼
Ubuntu Server
            │
            ▼
/var/www/mkdocs
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

# 사전 준비

프로젝트에서 사용한 환경

| 항목            | 내용                  |
| ------------- | ------------------- |
| OS            | Ubuntu 22.04        |
| Web Server    | Nginx               |
| Documentation | MkDocs              |
| Theme         | Material for MkDocs |
| Repository    | GitHub              |
| CI/CD         | GitHub Actions      |
| Deployment    | rsync               |
| DNS           | Cloudflare          |

---

# SSH Key 생성

GitHub Actions가 서버에 비밀번호 없이 접속하기 위해 SSH Key를 생성한다.

```bash
ssh-keygen -t ed25519 -C "github-actions-mkdocs"
```

저장 위치

```text
~/.ssh/github_actions_mkdocs
```

생성 결과

```text
github_actions_mkdocs
github_actions_mkdocs.pub
```

| 파일                        | 설명                |
| ------------------------- | ----------------- |
| github_actions_mkdocs     | 개인키 (Private Key) |
| github_actions_mkdocs.pub | 공개키 (Public Key)  |

---

# 서버에 Public Key 등록

공개키 확인

```bash
cat ~/.ssh/github_actions_mkdocs.pub
```

출력된 내용을 서버에 등록한다.

```text
/root/.ssh/authorized_keys
```

등록 후 권한 확인

```bash
chmod 600 ~/.ssh/authorized_keys
```

SSH 접속 테스트

```bash
ssh -i ~/.ssh/github_actions_mkdocs root@SERVER_IP
```

비밀번호를 묻지 않고 접속된다면 정상이다.

---

# GitHub Variables / Secrets 설정

Repository

```
Settings
    └── Secrets and variables
```

## Variables

민감하지 않은 정보를 저장한다.

| Name     | Value |
| -------- | ----- |
| HOST     | 서버 IP |
| USERNAME | root  |

추후 추가 예정

| Name        | Value           |
| ----------- | --------------- |
| DEPLOY_PATH | /var/www/mkdocs |
| SSH_PORT    | 22              |

---

## Secrets

외부에 노출되면 안 되는 정보를 저장한다.

| Name            | Value                           |
| --------------- | ------------------------------- |
| SSH_PRIVATE_KEY | github_actions_mkdocs 개인키 전체 내용 |

개인키 확인

```bash
cat ~/.ssh/github_actions_mkdocs
```

출력되는

```
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

전체를 등록한다.

> `.pub` 파일이 아니라 **Private Key**를 등록해야 한다.

---

# 처음 설계했던 방식

처음에는 Build와 Deploy를 분리하였다.

```
ci.yml
deploy.yml
```

구조

```text
Push

↓

CI

↓

Build

↓

Deploy
```

## 장점

* Build와 Deploy 역할이 명확
* 테스트하기 쉬움
* SSH 연결만 별도 검증 가능

## 단점

Deploy에서도 다시 Build를 수행해야 했다.

```text
CI

↓

mkdocs build

↓

Deploy

↓

mkdocs build
```

동일 작업이 두 번 수행되었다.

---

# Deploy를 workflow_dispatch로 시작한 이유

처음부터 자동 배포하지 않았다.

```yaml
on:
  workflow_dispatch:
```

이유

* SSH 연결 검증
* 권한 확인
* rsync 테스트
* 운영 서버 보호

자동 배포 전에 모든 기능을 검증하기 위함이다.

---

# rsync를 사용한 이유

정적 사이트는 변경된 파일만 복사하면 된다.

```bash
rsync -az --delete
```

장점

* 변경 파일만 복사
* 삭제된 파일 자동 삭제
* 매우 빠름
* 운영 환경에서 가장 많이 사용

scp와 비교

| rsync     | scp      |
| --------- | -------- |
| 변경 파일만 복사 | 전체 복사    |
| 삭제 동기화    | 불가능      |
| 속도 빠름     | 상대적으로 느림 |

---

# Artifact를 적용한 이유

처음 구조

```text
CI

↓

Build

↓

Deploy

↓

Build
```

Build를 두 번 수행하였다.

이를 개선하기 위해 Artifact를 사용하였다.

---

## Artifact란?

Build 결과물을 GitHub Actions가 보관하는 기능이다.

이번 프로젝트에서는

```
site/
```

디렉터리를 Artifact로 저장하였다.

```yaml
uses: actions/upload-artifact@v4
```

Deploy에서는

```yaml
uses: actions/download-artifact@v4
```

를 사용하여 동일한 결과물을 배포한다.

---

# Build Once, Deploy Many

Artifact를 사용하는 가장 큰 이유이다.

```text
Build

↓

Artifact

↓

Deploy

↓

Deploy

↓

Deploy
```

Build는 한 번만 수행하고

동일한 결과물을 여러 서버에 배포할 수 있다.

---

# workflow_run을 검토한 이유

처음에는

```
CI

↓

workflow_run

↓

Deploy
```

구조를 고려하였다.

하지만 GitHub Actions에서는 다른 Workflow의 Artifact를 사용하는 과정이 다소 복잡하다.

추가 API 호출 및 Artifact 조회가 필요하다.

---

# 최종 구조를 변경한 이유

최종적으로

```
mkdocs.yml
```

Workflow 하나로 통합하였다.

구조

```text
Git Push

↓

Workflow

↓

Build Job

↓

Artifact

↓

Deploy Job
```

Deploy는

```yaml
needs: build
```

를 사용하여 Build가 성공한 경우에만 실행된다.

---

# 최종 Workflow 구조

```text
Git Push
    │
    ▼
Build Job
    │
Checkout
    │
Python
    │
MkDocs Build
    │
Upload Artifact
    │
──────────────
needs: build
──────────────
    │
    ▼
Deploy Job
    │
Download Artifact
    │
SSH
    │
Known Hosts
    │
rsync
    │
Deploy 완료
```

---

# 최종 구조를 선택한 이유

기존

```
ci.yml

deploy.yml
```

↓

최종

```
mkdocs.yml
```

변경 이유

* Build를 한 번만 수행
* Artifact 재사용
* Deploy는 Build 성공 시에만 실행
* Workflow 관리가 쉬움
* 실무에서 많이 사용하는 구조
* 추후 Dev / Prod 환경 확장 가능

---

# 최종 프로젝트 구조

```
.github/
└── workflows/
    └── mkdocs.yml
```

---

# 이번 프로젝트에서 배운 점

* SSH Key 인증 기반 배포
* GitHub Variables와 Secrets의 차이
* GitHub Actions Workflow 구성
* Build와 Deploy의 역할 분리
* Artifact의 필요성
* Build Once, Deploy Many 개념
* rsync를 이용한 정적 사이트 배포
* GitHub Actions Job 간 의존성(needs)

---

# 향후 개선 예정

* Telegram 배포 성공/실패 알림
* Production / Development 환경 분리
* GitHub Environment 적용
* 배포 승인(Approval) 기능
* 배포 이력 관리
* Rollback 기능
* Cache 최적화
* Cloudflare Cache Purge 자동화

---

# 결론

이번 프로젝트는 단순한 MkDocs 자동 배포가 아니라, 실제 운영 환경을 고려한 GitHub Actions 기반 CI/CD 파이프라인을 구축하는 것을 목표로 하였다.

구축 과정에서 Build와 Deploy를 분리하여 각각의 역할을 검증하였으며, 이후 Artifact 기반으로 리팩토링하여 Build를 한 번만 수행하는 구조(Build Once, Deploy Many)로 개선하였다.

최종적으로 하나의 Workflow 안에서 Build Job과 Deploy Job을 분리하고 `needs`를 이용하여 의존성을 관리함으로써, 유지보수가 쉽고 확장 가능한 CI/CD 환경을 구성하였다.
