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
- Claude Max Wrapper = 코딩/추론 기본 엔진 (claude CLI 래핑)
- Tool Executor = 모든 도구 실행을 격리된 컨텍스트에서 수행 (권한 제어 필수)

---

## 1. 기술 스택 (제안)

| 영역 | 선택 | 사유 |
|---|---|---|
| Gateway/Backend | Python 3.11 + FastAPI | 비동기, LLM/Telegram SDK 생태계 풍부 |
| Telegram | python-telegram-bot (v21+, async) | webhook/polling 모두 지원 |
| 작업 큐 | Redis + RQ 또는 Celery | Claude Wrapper 작업 큐, 세션 관리 |
| 로컬 LLM 서빙 | Ollama (Gemma, Qwen 계열) | 설치/모델 스왑 간편, OpenAI 호환 API |
| 상태/세션 저장 | SQLite(초기) → PostgreSQL(확장) | 세션, 로그, 사용량 기록 |
| 컨테이너화 | Docker Compose | Gateway/Redis/Ollama/DB 각각 컨테이너 |
| 프로세스 관리 | systemd (또는 docker compose + restart:always) | 24시간 상시 구동 |
| MCP 연동 | mcp Python SDK | Tool Executor 확장 포인트 |
| Claude 연동 | Claude Code CLI (Max 구독) subprocess wrapper | 별도 API 키 불필요, Max 플랜 그대로 사용 |

> 참고: Claude Max는 공개 API가 아니라 `claude` CLI를 subprocess로 감싸는 방식이므로,
> 동시성 제어(단일 세션 direnv/workdir), 타임아웃, JSON 출력 파싱이 Wrapper의 핵심 과제.

---

## 2. 리포지토리 구조 (제안)

```
0601/
├── PLAN.md
├── docker-compose.yml
├── .env.example
├── gateway/                  # AI Gateway (FastAPI)
│   ├── main.py
│   ├── auth.py                # Telegram 사용자 화이트리스트 인증
│   ├── router/                # LLM Router
│   │   ├── classifier.py      # 요청 분류 (fast/analysis/coding/reasoning/search)
│   │   ├── registry.py        # 모델 플러그인 레지스트리
│   │   └── models/
│   │       ├── base.py        # LLMProvider 인터페이스
│   │       ├── ollama_provider.py
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
│   ├── models.yaml             # LLM 후보/우선순위/비용 설정
│   └── permissions.yaml        # 도구별 허용 범위, 화이트리스트
├── scripts/
│   ├── install.sh
│   └── systemd/*.service
└── tests/
```

---

## 3. 단계별 실행 계획

### Phase 0 — 기반 준비 (0.5주)
- [ ] Ubuntu 서버 환경 점검 (Docker, Docker Compose, Python 3.11, Ollama 설치)
- [ ] Telegram Bot 4개 생성 (BotFather), 토큰 발급
- [ ] 리포지토리 스캐폴딩 (위 구조), `.env.example` 작성
- [ ] 사용자 화이트리스트 기반 인증 설계 (Telegram user_id 허용 목록)

### Phase 1 — AI Gateway 최소 골격 (1주)
- [ ] FastAPI Gateway 기본 서버 (`/health`, `/chat` 엔드포인트)
- [ ] Telegram Bot → Gateway 호출 연결 (Bot 1개로 우선 검증, 예: Coding Bot)
- [ ] 요청/응답 로깅 (구조화 로그, 세션 ID 부여)
- [ ] 기본 인증 미들웨어 (허용된 Telegram user_id만 통과)

### Phase 2 — Claude Max Wrapper (1주)
- [ ] `claude` CLI subprocess 래퍼 구현 (작업 디렉터리 지정, 타임아웃, JSON 출력 파싱)
- [ ] 작업 큐 도입 (Redis + RQ) — 동시 요청 직렬화/세션별 격리
- [ ] 세션 관리 (사용자별 대화 컨텍스트 유지/초기화 명령)
- [ ] 스트리밍 응답 → Telegram 메시지 스트리밍(edit_message) 반영
- [ ] 로그 저장 (요청/응답/실행시간/에러)

