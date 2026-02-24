# OpenClaw 코드 분석 - 02. 프로젝트 구조 상세

---

## 전체 디렉토리 구조 (레이어별 설명)

```
openclaw/
│
├── 📄 package.json          ← 프로젝트 메타데이터 + 스크립트
├── 📄 tsconfig.json         ← TypeScript 설정
├── 📄 tsdown.config.ts      ← 번들러 설정
├── 📄 vitest.*.config.ts    ← 테스트 설정들
├── 📄 openclaw.mjs          ← CLI 바이너리 진입점 (bin)
│
├── 📁 src/                  ← [핵심] 모든 TypeScript 소스
├── 📁 extensions/           ← [확장] 채널 플러그인 패키지들
├── 📁 apps/                 ← [앱] iOS/Android/macOS 네이티브 앱
├── 📁 ui/                   ← [UI] 웹 컨트롤 패널 (Lit 기반)
├── 📁 docs/                 ← [문서] Mintlify 문서
├── 📁 scripts/              ← [스크립트] 빌드/배포/유틸 스크립트
├── 📁 packages/             ← [공유] 공유 패키지
├── 📁 skills/               ← [스킬] AI 에이전트 스킬 정의
├── 📁 assets/               ← 이미지 등 정적 파일
├── 📁 test/                 ← 전역 테스트 유틸리티
└── 📁 dist/                 ← [빌드 결과] 빌드 후 생성됨
```

---

## src/ 내부 구조 (레이어 설명)

### 레이어 1: 진입점

```
src/
├── entry.ts       ← 최초 실행 파일 (CLI 시작점)
├── index.ts       ← 라이브러리로 import할 때의 공개 API
├── runtime.ts     ← 런타임 환경 정의
└── globals.ts     ← 전역 변수/설정
```

### 레이어 2: CLI 레이어 (사용자 명령어 처리)

```
src/
├── cli/           ← CLI 인프라 (프로그램 구조, 옵션, 라우팅)
│   ├── program/   ← Commander 프로그램 빌더
│   ├── run-main.ts ← CLI 메인 진입점
│   ├── argv.ts    ← 명령어 인수 파싱
│   ├── route.ts   ← 서브커맨드 라우팅
│   ├── deps.ts    ← 의존성 주입 팩토리
│   ├── progress.ts ← 프로그레스 바
│   └── ...
│
└── commands/      ← 실제 CLI 커맨드 구현
    ├── gateway-*.ts   ← gateway 관련 커맨드
    ├── agent*.ts      ← agent 커맨드
    ├── channels*.ts   ← channels 커맨드
    ├── status*.ts     ← status 커맨드
    ├── onboard*.ts    ← 초기 설정 마법사
    ├── models*.ts     ← AI 모델 관리
    ├── sessions*.ts   ← 세션 관리
    ├── doctor*.ts     ← 진단 도구
    └── ...
```

### 레이어 3: 핵심 비즈니스 로직

```
src/
├── gateway/       ← [핵심] HTTP+WebSocket 게이트웨이 서버
│   ├── server.impl.ts   ← 게이트웨이 서버 구현 (메인)
│   ├── server-channels.ts ← 채널 관리
│   ├── server-chat.ts   ← 채팅/에이전트 이벤트
│   ├── server-http.ts   ← HTTP 엔드포인트
│   ├── server-methods.ts ← WebSocket 메서드 핸들러
│   ├── auth.ts          ← 인증
│   ├── hooks.ts         ← 훅 시스템
│   └── ...
│
├── channels/      ← 채널 공통 로직 (모든 채널이 공유)
│   ├── plugins/   ← 채널 플러그인 레지스트리
│   ├── session.ts ← 채널 세션 관리
│   ├── allow-from.ts ← 수신 허용 규칙
│   └── ...
│
├── routing/       ← 메시지 라우팅 (어느 에이전트로 보낼지)
│   ├── resolve-route.ts ← 라우팅 결정 로직
│   ├── session-key.ts   ← 세션 키 생성
│   └── bindings.ts      ← 채널-에이전트 바인딩
│
└── agents/        ← AI 에이전트 시스템
    ├── auth-profiles.ts ← AI 프로바이더 인증
    ├── bash-tools.ts    ← Bash 도구
    └── ...
```

### 레이어 4: 채널 구현

