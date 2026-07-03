# 배포 런북 (Ubuntu, OpenClaw 기반)

이 문서는 `PLAN.md` Phase 0~2를 실제 Ubuntu 서버에서 실행하는 순서다.
값을 채워야 하는 부분(Phase 0에서 확보할 것들)은 굵게 표시.

## 0. 사전 준비물 (사용자가 직접 확보)

- Ubuntu 서버 SSH 접속 정보
- Telegram Bot 4개 토큰 (BotFather: `/newbot` 4번 실행)
- 본인 Telegram 사용자 ID (예: `@userinfobot`에게 물어보면 알려줌)
- OpenRouter API 키 (https://openrouter.ai)
- OpenAI API 키 (https://platform.openai.com)
- Ollama 서버 주소 (이미 어딘가 구동 중이어야 함 — 이 프로젝트는 설치하지 않음)
- Claude Max/Pro 구독 계정 (Claude Code CLI 로그인용)

## 1. OpenClaw 설치

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
openclaw --version
```

## 2. Claude Max 로그인 (Claude CLI backend용)

```bash
claude login
claude --version   # 인증 확인
```

## 3. 설정 파일 배치

이 저장소(`0601`)의 `openclaw.json`을 `~/.openclaw/openclaw.json`에 복사하고,
`.env.example`을 참고해 `~/.openclaw/.env`에 실제 값을 채운다.

```bash
mkdir -p ~/.openclaw
cp openclaw.json ~/.openclaw/openclaw.json
cp .env.example ~/.openclaw/.env
# ~/.openclaw/.env 를 열어 토큰/키 값 채우기
chmod 600 ~/.openclaw/.env
```

## 4. 설정 검증

```bash
openclaw doctor
```

경고/오류가 없을 때까지 `openclaw.json`/`.env`를 수정한다.
(SecretRef `${VAR}` 형태가 정상 resolve 되는지도 이 단계에서 확인됨 —
 만약 실패하면 `secrets.providers.default: { source: "env" }` 를 `openclaw.json`에
 명시적으로 추가해야 할 수 있음)

## 5. 4-Agent / 4-Bot 라우팅 확인

```bash
openclaw agents list --bindings
openclaw channels status --probe
```

각 봇이 정상 응답하는지 Telegram 앱에서 4개 채팅방 각각 `/start` 후 메시지를 보내 확인한다.

## 6. 24시간 상시 구동 등록 (systemd)

```bash
openclaw gateway install   # systemd user service 등록
openclaw gateway status
```

## 7. 운영 점검 명령 모음

```bash
openclaw doctor
openclaw gateway status
openclaw agents list --bindings
openclaw channels status --probe
```

## 트러블슈팅

- `getMe returned 401` → 해당 봇 토큰이 잘못됨. `.env`의 `TELEGRAM_BOT_TOKEN_*` 값 재확인
- 특정 agent가 응답 없음 → `openclaw agents list --bindings`로 binding이 올바른 accountId에 연결됐는지 확인
- Claude 응답 실패 → 서버에서 `claude --version`으로 로그인 세션이 살아있는지 확인, 필요 시 `claude login` 재실행
