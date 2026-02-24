# OpenClaw 코드 분석 - 00. 프로젝트 개요

> **목표**: OpenClaw 프로젝트 전체 구조를 이해하고, 처음부터 클론 코딩할 수 있도록 기반 지식을 쌓는다.

---

## 1. OpenClaw란 무엇인가?

**OpenClaw**는 **Multi-channel AI Gateway**다.

쉽게 말하면:

```
[사용자]
  └─ Telegram, Discord, Slack, WhatsApp, Signal 등으로 메시지를 보냄
        ↓
[OpenClaw Gateway] ← 핵심 서버
        ↓
  [AI 에이전트] ← Anthropic, OpenAI, Gemini 등 LLM에 연결
        ↓
  [응답을 다시 메시지 채널로 전달]
```

즉, **여러 메시징 앱(채널)**에서 오는 메시지를 받아서 **AI 에이전트**에게 전달하고, 응답을 다시 채널로 보내주는 **중간 서버(게이트웨이)**다.

---

## 2. 핵심 개념 3가지

### ① Gateway (게이트웨이)
- 프로젝트의 **심장부**
- HTTP + WebSocket 서버
- 모든 채널의 메시지를 받아 라우팅
- AI 모델 호출 및 응답 처리

### ② Channel (채널)
- 메시지가 오고 가는 **통신 수단**
- 내장 채널: Telegram, Discord, Slack, WhatsApp, Signal, iMessage, LINE
- 확장 채널(extension): MS Teams, Matrix, Zalo, Twitch, IRC 등
- 채널마다 **플러그인** 형태로 분리되어 있음

### ③ Agent (에이전트)
- AI 대화를 담당하는 **주체**
- 각 채널마다 다른 에이전트를 연결할 수 있음
- Anthropic Claude, OpenAI GPT, Gemini 등 다양한 LLM 프로바이더 지원

---

## 3. 주요 기술 스택

| 구분 | 기술 | 역할 |
|------|------|------|
| 언어 | TypeScript (strict) | 타입 안전성 |
| 런타임 | Node.js 22+ / Bun | 실행 환경 |
| 패키지 매니저 | pnpm | 의존성 관리 |
| 모듈 시스템 | ESM (`"type": "module"`) | import/export |
| 빌드 도구 | tsdown (esbuild 기반) | 번들링 |
| 타입 체크 | TypeScript + tsgo | 정적 분석 |
| 린트/포맷 | Oxlint + Oxfmt | 코드 품질 |
| 테스트 | Vitest + V8 | 유닛/통합 테스트 |
| CLI | Commander.js | 명령어 처리 |
| 설정 | JSON5 + Zod | 설정 파일 검증 |
| 스키마 | TypeBox (@sinclair/typebox) | 런타임 타입 검증 |

---

## 4. 지원 메시징 채널 목록

### 내장 채널 (src/ 안에 있음)
- `src/telegram/` - Telegram 봇
- `src/discord/` - Discord 봇
- `src/slack/` - Slack 앱
- `src/signal/` - Signal 메신저
- `src/imessage/` - iMessage (macOS 전용)
- `src/whatsapp/` - WhatsApp Web
- `src/line/` - LINE 메신저
- `src/web/` - Web UI

### 확장 채널 (extensions/ 안에 있음)
- `extensions/msteams/` - Microsoft Teams
- `extensions/matrix/` - Matrix
- `extensions/discord/` - Discord (별도 버전)
- `extensions/telegram/` - Telegram (별도 버전)
- `extensions/slack/` - Slack (별도 버전)
- `extensions/zalo/` - Zalo
- `extensions/voice-call/` - 음성 통화
- `extensions/irc/` - IRC

---

## 5. 지원 AI 프로바이더

```
Anthropic Claude (기본)
OpenAI GPT
Google Gemini
xAI
HuggingFace
GitHub Copilot
OpenRouter
vLLM (자체 호스팅)
Amazon Bedrock
BytePlus
MiniMax
Qwen (阿里云)
Moonshot
Volcengine
```

