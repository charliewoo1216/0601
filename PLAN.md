# 개인 AI Agent 플랫폼 구현 계획 (V1.0)

> 본 문서는 `요구사항 V1.0`을 실제로 구축하기 위한 단계별 실행 계획이다.
> 목표: Ubuntu 서버에서 24시간 상시 동작하는, Telegram 기반의 단일 AI Gateway +
> 멀티 LLM Router + Tool Executor 구조의 개인 AI Agent 플랫폼.

---

## 0. 설계 요약

```
Telegram Bots (Coding / Analysis / Web / Server)
        │  (webhook or long-polling)
        ▼
   AI Gateway (FastAPI)  ── 인증 / 세션 / 라우팅 진입점
        │
   LLM Router            ── 요청 분류 후 최적 모델 선택
        │
   ┌────┴─────────────────────────────────────┐
   │  Fast │ Analysis │ Coding │ Reasoning │ Search
   └────┬─────────────────────────────────────┘
        │
   Tool Executor          ── 샌드박스 내 도구 실행 (Bash/Git/Docker/...)
        │
   결과 스트리밍 → Telegram
```

핵심 원칙:
- Telegram = UI만 담당 (비즈니스 로직 없음)
- Gateway = 모든 요청의 단일 진입점, 인증/로깅/큐잉 담당
- Router = 플러그인 구조 (모델 추가 시 코드 최소 변경)
- 모든 LLM(로컬/원격 포함)은 **API로만 연결** — 이 프로젝트가 LLM 서버를 직접 설치/구동하지 않음
- 모든 설정값(엔드포인트, 키, 모델명, 봇 토큰 등)은 **`.env` 하나로 관리**
- Claude Max Wrapper = 코딩/추론 기본 엔진 (claude CLI 래핑)
- Tool Executor = 모든 도구 실행을 격리된 컨텍스트에서 수행 (권한 제어 필수)
- 배포 대상 = Ubuntu 서버 단일 환경 (24시간 상시 구동)

---

## 1. 기술 스택 (제안)

| 영역 | 선택 | 사유 |
|---|---|---|
| Gateway/Backend | Python 3.11 + FastAPI | 비동기, LLM/Telegram SDK 생태계 풍부 |
| Telegram | python-telegram-bot (v21+, async) | webhook/polling 모두 지원 |
| 작업 큐 | Redis + RQ 또는 Celery | Claude Wrapper 작업 큐, 세션 관리 |
| LLM 연결 | HTTP API 클라이언트 (OpenAI 호환 스펙 우선) | Fast/Analysis/Reasoning/Search 모두 **API 엔드포인트로만 연결**. 로컬이든 원격이든 이 프로젝트는 LLM 서버를 설치·구동하지 않고 `.env`에 지정된 `BASE_URL`/`API_KEY`/`MODEL`만 사용 |
| 상태/세션 저장 | SQLite(초기) → PostgreSQL(확장) | 세션, 로그, 사용량 기록 |
| 컨테이너화 | Docker Compose | Gateway/Redis/DB 컨테이너 (LLM 서버는 포함하지 않음) |
| 프로세스 관리 | systemd (또는 docker compose + restart:always) | Ubuntu 서버에서 24시간 상시 구동 |
| MCP 연동 | mcp Python SDK | Tool Executor 확장 포인트 |
| Claude 연동 | Claude Code CLI (Max 구독) subprocess wrapper | `.env`에 인증 정보 지정, CLI를 큐로 감싸서 사용 |

> 참고: Claude Max는 공개 API 요금제가 아니라 `claude` CLI 로그인 세션을 사용하는 구독제이다.
> 인증 방식은 두 가지를 모두 지원하도록 설계한다.
> 1) `claude login`으로 생성된 자격 증명 디렉터리를 `.env`의 `CLAUDE_CONFIG_DIR` 경로로 지정 (기본 권장)
> 2) 별도 API 키/토큰 기반 인증을 쓰는 경우 `CLAUDE_MAX_AUTH_TOKEN` 등으로 `.env`에 저장 후 wrapper가 주입
> 어느 방식이든 **키/토큰 원문은 절대 코드나 config yaml에 하드코딩하지 않고 `.env`에서만 로드**한다.

---

## 2. 환경 변수(`.env`) 설계

모든 실행 옵션(엔드포인트, 키, 모델명, 봇 토큰, 권한 스위치)은 `.env` 하나로 제어한다.
`config/*.yaml`에는 **비밀값을 넣지 않고**, 라우팅 우선순위·타임아웃 같은 비민감 설정만 둔다.