### Phase 3 — LLM Router + 로컬 LLM (1~1.5주)
- [ ] Ollama 설치 및 Fast/Analysis 후보 모델 pull (Gemma, Qwen 등)
- [ ] `LLMProvider` 공통 인터페이스 정의 (동기/스트리밍 `generate()`)
- [ ] 요청 분류기(classifier) 구현: 규칙 기반 우선 (키워드/길이/봇 종류) → 추후 소형 분류 모델로 고도화
- [ ] `models.yaml` 기반 라우팅 설정 (역할별 후보 목록 + fallback 순서)
- [ ] Fast/Analysis 경로를 로컬 LLM으로 연결, Coding/Reasoning은 Claude Wrapper로 연결

### Phase 4 — Tool Executor (1.5~2주)
- [ ] 권한 제어 설계 (`permissions.yaml`: 도구별 allow/deny, 명령 화이트리스트, 위험 명령 확인 절차)
- [ ] Bash/Python/Git/File Editor 도구 구현 (샌드박스 작업 디렉터리 강제)
- [ ] Docker/Docker Compose/K8s 도구 구현 (읽기 전용 우선, 쓰기 작업은 확인 절차)
- [ ] SSH 도구 구현 (대상 서버 화이트리스트, 키 기반 인증만 허용)
- [ ] Playwright/Browser 도구 구현 (Web Bot 연결)
- [ ] DB 도구 구현 (MySQL/PostgreSQL 우선, Oracle/Tibero는 커넥터 확인 후 추가)
- [ ] MCP 클라이언트 통합 (외부 MCP 서버 plug-in 방식 연결)

### Phase 5 — 나머지 Bot 3종 연결 (1주)
- [ ] Analysis Bot (로그/PDF/CSV/RAG 파이프라인 — 임베딩 저장소 선정: 초기엔 SQLite+FTS 또는 Chroma)
- [ ] Web Bot (검색 API + Playwright 크롤링 결과 요약 파이프라인)
- [ ] Server Bot (서버 상태 조회/재시작 — 고위험 명령은 반드시 사용자 확인 단계 삽입)

### Phase 6 — 확장 Provider 추가 (병행 가능)
- [ ] OpenRouter Provider (Reasoning/Search 대체 후보)
- [ ] 검색 API Provider (예: Brave Search / SerpAPI 등, 사용자 선택 필요)
- [ ] Provider 플러그인 등록 방식 문서화 (신규 LLM 추가 시 체크리스트)

### Phase 7 — 운영/안정화 (1주)
- [ ] systemd 서비스 등록 또는 `docker compose --restart always` 로 24시간 구동
- [ ] 헬스체크 + 자동 재시작
- [ ] 비용/사용량 로그 대시보드 (간단한 CLI 리포트 또는 로그 집계)
- [ ] 에러 알림 (Telegram 관리자 채널로 장애 알림)
- [ ] 백업 (세션/설정/로그 주기적 백업)

---

## 4. 주요 리스크 & 결정 필요 사항

| 항목 | 리스크 | 결정 필요 |
|---|---|---|
| Claude Max Wrapper | CLI 기반이라 동시 세션/Rate limit 이슈 가능 | 큐 직렬화 정책, 동시 실행 수 제한 |
| 도구 권한 (Bash/SSH/DB) | 잘못된 명령으로 서버 손상 위험 | 위험 명령 목록 정의 + 실행 전 사용자 확인(Telegram 버튼) 도입 여부 |
| 로컬 LLM 성능 | 저사양 서버에서 80B 모델 구동 불가 가능 | 서버 GPU/RAM 스펙 확인 후 모델 크기 조정 |
| RAG/문서 저장소 | 벡터 DB 선택 미정 | Chroma(로컬, 간단) vs pgvector(확장성) 결정 |
| 인증 | Telegram만으로 충분한가 | 다중 관리자/사용자 시 역할별 권한 분리 필요 여부 |

---

## 5. 다음 액션

1. 위 리포지토리 구조에 동의하는지 확인
2. Phase 0~1 (Gateway 골격 + Bot 1개 연결)부터 실제 코드 스캐폴딩 시작
3. Ubuntu 서버 스펙(CPU/GPU/RAM) 확인 → 로컬 LLM 모델 크기 확정
