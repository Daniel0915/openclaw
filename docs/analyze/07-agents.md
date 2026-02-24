# OpenClaw 코드 분석 - 07. AI 에이전트 시스템

---

## 에이전트란?

에이전트 = **AI와 실제로 대화하는 주체**.

```
메시지 도착 → Gateway → Routing → Agent → AI 모델 API 호출 → 응답
```

하나의 게이트웨이에 **여러 에이전트**를 만들 수 있다:
```
agent "default"     → Anthropic Claude (기본 대화)
agent "coder"       → OpenAI GPT-4 (코딩 전문)
agent "support"     → Google Gemini (고객 지원)
agent "secretary"   → Claude (일정 관리)
```

각 채널을 다른 에이전트에 연결하는 건 라우팅에서 처리.

---

## 에이전트 설정 (config/types.agents.ts)

```typescript
type AgentConfig = {
  id: string;              // 에이전트 고유 ID
  name?: string;           // 표시 이름
  model?: string;          // AI 모델 (예: "claude-sonnet-4-5")
  provider?: string;       // 프로바이더 (예: "anthropic", "openai")

  // 시스템 프롬프트
  systemPrompt?: string;

  // AI 파라미터
  temperature?: number;    // 창의성 (0~1)
  maxTokens?: number;      // 최대 응답 길이

  // 도구/스킬
  tools?: AgentToolConfig[];
  skills?: AgentSkillConfig[];

  // 샌드박스 (코드 실행 환경)
  sandbox?: SandboxConfig;
};
```

---

## 지원 AI 프로바이더

`src/commands/auth-choice*.ts` 파일들에서 볼 수 있음:

```
apply.anthropic.ts    → Anthropic Claude
apply.openai.ts       → OpenAI GPT
apply.github-copilot.ts → GitHub Copilot
apply.google-gemini-cli.ts → Google Gemini
apply.xai.ts          → xAI (Grok)
apply.openrouter.ts   → OpenRouter (여러 모델 통합)
apply.vllm.ts         → vLLM (자체 호스팅)
apply.huggingface.ts  → HuggingFace
apply.minimax.ts      → MiniMax (중국)
apply.qwen-portal.ts  → Qwen/阿里云
apply.volcengine.ts   → Volcengine/字节跳动
apply.byteplus.ts     → BytePlus
apply.moonshot.ts     → Moonshot AI
apply.copilot-proxy.ts → Copilot Proxy
apply.oauth.ts        → OAuth 방식 (GitHub 등)
```

---

## 인증 프로파일 시스템 (auth-profiles.ts)

여러 AI 프로바이더를 관리하는 시스템.

```typescript
// auth-profiles.ts
type AuthProfile = {
  id: string;           // 프로파일 ID
  provider: string;     // 프로바이더 이름
  apiKey?: string;      // API 키
  model?: string;       // 기본 모델

  // 라운드 로빈 / 폴백 지원
  cooldown?: {
    expiresAt?: number;  // 쿨다운 만료 시각
    reason?: string;     // 쿨다운 이유 (rate limit 등)
  };
};
```

### 여러 API 키 관리

여러 API 키를 등록하면 **자동 로테이션**:
```
API Key 1 → rate limit 도달 → 쿨다운
API Key 2 → 사용 시작
API Key 3 → 대기
...
```
→ 라운드 로빈 방식으로 API 키를 돌아가며 사용

---

## 에이전트 도구 (Tools)

에이전트에게 **도구(Tool)**를 줄 수 있다.
Claude/GPT의 Function Calling 기능 활용.

```typescript
// 지원 도구들
type AgentTool =
  | "bash"         // 셸 명령 실행 (샌드박스 안에서)
  | "browser"      // 웹 브라우저 제어 (Playwright)
  | "memory"       // 장기 기억 저장/조회
  | "files"        // 파일 읽기/쓰기
  | "python"       // Python 실행
  | "nodes"        // 연결된 기기 제어
```

### Bash 도구 예시

```
사용자: "현재 디렉토리의 파일 목록 보여줘"
에이전트: (bash 도구 호출) ls -la
에이전트: "현재 디렉토리에는 다음 파일들이 있습니다: ..."
```

### 샌드박스 보안

Bash 도구는 기본적으로 **샌드박스**(Docker 컨테이너) 안에서 실행:
```typescript
// config/types.sandbox.ts
type SandboxConfig = {
  enabled: boolean;
  image?: string;        // Docker 이미지
  workDir?: string;      // 작업 디렉토리
  maxTimeout?: number;   // 최대 실행 시간
};
```

---

## 실행 승인 시스템 (Exec Approval)

위험할 수 있는 명령은 실행 전 **사용자 승인** 요청:

```
에이전트: "rm -rf /tmp/old-files 실행할까요?"
사용자: [승인] 또는 [거부]
에이전트: (승인 시) 실행
```

`src/gateway/exec-approval-manager.ts`에서 관리.

---

## 스킬 시스템 (Skills)

스킬 = 에이전트에게 추가할 수 있는 **전문 지식/능력**.

```
skills/
├── mintlify.md     ← Mintlify 문서 작성 스킬
├── PR_WORKFLOW.md  ← PR 작업 스킬
└── ...
```

스킬은 마크다운 문서 형태:
```markdown
# Mintlify 스킬

이 스킬을 사용하면 Mintlify 문서를...
(에이전트가 읽고 사용하는 시스템 프롬프트)
```

---

## Pi 에이전트 (Pi AI)

OpenClaw는 **Pi**라는 내장 에이전트 프레임워크도 포함:

```typescript
// @mariozechner/pi-agent-core
// @mariozechner/pi-ai
// @mariozechner/pi-coding-agent
// @mariozechner/pi-tui
```

Pi는 **코딩 에이전트** 기능 제공:
- 코드 작성/수정
- 파일 시스템 조작
- 테스트 실행
- Git 작업

---

## 에이전트 세션 관리

각 에이전트-사용자 조합은 독립적인 **세션**을 가짐.

```
세션 저장 위치: ~/.openclaw/agents/<agentId>/sessions/
```

세션 파일 구조 (`*.jsonl`):
```jsonl
{"role":"user","content":"안녕하세요","timestamp":"..."}
{"role":"assistant","content":"안녕하세요! 무엇을 도와드릴까요?","timestamp":"..."}
{"role":"user","content":"날씨 알려줘","timestamp":"..."}
```

JSONL = 한 줄에 JSON 하나. 대화 내역을 줄 단위로 저장.

---

## 메모리 시스템 (Memory)

에이전트는 **장기 기억**을 가질 수 있음.

```
src/memory/          ← 메모리 코어 (SQLite + 벡터 검색)
extensions/memory-core/     ← 메모리 플러그인
extensions/memory-lancedb/  ← LanceDB 기반 메모리
```

```typescript
// 메모리 저장
await memory.save("peter의 생일은 1월 1일이다");

// 메모리 검색 (벡터 유사도 검색)
const results = await memory.search("peter에 대해 알고 있는 것?");
// → "peter의 생일은 1월 1일이다" 반환
```

SQLite + `sqlite-vec` 확장으로 벡터 검색 구현.

---

## 에이전트 추가/관리 CLI

```bash
# 에이전트 목록
openclaw agents list

# 에이전트 추가
openclaw agents add --name "coder" --model claude-sonnet-4-5

# 에이전트 삭제
openclaw agents delete coder

# 에이전트로 메시지 전송 (테스트)
openclaw agent --message "안녕하세요"
```

---

## 다음: [08-config.md](./08-config.md)
