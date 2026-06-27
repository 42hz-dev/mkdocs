# Telegram Bot 확인 및 Chat ID 설정

## 개요

GitHub Actions CI/CD에서 Telegram 메시지 알림을 보내기 위해서는 아래 두 가지 값이 필요하다.

| 값 | 설명 |
|---|---|
| TELEGRAM_BOT_TOKEN | Telegram Bot 인증 Token |
| TELEGRAM_CHAT_ID | 메시지를 받을 사용자 또는 그룹 ID |

---

# Telegram Bot 생성

Telegram에서 BotFather를 검색한다.


@BotFather


BotFather 접속 후 Bot 생성 명령 실행


/newbot


생성 완료 후 Bot Token을 발급받는다.

예:


123456789:AAxxxxxxxxxxxxxxxxxxxx


해당 값이:


TELEGRAM_BOT_TOKEN


이다.

---

# Bot 실행 확인

생성한 Bot을 검색한다.

예:


@mkdocs_notification_bot


Bot 접속 후 아래 명령을 실행한다.


/start


주의:

`/start`를 입력해도 Telegram 화면에서 별도 응답 메시지가 표시되지 않을 수 있다.

Chat ID 확인은 Telegram Bot API의 `getUpdates`를 통해 확인한다.

---

# Chat ID 확인 방법

브라우저에서 아래 주소를 호출한다.


https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/getUpdates


예:


https://api.telegram.org/bot123456789:AAxxxx/getUpdates


정상 결과:

```json
{
  "ok": true,
  "result": [
    {
      "message": {
        "chat": {
          "id": 987654321,
          "first_name": "Shin",
          "type": "private"
        },
        "text": "/start"
      }
    }
  ]
}

결과에서 아래 값을 확인한다.

"id": 987654321

해당 값이:

TELEGRAM_CHAT_ID

이다.

GitHub Secrets 등록

Repository 이동

Settings
 └── Secrets and variables
      └── Actions

Secrets 등록

Name	Value
TELEGRAM_BOT_TOKEN	BotFather에서 발급받은 Token
TELEGRAM_CHAT_ID	getUpdates에서 확인한 Chat ID
설정 완료 흐름
Telegram Bot 생성

        ↓

BOT_TOKEN 발급

        ↓

사용자가 Bot에게 /start 입력

        ↓

getUpdates 확인

        ↓

CHAT_ID 확인

        ↓

GitHub Secrets 등록

        ↓

GitHub Actions에서 Telegram 메시지 전송
주의 사항
Bot ID와 Chat ID 혼동 주의

잘못된 설정:

TELEGRAM_CHAT_ID = Bot ID

Bot은 자기 자신에게 메시지를 보낼 수 없다.

정상 설정:

TELEGRAM_CHAT_ID = 사용자 또는 그룹 Chat ID
오류 예시

발생 오류:

{
  "ok": false,
  "error_code": 403,
  "description": "Forbidden: the bot can't send messages to the bot"
}

원인:

Bot ID를 TELEGRAM_CHAT_ID로 등록한 경우 발생한다.

해결:

Telegram에서 Bot에게 /start 입력
getUpdates 실행
chat.id 값 확인
GitHub Secrets의 TELEGRAM_CHAT_ID 수정