```dotenv
# ── Gateway ─────────────────────────────
GATEWAY_HOST=0.0.0.0
GATEWAY_PORT=8000
GATEWAY_LOG_LEVEL=info
GATEWAY_SECRET_KEY=changeme

# ── Telegram ────────────────────────────
TELEGRAM_BOT_TOKEN_CODING=xxxx
TELEGRAM_BOT_TOKEN_ANALYSIS=xxxx
TELEGRAM_BOT_TOKEN_WEB=xxxx
TELEGRAM_BOT_TOKEN_SERVER=xxxx
TELEGRAM_ALLOWED_USER_IDS=111111,222222

# ── Claude Max Wrapper ──────────────────
CLAUDE_CLI_PATH=/usr/local/bin/claude
CLAUDE_CONFIG_DIR=/root/.claude          # claude login 세션 디렉터리
CLAUDE_MAX_AUTH_TOKEN=                    # 필요 시에만 사용 (선택)
CLAUDE_WORKDIR=/opt/agent-platform/workspace
CLAUDE_DEFAULT_MODEL=claude-sonnet-5
CLAUDE_TIMEOUT_SEC=300

# ── Fast LLM (API) ──────────────────────
FAST_LLM_BASE_URL=http://<host>:<port>/v1   # 로컬/원격 API 서버 엔드포인트
FAST_LLM_API_KEY=xxxx
FAST_LLM_MODEL=gemma-2-9b-it

# ── Analysis LLM (API) ──────────────────
ANALYSIS_LLM_BASE_URL=http://<host>:<port>/v1
ANALYSIS_LLM_API_KEY=xxxx
ANALYSIS_LLM_MODEL=qwen2.5-72b-instruct

# ── Reasoning / Search (OpenRouter 등) ──
OPENROUTER_API_KEY=xxxx
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
REASONING_MODEL=anthropic/claude-opus
SEARCH_MODEL=perplexity/sonar

# ── 검색 API ─────────────────────────────
SEARCH_PROVIDER=brave                     # brave | serpapi | tavily 등
SEARCH_API_KEY=xxxx

# ── 인프라 ───────────────────────────────
REDIS_URL=redis://localhost:6379/0
DATABASE_URL=sqlite:///./data/app.db      # 추후 postgresql://... 로 교체

# ── Tool Executor 권한 ──────────────────
TOOL_BASH_ENABLED=true
TOOL_SSH_ENABLED=false
TOOL_DOCKER_ENABLED=true
TOOL_DB_ENABLED=false
SSH_ALLOWED_HOSTS=
```

원칙:
- `.env.example`에는 키 이름만 두고 값은 비워둔다 (git에는 example만 커밋, 실제 `.env`는 `.gitignore` 처리)
- 각 LLM Provider는 `BASE_URL / API_KEY / MODEL` 3종 세트만 있으면 즉시 교체 가능한 구조 (OpenAI 호환 API 가정)
- 새 LLM을 추가할 때도 코드 수정 없이 `.env`에 `{ROLE}_LLM_BASE_URL/API_KEY/MODEL` 세트만 추가 + `config/models.yaml`에 후보 등록

---

## 3. 리포지토리 구조 (제안)

```
0601/
├── PLAN.md
├── docker-compose.yml          # Gateway + Redis (+ 필요 시 Postgres) 만 포함, LLM 서버 없음
├── .env.example
├── gateway/                  # AI Gateway (FastAPI)
│   ├── main.py
│   ├── settings.py             # .env 로드 (pydantic-settings)
│   ├── auth.py                # Telegram 사용자 화이트리스트 인증
│   ├── router/                # LLM Router
│   │   ├── classifier.py      # 요청 분류 (fast/analysis/coding/reasoning/search)
│   │   ├── registry.py        # 모델 플러그인 레지스트리
│   │   └── models/
│   │       ├── base.py        # LLMProvider 인터페이스 (OpenAI 호환 API 클라이언트)
│   │       ├── openai_compatible_provider.py   # Fast/Analysis 등 범용 API Provider
│   │       ├── claude_wrapper_provider.py
│   │       ├── openrouter_provider.py
│   │       └── ...
│   ├── tools/                  # Tool Executor
│   │   ├── executor.py         # 실행 디스패처 + 권한 체크
│   │   ├── bash_tool.py
│   │   ├── python_tool.py
│   │   ├── git_tool.py
│   │   ├── docker_tool.py
│   │   ├── ssh_tool.py
│   │   ├── browser_tool.py     # Playwright
│   │   ├── file_editor_tool.py
│   │   ├── db_tool.py          # MySQL/PG/Oracle/Tibero
│   │   └── mcp_tool.py
│   ├── claude_wrapper/
│   │   ├── wrapper.py          # claude CLI subprocess 래퍼
│   │   ├── session_manager.py
│   │   └── queue.py
│   └── logging_store.py
├── bots/                       # Telegram Bot 4종 (얇은 레이어, Gateway 호출만)
│   ├── coding_bot.py
│   ├── analysis_bot.py
│   ├── web_bot.py
│   └── server_bot.py
├── config/
│   ├── models.yaml             # LLM 역할별 후보 목록/우선순위/fallback (비밀값 없음, env 키 이름만 참조)
│   └── permissions.yaml        # 도구별 허용 범위, 화이트리스트
├── scripts/
│   ├── install.sh               # Ubuntu 서버용 설치 스크립트
│   └── systemd/*.service
└── tests/
```