---

## 6. 전체 디렉토리 지도

```
openclaw/
├── src/                    ← 핵심 소스 코드
│   ├── entry.ts            ← CLI 시작점 (가장 먼저 실행)
│   ├── cli/                ← CLI 명령어 처리
│   ├── commands/           ← 각 CLI 서브커맨드 구현
│   ├── gateway/            ← 게이트웨이 서버 (핵심!)
│   ├── channels/           ← 채널 공통 로직
│   ├── routing/            ← 메시지 라우팅
│   ├── agents/             ← AI 에이전트
│   ├── config/             ← 설정 관리
│   ├── plugins/            ← 플러그인 시스템
│   ├── plugin-sdk/         ← 플러그인 개발 SDK
│   ├── telegram/           ← Telegram 채널
│   ├── discord/            ← Discord 채널
│   ├── slack/              ← Slack 채널
│   ├── signal/             ← Signal 채널
│   ├── imessage/           ← iMessage 채널
│   ├── whatsapp/           ← WhatsApp 채널
│   ├── line/               ← LINE 채널
│   ├── web/                ← Web 채널
│   ├── infra/              ← 인프라 유틸리티
│   ├── terminal/           ← 터미널 UI 헬퍼
│   ├── logging/            ← 로깅
│   ├── media/              ← 미디어 처리
│   ├── memory/             ← 메모리/벡터 DB
│   └── tts/                ← Text-to-Speech
│
├── extensions/             ← 확장 채널 플러그인 (별도 패키지)
│   ├── msteams/
│   ├── matrix/
│   ├── voice-call/
│   └── ...
│
├── apps/                   ← 네이티브 앱
│   ├── ios/                ← iOS 앱 (Swift)
│   ├── android/            ← Android 앱 (Kotlin)
│   ├── macos/              ← macOS 메뉴바 앱 (Swift)
│   └── shared/             ← 공유 Swift 코드
│
├── ui/                     ← Web UI (Lit + TypeScript)
├── docs/                   ← 문서 (Mintlify)
├── scripts/                ← 빌드/배포 스크립트
├── packages/               ← 공유 패키지
└── skills/                 ← AI 에이전트 스킬 정의
```

---

## 7. 어떻게 동작하는가? (전체 흐름)

```
1. 사용자가 npm install -g openclaw 로 설치
2. openclaw gateway run 명령으로 게이트웨이 시작
3. Telegram/Discord/Slack 등 채널 봇 설정
4. 사용자가 Telegram에서 "@bot 질문해줘" 전송
5. Telegram 봇이 메시지 수신 → Gateway로 전달
6. Gateway가 라우팅 규칙에 따라 에이전트 선택
7. 에이전트가 Claude/GPT 등에 API 호출
8. AI 응답을 Telegram으로 다시 전송
```

---

## 다음 단계

| 파일 | 내용 |
|------|------|
| [01-tech-stack.md](./01-tech-stack.md) | 기술 스택 상세 (TypeScript ESM, tsdown, Vitest) |
| [02-project-structure.md](./02-project-structure.md) | 디렉토리 구조 상세 분석 |
| [03-entry-cli.md](./03-entry-cli.md) | CLI 진입점 흐름 (entry.ts → run-main → program) |
| [04-gateway.md](./04-gateway.md) | Gateway 서버 아키텍처 |
| [05-channels.md](./05-channels.md) | 채널 플러그인 시스템 |
| [06-routing.md](./06-routing.md) | 메시지 라우팅 |
| [07-agents.md](./07-agents.md) | AI 에이전트 시스템 |
| [08-config.md](./08-config.md) | 설정 시스템 |
| [09-clone-guide.md](./09-clone-guide.md) | 클론 코딩 순서 가이드 |