```
src/
├── telegram/      ← Telegram 봇 구현
├── discord/       ← Discord 봇 구현
├── slack/         ← Slack 앱 구현
├── signal/        ← Signal 메신저
├── imessage/      ← iMessage (macOS)
├── whatsapp/      ← WhatsApp Web
├── line/          ← LINE 메신저
└── web/           ← Web 인터페이스
```

### 레이어 5: 인프라/유틸리티

```
src/
├── config/        ← 설정 파일 읽기/쓰기/검증
├── infra/         ← 저수준 인프라 (env, paths, errors 등)
├── logging/       ← 로깅 시스템
├── terminal/      ← 터미널 UI 헬퍼 (테이블, 색상 팔레트)
├── media/         ← 미디어 파일 처리 (이미지, 오디오, 영상)
├── memory/        ← 메모리/벡터 DB (RAG 기능)
├── tts/           ← Text-to-Speech
├── plugins/       ← 플러그인 런타임
├── plugin-sdk/    ← 플러그인 개발자용 SDK (공개 API)
├── process/       ← 프로세스 관리
├── hooks/         ← 훅 시스템
├── sessions/      ← 세션 스토리지
└── shared/        ← 공유 유틸리티
```

---

## extensions/ 구조

extensions/는 각각 독립적인 npm 패키지다.
채널 플러그인들이 여기 있음.

```
extensions/
├── msteams/       ← @openclaw/msteams
├── matrix/        ← @openclaw/matrix
├── discord/       ← @openclaw/discord (확장 버전)
├── telegram/      ← @openclaw/telegram (확장 버전)
├── slack/         ← @openclaw/slack (확장 버전)
├── voice-call/    ← @openclaw/voice-call
├── zalo/          ← @openclaw/zalo
├── zalouser/      ← @openclaw/zalouser
├── irc/           ← @openclaw/irc
├── line/          ← @openclaw/line
├── bluebubbles/   ← @openclaw/bluebubbles (iMessage via BlueBubbles)
├── whatsapp/      ← @openclaw/whatsapp (확장 버전)
├── feishu/        ← @openclaw/feishu (飞书/Lark)
└── ...
```

각 extension 구조:
```
extensions/msteams/
├── package.json        ← 독립적인 패키지
├── src/
│   ├── index.ts        ← 플러그인 진입점
│   ├── plugin.ts       ← 플러그인 정의
│   └── ...
└── tsconfig.json
```

---

## apps/ 구조

```
apps/
├── ios/           ← iOS 앱 (Swift + SwiftUI)
│   ├── Sources/
│   └── project.yml (XcodeGen)
│
├── android/       ← Android 앱 (Kotlin + Gradle)
│   └── app/
│       ├── build.gradle.kts
│       └── src/
│
├── macos/         ← macOS 메뉴바 앱 (Swift + SwiftUI)
│   └── Sources/OpenClaw/
│
└── shared/        ← iOS/macOS 공유 코드 (OpenClawKit)
    └── OpenClawKit/Sources/
```

모바일/데스크탑 앱은 WebSocket으로 Gateway에 연결함.

---

## ui/ 구조

```
ui/
├── src/
│   ├── components/    ← 웹 UI 컴포넌트 (Lit Web Components)
│   └── ...
└── package.json
```

Web UI는 **Lit** (Google의 Web Components 라이브러리) 사용.
브라우저에서 Gateway를 제어하는 컨트롤 패널.

---

## 파일 명명 규칙

OpenClaw는 **파일 이름으로 역할을 나타내는** 패턴을 사용:

| 패턴 | 의미 | 예시 |
|------|------|------|
| `*.ts` | 구현 파일 | `resolve-route.ts` |
| `*.test.ts` | 유닛 테스트 | `resolve-route.test.ts` |
| `*.e2e.test.ts` | E2E 테스트 | `gateway.e2e.test.ts` |
| `*.live.test.ts` | 실제 API 테스트 | `anthropic.setup-token.live.test.ts` |
| `types.*.ts` | 타입 전용 파일 | `types.agents.ts` |
| `server-*.ts` | 서버 관련 모듈 | `server-channels.ts` |
| `register.*.ts` | 커맨드 등록 | `register.agent.ts` |

---

## 다음: [03-entry-cli.md](./03-entry-cli.md)