`config/models.yaml` 예시 (비밀값 없이 env 키 이름만 참조):
```yaml
fast:
  - provider: openai_compatible
    base_url_env: FAST_LLM_BASE_URL
    api_key_env: FAST_LLM_API_KEY
    model_env: FAST_LLM_MODEL
analysis:
  - provider: openai_compatible
    base_url_env: ANALYSIS_LLM_BASE_URL
    api_key_env: ANALYSIS_LLM_API_KEY
    model_env: ANALYSIS_LLM_MODEL
coding:
  - provider: claude_wrapper
reasoning:
  - provider: claude_wrapper
  - provider: openrouter
    model_env: REASONING_MODEL
search:
  - provider: openrouter
    model_env: SEARCH_MODEL
```

---

## 4. 단계별 실행 계획

### Phase 0 — 기반 준비 (0.5주)
- [ ] Ubuntu 서버 환경 점검 (Docker, Docker Compose, Python 3.11) — **LLM 서버 설치는 불필요**, 접속할 API 엔드포인트만 확보
- [ ] Telegram Bot 4개 생성 (BotFather), 토큰 발급
- [ ] 리포지토리 스캐폴딩 (위 구조), `.env.example` 작성 (섹션 2 항목 전부 포함)
- [ ] `claude login` 수행하여 Claude Max 인증 세션 확보, `.env`의 `CLAUDE_CONFIG_DIR`에 경로 지정
- [ ] 사용자 화이트리스트 기반 인증 설계 (Telegram user_id 허용 목록)

### Phase 1 — AI Gateway 최소 골격 (1주)
- [ ] `pydantic-settings` 기반 `.env` 로더(`settings.py`) 구현 — 모든 하드코딩 제거
- [ ] FastAPI Gateway 기본 서버 (`/health`, `/chat` 엔드포인트)
- [ ] Telegram Bot → Gateway 호출 연결 (Bot 1개로 우선 검증, 예: Coding Bot)
- [ ] 요청/응답 로깅 (구조화 로그, 세션 ID 부여)
- [ ] 기본 인증 미들웨어 (허용된 Telegram user_id만 통과)

### Phase 2 — Claude Max Wrapper (1주)
- [ ] `claude` CLI subprocess 래퍼 구현 (`.env`의 `CLAUDE_CLI_PATH`/`CLAUDE_CONFIG_DIR`/`CLAUDE_WORKDIR` 사용, 타임아웃, JSON 출력 파싱)
- [ ] 작업 큐 도입 (Redis + RQ) — 동시 요청 직렬화/세션별 격리
- [ ] 세션 관리 (사용자별 대화 컨텍스트 유지/초기화 명령)
- [ ] 스트리밍 응답 → Telegram 메시지 스트리밍(edit_message) 반영
- [ ] 로그 저장 (요청/응답/실행시간/에러)

### Phase 3 — LLM Router + API 기반 LLM 연결 (1~1.5주)
- [ ] `LLMProvider` 공통 인터페이스 정의 (동기/스트리밍 `generate()`, OpenAI 호환 API 기준)
- [ ] `openai_compatible_provider.py` 구현 — `.env`의 `BASE_URL/API_KEY/MODEL`만으로 Fast/Analysis 어떤 API 서버든 연결 (로컬 서버든 클라우드든 무관)
- [ ] 요청 분류기(classifier) 구현: 규칙 기반 우선 (키워드/길이/봇 종류) → 추후 소형 분류 모델로 고도화
- [ ] `models.yaml` 기반 라우팅 설정 (역할별 후보 목록 + fallback 순서, 실제 값은 env에서 주입)
- [ ] Fast/Analysis 경로를 지정된 API로 연결, Coding/Reasoning은 Claude Wrapper로 연결
- [ ] API 연결 실패 시 fallback 정책 (예: Fast → Claude Wrapper로 강등)

