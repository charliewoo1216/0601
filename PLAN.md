# 개인 AI Agent 플랫폼 구현 계획 (V2.0 — OpenClaw 기반)

> V1.0에서는 Gateway/LLM Router/Tool Executor를 처음부터 설계했으나, 실제 요구사항 대부분을
> 이미 구현하고 있는 오픈소스 프로젝트 **OpenClaw**(https://github.com/openclaw/openclaw,
> fork: https://github.com/charliewoo1216/openclaw)를 확인한 뒤 방향을 전환했다.
> **처음부터 만들지 않고, OpenClaw를 채택 → 설정(config) → 필요한 부분만 소스코드/확장(extension)으로
> 수정·추가**하는 방식으로 진행한다.

---

## 0. 왜 OpenClaw인가 (근거)

저장소(`/workspace/openclaw`, `package.json` 설명: *"Multi-channel AI gateway with extensible
messaging integrations"*)를 실제로 열어 확인한 결과:

| 우리가 설계하려던 것 | OpenClaw에 이미 있는 것 | 근거 파일 |
|---|---|---|
| Telegram Gateway, 다중 Bot | `channels.telegram.accounts.<id>` — 채널당 여러 봇 계정 지원 | `docs/concepts/multi-agent.md` |
| 역할별 Bot(코딩/분석/웹/서버) | **Multi-agent routing**: `agents.list[]` 각각 독립 workspace/model/tool정책/sandbox, `bindings`로 채널 계정 ↔ agent 매핑 | `docs/concepts/multi-agent.md` |
| Claude Max Wrapper (CLI 래핑, 세션, JSON, 스트리밍) | `anthropic` 확장의 **CLI backend**가 `claude -p --output-format stream-json --session-id --resume` 를 그대로 구현, MCP 번들링까지 포함 | `extensions/anthropic/cli-backend.ts` |
| OpenRouter / OpenAI / Ollama Provider | 전부 built-in 확장으로 존재 (`extensions/openrouter`, `extensions/ollama`, OpenAI는 core) | `extensions/openrouter`, `extensions/ollama` |
| Tool Executor: Bash/Python | `exec` 툴 (`group:runtime`), 백그라운드 실행/타임아웃/승격 실행 지원 | `docs/gateway/background-process.md`, `docs/gateway/config-tools.md` |
| Tool Executor: File Editor | `read/write/edit/apply_patch` (`group:fs`) | `docs/gateway/config-tools.md` |
| Tool Executor: Browser | `browser` 툴 (Playwright 기반, CDP, sandbox 격리) | `docs/gateway/sandboxing.md` |
| Tool Executor: Docker 격리 | `sandbox.mode/scope/backend` — agent/session/shared 단위 Docker 컨테이너 격리 | `docs/gateway/sandboxing.md` |
| Tool Executor: SSH | **sandbox backend로 `"ssh"` 지원** — 원격 호스트에 SSH로 실행 (Server bot에 적합) | `docs/gateway/sandboxing.md` (`sandbox.ssh.target`, `identityFile`) |
| Tool Executor: MCP | `mcp.servers` 설정으로 외부 MCP 서버 등록, 에이전트별 tool 허용목록으로 노출 제어 | `docs/gateway/config-tools.md` |
| 권한 제어 (도구별 allow/deny, 위험 명령 확인) | `tools.profile`(`minimal/coding/messaging/full`), `tools.allow/deny`, `agents.list[].tools`, `tools.elevated` | `docs/gateway/config-tools.md`, `docs/tools/elevated.md` |
| 배포(Ubuntu, 24시간) | 공식 설치 스크립트(`curl -fsSL https://openclaw.ai/install.sh \| bash`), Docker/Docker Compose, systemd 연동(child-process bridge) | `docs/install/index.md`, `docs/gateway/background-process.md` |

**없는 것 (직접 채워야 하는 부분)**:
- DB 조회 tool → 기본 제공 안 됨. **DB는 SQLite만 지원**하기로 확정(MySQL/PostgreSQL/Oracle/Tibero 지원 안 함) —
  `exec` 툴로 `sqlite3` CLI를 바로 실행하면 되므로 별도 구현이 사실상 불필요. 더 안전한 구조화된 접근이
  필요해지면 SQLite 전용 MCP 서버(예: 공식/커뮤니티 `mcp-server-sqlite`)를 `mcp.servers`에 등록
- "간단한 질문은 Fast 모델, 복잡하면 Reasoning 모델로 자동 승격"하는 **task-복잡도 기반 동적 라우팅**은 없음. OpenClaw의 모델 선택은 기본적으로 **agent당 고정 primary + 장애 시 fallback**(신뢰성 목적, 비용/속도 목적 아님) — `docs/concepts/model-failover.md`. 이 부분이 필요하면 별도 커스텀 확장(후술 Phase 5)으로 채운다.

---

## 1. 핵심 개념 매핑 (요구사항 → OpenClaw)

```
요구사항                          →  OpenClaw 개념
──────────────────────────────────────────────────────
Telegram Bot 4종(코딩/분석/웹/서버)  →  agents.list[] 4개 + 각기 다른 telegram accountId
                                       + bindings로 봇↔agent 연결
LLM Router (Fast/Analysis/…)      →  agent별 고정 model 배정 (모델별 provider는 임의 지정,
                                       추후 agents.list[].model 값만 바꾸면 교체)
Claude Max Wrapper                →  extensions/anthropic 의 claude CLI backend (이미 구현됨)
Tool Executor                      →  exec/read/write/edit/browser + sandbox(mode/scope/backend)
                                       + mcp.servers (SSH는 sandbox backend, DB는 SQLite만: exec+sqlite3)
확장성 (신규 LLM/도구 추가)          →  extensions/ 플러그인 SDK (`openclaw/plugin-sdk`)
```

---

## 2. 4개 Bot ↔ Agent 설계 (초안, 값은 배치 전 확정)

| Bot(요구사항) | agentId | 기본 tools 프로파일 | sandbox | 1차 모델(provider) — **임의 배정, 나중에 교체 가능** |
|---|---|---|---|---|
| Coding | `coding` | `coding` (`group:fs`,`group:runtime`,`group:web`,`group:sessions`,`cron` 등) | `mode: "non-main"`, `backend: "docker"` | `anthropic` (Claude CLI backend = Claude Max) |
| Analysis | `analysis` | `group:fs`(read 위주) + `group:memory` + `exec`(sqlite3 CLI로 SQLite 조회) | `mode: "all"`, `backend: "docker"` | `openai` 또는 `ollama` |
| Web | `web` | `group:web` + `browser`(Playwright) | `mode: "all"`, `backend: "docker"` (sandbox browser) | `openrouter` |
| Server | `server` | `exec`(elevated 일부 허용) + `group:nodes` | `backend: "ssh"` (대상 Ubuntu 서버로 원격 실행) | `anthropic` 또는 `ollama` |

각 agent는 `bindings`로 Telegram의 서로 다른 `accountId`(= BotFather로 만든 개별 봇 토큰)에 연결한다
(`docs/concepts/multi-agent.md`의 "Telegram bots per agent" 예시 그대로 사용 가능).

### 사용자 경험(UX): 휴대폰 1대, 채팅방 4개 (확정)

봇/휴대폰/계정을 늘리는 게 아니라, **본인 Telegram 앱 안에 채팅방 4개**가 생기는 방식으로 확정.

- 봇 4개(코딩/분석/웹/서버)는 각각 고유 사용자명을 가지며, 본인 계정에서 각 봇을 검색해 `/start`만 하면
  채팅방으로 추가됨 (계정·기기 추가 불필요).
- 어떤 역할이 필요한지는 **어느 채팅방을 여는지**로 선택 (예: 서버 점검 → `@my_server_bot` 채팅방).
- 4개 채팅방은 Telegram의 **채팅 폴더(Chat Folders)** 로 묶어 탭 한 번에 전환, 필요하면 각각 **고정(pin)**.
- 대안(봇 1개 + 그룹 Topics/peer 라우팅으로 채팅방 1개에 합치는 방식)은 라우팅이 복잡해져 채택하지 않음.

---

## 3. 환경 변수(`.env`) 설계

OpenClaw는 자체 `.env.example`(레포 루트)을 이미 제공한다. 우리는 그 컨벤션을 그대로 따르되,
필요한 4개 provider + 4개 Telegram 봇 토큰만 채운다.

```dotenv
# ── Gateway ─────────────────────────────
OPENCLAW_GATEWAY_TOKEN=              # openssl rand -hex 32 로 생성

# ── Telegram (봇 4개, 각각 BotFather에서 발급) ──
TELEGRAM_BOT_TOKEN=xxxx              # accountId: default → coding agent
TELEGRAM_BOT_TOKEN_ANALYSIS=xxxx     # openclaw.json의 channels.telegram.accounts.analysis.botToken 로 참조
TELEGRAM_BOT_TOKEN_WEB=xxxx
TELEGRAM_BOT_TOKEN_SERVER=xxxx

# ── Claude Max (CLI backend, API 키 아님) ──
# claude login 을 서버에서 먼저 수행 → ~/.claude 세션 재사용
# (OpenClaw가 자동으로 `claude` CLI를 호출하므로 별도 ANTHROPIC_API_KEY 불필요)

# ── OpenRouter ───────────────────────────
OPENROUTER_API_KEY=xxxx

# ── OpenAI API ───────────────────────────
OPENAI_API_KEY=xxxx

# ── Ollama (원격/기존 서버, 이 프로젝트가 설치 안 함) ──
OLLAMA_BASE_URL=http://<ollama-host>:11434
```

원칙(V1.0에서 정한 것과 동일하게 유지):
- 비밀값은 전부 `.env`. `openclaw.json`에는 구조(agents/bindings/tools/mcp) — 즉 "무엇을 어떻게
  연결할지"만 두고 키 원문은 두지 않는다.
- provider/모델 배정은 `openclaw.json`의 `agents.list[].model` 값만 바꾸면 코드 수정 없이 교체된다
  (V1.0에서 설계한 `models.yaml`의 역할을 OpenClaw의 `agents.list[].model` + `agents.defaults.model.fallbacks`가 대신함).

---

## 4. 리포지토리/작업 구성

```
0601/                         # 우리 작업 저장소 — 설정/커스텀 확장만 관리
├── PLAN.md
├── openclaw.json              # 실제 배포 설정 (agents/bindings/channels/tools/mcp) — 비밀값 없음
├── .env.example
├── extensions/                 # 우리가 추가하는 커스텀 OpenClaw 확장 (plugin-sdk 사용)
│   └── model-router/            # (선택/Phase 5) 대화 내 fast↔reasoning 동적 승격 훅
├── mcp-servers/                 # (선택) SQLite 전용 MCP 서버 설정 — 기본은 exec+sqlite3로 충분
├── scripts/
│   └── deploy-ubuntu.sh         # 설치 스크립트 + systemd 등록
└── docs/
    └── ops-runbook.md
```

`charliewoo1216/openclaw` (fork)는 별도 workspace(`/workspace/openclaw`)에서 관리한다.
우리가 커스텀 확장을 만들 때만 그 fork에 커밋하고, 이 `0601` 저장소에는 **배포 설정과 계획/운영 문서**를 둔다.
(두 저장소를 분리 유지할지, 하나로 합칠지는 Phase 0에서 확정)

---

## 5. 단계별 실행 계획

### Phase 0 — 기반 준비
- [x] OpenClaw 저장소 fork 및 클론 확인 (`charliewoo1216/openclaw`)
- [ ] Ubuntu 서버에 Node 22.19+/24, (선택)Docker 설치 확인
- [ ] `claude login` 수행 (Claude Max 세션 확보 — CLI backend가 이 세션을 그대로 사용)
- [ ] Telegram Bot 4개 생성 (BotFather), 토큰 4개 확보
- [ ] OpenRouter/OpenAI API 키 발급, Ollama 서버 주소 확보
- [ ] `openclaw.json` / `0601` 저장소 분리 방식 확정 (fork에 직접 두는지, 별도 config repo로 두는지)

### Phase 1 — OpenClaw 설치 + 단일 Agent 동작 확인
- [ ] Ubuntu 서버에 설치 스크립트로 OpenClaw 설치
- [ ] Telegram 기본 채널 1개 연결, 기본(main) agent로 대화 확인 (`openclaw onboard`)
- [ ] Claude CLI backend가 정상 동작하는지 확인 (Claude Max 세션으로 응답 생성되는지)

### Phase 2 — 4-Agent / 4-Bot 멀티 라우팅 구성
- [ ] `openclaw agents add coding|analysis|web|server` 로 4개 agent 생성
- [ ] `channels.telegram.accounts.<id>` 4개 등록 (각 봇 토큰), `bindings`로 agent와 연결
- [ ] agent별 1차 모델 배정 (표 2번 기준, 임의 배정 — 나중에 교체)
- [ ] `openclaw agents list --bindings`, `openclaw channels status --probe` 로 검증

### Phase 3 — Provider 4종 연결 검증
- [ ] anthropic(CLI backend), openrouter, openai, ollama 각각 최소 1개 agent에서 응답 확인
- [ ] `agents.defaults.model.fallbacks` 로 장애 시 대체 provider 체인 구성

### Phase 4 — Tool 정책 / 샌드박스 구성
- [ ] Coding agent: `tools.profile: "coding"` (exec/fs/web/sessions)
- [ ] Analysis agent: 읽기 위주 + 향후 RAG/MCP 연결 지점 확보
- [ ] Web agent: `browser` 툴 활성화, sandbox browser 네트워크 격리 확인
- [ ] Server agent: `sandbox.backend: "ssh"`로 대상 Ubuntu 서버 원격 실행 구성, `tools.elevated`는 최소한만 허용
- [ ] 각 agent `tools.allow/deny`로 과도한 권한 제거 (특히 Server agent의 쓰기/삭제성 명령)

### Phase 5 — 부족한 부분 커스텀 구현
- [ ] **DB 도구**: SQLite만 지원 — Analysis agent에 `exec` 허용 후 `sqlite3 <db파일> "<쿼리>"` 형태로 바로 사용,
  파일 경로는 sandbox workspace 내부로 제한
- [ ] (선택) 반복적으로 안전한 조회만 노출하고 싶다면 SQLite 전용 MCP 서버를 `mcp.servers`에 등록해
  임의 쓰기 쿼리를 차단
- [ ] **(선택) 동적 모델 라우팅**: 정말 필요하다면 요청 분류 후 세션 모델을 즉석에서 바꾸는 소형 훅/확장 작성
  (`session_status(model=...)` 또는 `/model` 전환 메커니즘 활용) — V1 범위에서는 생략 가능,
  agent별 고정 모델 배정으로 실용적으로 충분한지 먼저 운영하며 판단

### Phase 6 — 운영/배포 (Ubuntu, 24시간)
- [ ] 설치 스크립트 기반 systemd 서비스 등록 (또는 Docker Compose)
- [ ] 헬스체크(`openclaw channels status`, `gateway health`), 자동 재시작
- [ ] `~/.openclaw` 상태 디렉터리(세션/인증/설정) 백업
- [ ] 로그/사용량 확인 방법 정리 (OpenClaw 자체 로깅/진단 명령 활용)

---

## 6. 주요 리스크 & 결정 필요 사항

| 항목 | 리스크 | 결정 필요 |
|---|---|---|
| Claude Max 구독 사용 정책 | `docs/providers/claude-max-api-proxy.md`에 Anthropic이 Claude Code 외 구독 사용을 제한할 수 있다는 경고 존재. 단, 우리는 프록시가 아니라 OpenClaw의 **네이티브 CLI backend**(공식 지원 경로)를 쓰므로 리스크 낮음 — 그래도 확인 필요 | 최신 Anthropic 정책 재확인 |
| Task-복잡도 기반 자동 라우팅 부재 | agent당 모델이 고정이라 "간단한 질문→Fast, 복잡한 질문→Reasoning" 자동 전환은 기본 미지원 | Phase 5까지 없이 운영해보고 실제로 필요한지 판단 (많은 경우 agent별 고정 모델로 충분) |
| SQLite 파일 쓰기 권한 | `exec`로 `sqlite3` 직접 실행 시 조회뿐 아니라 쓰기/삭제 쿼리도 가능 | Analysis agent는 읽기 전용 쿼리만 쓰도록 운영 규칙/allow 목록으로 제한, 필요 시 읽기 전용 MCP 서버로 전환 |
| Server agent 권한 | SSH sandbox backend로 원격 서버 실행 시 권한 범위(쓰기/삭제/재시작) 통제 필요 | `tools.elevated`, `tools.allow/deny` 세부 정책 확정, 위험 명령 실행 전 Telegram 확인 절차 필요 여부 |
| fork 관리 | OpenClaw 업스트림이 활발히 개발 중(active, 잦은 커밋) — fork를 그대로 두면 업데이트 추적 필요 | 커스텀 확장은 `extensions/`에만 넣어 업스트림 rebase 충돌 최소화, 정기 업스트림 sync 정책 필요 |
| 비밀값 관리 | `.env` 유출 시 전체 키 노출 (기존 리스크 동일) | 파일 권한, git 제외 유지 |

---

## 7. 다음 액션

1. Phase 0 항목(Telegram 봇 4개 토큰, OpenRouter/OpenAI 키, Ollama 서버 주소, Ubuntu 서버 준비) 확보
2. `openclaw.json` 초안(4-agent/4-binding) 작성 → Phase 1~2 실제 배포 테스트
3. Analysis agent의 SQLite 접근 범위(어떤 디렉터리/파일까지 허용할지) 확정
