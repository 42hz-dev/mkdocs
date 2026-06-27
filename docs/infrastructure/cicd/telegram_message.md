# GitHub Actions CI/CD Telegram 알림 구축

## 개요

GitHub Actions 기반 CI/CD 환경에서 Build 및 Deploy 결과를 Telegram Bot을 통해 알림 받도록 구성한다.

배포 성공 여부를 GitHub Actions 화면에 접속하지 않고도 확인할 수 있도록 하며, 운영 환경에서 배포 상태를 빠르게 확인하는 것을 목표로 한다.

---

# 동작 구조

```text
Git Push
    │
    ▼
GitHub Actions
    │
    ├── Build Job
    │       │
    │       ├── Success
    │       │       │
    │       │       ▼
    │       │   Telegram 알림
    │       │
    │       └── Failed
    │               │
    │               ▼
    │          Telegram 알림
    │
    ▼
Deploy Job
    │
    ├── Success
    │       │
    │       ▼
    │   Telegram 알림
    │
    └── Failed
            │
            ▼
       Telegram 알림
```

---

# GitHub Secrets 설정

Telegram Bot 정보는 민감한 데이터이기 때문에 GitHub Secrets에 저장한다.

경로

```
Repository
 └── Settings
      └── Secrets and variables
            └── Actions
```

---

## Secrets

등록 항목

| Name               | Value              |
| ------------------ | ------------------ |
| TELEGRAM_BOT_TOKEN | Telegram Bot Token |
| TELEGRAM_CHAT_ID   | Telegram Chat ID   |

---

## TELEGRAM_BOT_TOKEN

Bot 생성 후 발급받은 Token

예시

```
123456789:AAxxxxxxxxxxxxxxxx
```

---

## TELEGRAM_CHAT_ID

메시지를 받을 Telegram 채팅 ID

예시

```
123456789
```

---

# Build Success 알림

Build Job 마지막 단계에 추가한다.

```yaml
- name: Telegram - Build Success
  if: success()
  run: |
    curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
    -d chat_id="${{ secrets.TELEGRAM_CHAT_ID }}" \
    -d parse_mode="Markdown" \
    -d text="🟢 *BUILD SUCCESS*

📦 Repository
${{ github.repository }}

🌿 Branch
${{ github.ref_name }}

👤 Actor
${{ github.actor }}

📝 Commit
${{ github.sha }}

💬 Message
${{ github.event.head_commit.message }}

⚙️ Workflow
${{ github.workflow }}

🔗 Run URL
${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

⏰ Time
$(date '+%Y-%m-%d %H:%M:%S')"
```

---

# Build Failed 알림

Build 실패 시 실행된다.

```yaml
- name: Telegram - Build Failed
  if: failure()
  run: |
    curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
    -d chat_id="${{ secrets.TELEGRAM_CHAT_ID }}" \
    -d parse_mode="Markdown" \
    -d text="🔴 *BUILD FAILED*

📦 Repository
${{ github.repository }}

🌿 Branch
${{ github.ref_name }}

👤 Actor
${{ github.actor }}

📝 Commit
${{ github.sha }}

💬 Message
${{ github.event.head_commit.message }}

⚙️ Workflow
${{ github.workflow }}

🔗 Run URL
${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

⏰ Time
$(date '+%Y-%m-%d %H:%M:%S')"
```

---

# Deploy Success 알림

Deploy Job 마지막 단계에 추가한다.

```yaml
- name: Telegram - Deploy Success
  if: success()
  run: |
    curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
    -d chat_id="${{ secrets.TELEGRAM_CHAT_ID }}" \
    -d parse_mode="Markdown" \
    -d text="🚀 *DEPLOY SUCCESS*

🌐 Site
https://mkdocs.42hzs.com

📦 Repository
${{ github.repository }}

🌿 Branch
${{ github.ref_name }}

👤 Actor
${{ github.actor }}

📝 Commit
${{ github.sha }}

💬 Message
${{ github.event.head_commit.message }}

⚙️ Workflow
${{ github.workflow }}

🔗 Run URL
${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

⏰ Time
$(date '+%Y-%m-%d %H:%M:%S')"
```

---

# Deploy Failed 알림

Deploy 실패 시 실행된다.

```yaml
- name: Telegram - Deploy Failed
  if: failure()
  run: |
    curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
    -d chat_id="${{ secrets.TELEGRAM_CHAT_ID }}" \
    -d parse_mode="Markdown" \
    -d text="🔥 *DEPLOY FAILED*

🌐 Site
https://mkdocs.42hzs.com

📦 Repository
${{ github.repository }}

🌿 Branch
${{ github.ref_name }}

👤 Actor
${{ github.actor }}

📝 Commit
${{ github.sha }}

💬 Message
${{ github.event.head_commit.message }}

⚙️ Workflow
${{ github.workflow }}

🔗 Run URL
${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

⏰ Time
$(date '+%Y-%m-%d %H:%M:%S')"
```

---

# Telegram 알림 예시

배포 성공 시 아래와 같은 형태로 수신된다.

```
🚀 DEPLOY SUCCESS

🌐 Site
https://mkdocs.42hzs.com

📦 Repository
42hz-dev/mkdocs

🌿 Branch
main

👤 Actor
42hz-dev

📝 Commit
7ab23cd

💬 Message
Add telegram notification

⚙️ Workflow
MkDocs CI/CD

🔗 Run URL
https://github.com/.../actions/runs/12345

⏰ Time
2026-06-27 21:10:22
```

---

# 구성 이유

## 왜 Telegram 알림을 추가했는가?

CI/CD 환경에서는 배포 성공 여부를 항상 GitHub Actions 화면에서 확인해야 하는 불편함이 있다.

Telegram Bot을 이용하면:

* 배포 성공 즉시 확인 가능
* 장애 발생 시 빠른 대응 가능
* 운영 상태 모니터링 가능

---

# 최종 CI/CD 구성

```text
GitHub Push

↓

GitHub Actions

↓

Build

↓

Artifact 생성

↓

Deploy

↓

rsync

↓

Nginx 반영

↓

Telegram Notification
```

현재 구성은 단순 자동 배포를 넘어, 배포 상태를 확인할 수 있는 운영 환경 형태로 구성하였다.