### Phase 4 — Tool Executor (1.5~2주)
- [ ] 권한 제어 설계 (`permissions.yaml` + `.env`의 `TOOL_*_ENABLED` 스위치: 도구별 allow/deny, 명령 화이트리스트, 위험 명령 확인 절차)
- [ ] Bash/Python/Git/File Editor 도구 구현 (샌드박스 작업 디렉터리 강제)
- [ ] Docker/Docker Compose/K8s 도구 구현 (읽기 전용 우선, 쓰기 작업은 확인 절차)
- [ ] SSH 도구 구현 (`SSH_ALLOWED_HOSTS` 화이트리스트, 키 기반 인증만 허용)
- [ ] Playwright/Browser 도구 구현 (Web Bot 연결)
- [ ] DB 도구 구현 (MySQL/PostgreSQL 우선, Oracle/Tibero는 커넥터 확인 후 추가, 접속정보는 전부 `.env`)
- [ ] MCP 클라이언트 통합 (외부 MCP 서버 plug-in 방식 연결)

### Phase 5 — 나머지 Bot 3종 연결 (1주)
- [ ] Analysis Bot (로그/PDF/CSV/RAG 파이프라인 — 임베딩 저장소 선정: 초기엔 SQLite+FTS 또는 Chroma)
- [ ] Web Bot (검색 API + Playwright 크롤링 결과 요약 파이프라인, `SEARCH_PROVIDER`/`SEARCH_API_KEY` 사용)
- [ ] Server Bot (서버 상태 조회/재시작 — 고위험 명령은 반드시 사용자 확인 단계 삽입)

### Phase 6 — 확장 Provider 추가 (병행 가능)
- [ ] OpenRouter Provider (Reasoning/Search 대체 후보, `.env`의 `OPENROUTER_API_KEY`)
- [ ] 검색 API Provider (Brave/SerpAPI/Tavily 등, `.env`의 `SEARCH_PROVIDER`로 스위치)
- [ ] Provider 플러그인 등록 방식 문서화 (신규 LLM 추가 시 `.env` + `models.yaml` 체크리스트)

### Phase 7 — 운영/안정화 (Ubuntu 배포) (1주)
- [ ] Ubuntu 서버에 `scripts/install.sh`로 의존성 설치 (Python venv, Redis, systemd 유닛 등록)
- [ ] systemd 서비스 등록 (또는 `docker compose --restart always`)로 24시간 구동
- [ ] 헬스체크 + 자동 재시작
- [ ] 비용/사용량 로그 대시보드 (간단한 CLI 리포트 또는 로그 집계)
- [ ] 에러 알림 (Telegram 관리자 채널로 장애 알림)
- [ ] 백업 (세션/설정/로그 주기적 백업, `.env`는 별도 안전한 위치에 백업)

---

## 5. 주요 리스크 & 결정 필요 사항

| 항목 | 리스크 | 결정 필요 |
|---|---|---|
| Claude Max Wrapper 인증 | CLI 로그인 세션 방식이라 컨테이너 재시작/서버 이전 시 재로그인 필요 가능 | `CLAUDE_CONFIG_DIR`을 영속 볼륨/디렉터리로 고정, 세션 만료 감지 및 알림 |
| Claude Max Wrapper 동시성 | CLI 기반이라 동시 세션/Rate limit 이슈 가능 | 큐 직렬화 정책, 동시 실행 수 제한 |
| 외부 LLM API 엔드포인트 | Fast/Analysis용 API 서버(로컬망 또는 클라우드)의 가용성·지연시간 미확정 | 실제 사용할 API 엔드포인트/제공자 확정 필요 (사용자가 이미 운영 중인 서버가 있는지 확인) |
| 도구 권한 (Bash/SSH/DB) | 잘못된 명령으로 서버 손상 위험 | 위험 명령 목록 정의 + 실행 전 사용자 확인(Telegram 버튼) 도입 여부 |
| RAG/문서 저장소 | 벡터 DB 선택 미정 | Chroma(로컬, 간단) vs pgvector(확장성) 결정 |
| 인증 | Telegram만으로 충분한가 | 다중 관리자/사용자 시 역할별 권한 분리 필요 여부 |
| 비밀값 관리 | `.env` 파일 유출 시 전체 키 노출 | 파일 권한(600), git 제외, 서버 반입 시 안전한 전달 방식 확정 |

---

## 6. 다음 액션

1. Fast/Analysis LLM이 실제로 어떤 API를 가리킬지 확정 (자체 운영 중인 API 서버 주소가 있는지, 아니면 어떤 클라우드 제공자를 쓸지)
2. `.env.example` 초안에 동의하는지 확인 후 Phase 0~1 스캐폴딩 시작
3. Ubuntu 서버 접근 정보 확보 (SSH 접속 가능 여부) → `scripts/install.sh` 작성 및 실제 배포 테스트
