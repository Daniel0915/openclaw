# OpenClaw 코드 분석 - 01. 기술 스택 상세

---

## 1. TypeScript ESM 프로젝트

### ESM이란?
ESM = ECMAScript Modules. 브라우저/Node.js 표준 모듈 시스템.

```json
// package.json
{
  "type": "module"  // ← 이 프로젝트는 ESM 사용
}
```

CommonJS(구방식) vs ESM(신방식):
```js
// CommonJS (구방식) - require
const express = require('express')
module.exports = { foo }

// ESM (신방식) - import/export
import express from 'express'
export { foo }
```

### tsconfig.json 핵심 설정

```json
{
  "compilerOptions": {
    "module": "NodeNext",        // Node.js ESM 방식으로 모듈 처리
    "moduleResolution": "NodeNext", // .js 확장자를 명시해야 함
    "target": "es2023",          // 최신 JS 문법 사용
    "strict": true,              // 엄격 타입 체크
    "noEmit": true,              // 타입 체크만, 빌드는 tsdown이 담당
    "experimentalDecorators": true, // @Decorator 문법 허용
    "paths": {
      // 플러그인 SDK 경로 별칭
      "openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"]
    }
  }
}
```

**중요**: `"noEmit": true` → TypeScript 컴파일러는 타입 체크만 하고,
실제 번들링은 **tsdown**이 담당한다.

---

## 2. 빌드 도구: tsdown

tsdown은 esbuild 기반의 빠른 번들러.

### 빌드 과정 (package.json scripts.build)

```
pnpm build
  ↓
1. canvas:a2ui:bundle   → Canvas UI 번들 생성
2. tsdown               → TypeScript → dist/ 번들링 (핵심)
3. build:plugin-sdk:dts → 플러그인 SDK 타입 정의 생성
4. write-plugin-sdk-entry-dts → SDK entry 타입 생성
5. canvas-a2ui-copy     → Canvas 파일 복사
6. copy-hook-metadata   → 훅 메타데이터 복사
7. copy-export-html     → HTML 템플릿 복사
8. write-build-info     → 빌드 정보 기록
9. write-cli-compat     → CLI 호환성 파일 생성
```

### 결과물 (dist/)
```
dist/
├── index.js        ← 메인 라이브러리 (import 시 사용)
├── entry.js        ← CLI 실행 엔트리
├── plugin-sdk/     ← 플러그인 개발 SDK
└── ...
```

---

## 3. 패키지 매니저: pnpm

pnpm = 성능 좋은 npm 대안. 심볼릭 링크 기반으로 디스크 공간 절약.

### 왜 pnpm인가?
- 모노레포 지원 (`pnpm-workspace.yaml`)
- 빠른 설치 속도
- 엄격한 의존성 격리

### pnpm-workspace.yaml
```yaml
packages:
  - extensions/*    # extensions/ 하위 모든 패키지
  - packages/*      # packages/ 하위 모든 패키지
  - apps/*          # apps/ 하위 모든 패키지
```
→ **모노레포** 구조: 하나의 git 저장소에 여러 패키지가 있음

---

## 4. 테스트: Vitest

Vitest = Vite 기반의 빠른 테스트 프레임워크 (Jest와 API 호환).

### 테스트 설정 파일 분리

| 파일 | 용도 |
|------|------|
| `vitest.unit.config.ts` | 유닛 테스트 (빠름) |
| `vitest.e2e.config.ts` | E2E 테스트 |
| `vitest.live.config.ts` | 실제 API 키로 테스트 |
| `vitest.gateway.config.ts` | 게이트웨이 테스트 |
| `vitest.extensions.config.ts` | 확장 플러그인 테스트 |

### 테스트 실행 방법

```bash
pnpm test             # 전체 테스트 (병렬)
pnpm test:fast        # 유닛 테스트만
pnpm test:coverage    # 커버리지 포함
```

### 테스트 파일 위치
소스 파일과 **같은 위치**에 `.test.ts` 파일을 둠 (코로케이션 패턴):
```
src/routing/resolve-route.ts
src/routing/resolve-route.test.ts  ← 같은 디렉토리
```

---

## 5. 린트/포맷: Oxlint + Oxfmt

### Oxlint
Rust로 만든 매우 빠른 JavaScript/TypeScript 린터.
ESLint의 대안.

```bash
pnpm lint           # 린트 체크
pnpm lint:fix       # 자동 수정
```

### Oxfmt
Rust로 만든 포맷터. Prettier의 대안.

```bash
pnpm format:check   # 포맷 확인
pnpm format:fix     # 자동 포맷
```

---

## 6. 주요 런타임 의존성

### CLI/터미널
- `commander` - CLI 명령어 파서 (`program.ts`에서 사용)
- `@clack/prompts` - 예쁜 인터랙티브 CLI 프롬프트
- `chalk` - 터미널 색상
- `osc-progress` - 프로그레스 바

### 웹/HTTP
- `express` v5 - HTTP 서버
- `ws` - WebSocket 서버
- `undici` - 고성능 HTTP 클라이언트

### 데이터 처리
- `zod` v4 - 런타임 타입 검증/파싱
- `@sinclair/typebox` - JSON Schema + 런타임 타입 (API 스키마)
- `json5` - 주석 허용하는 JSON (설정 파일에 사용)
- `yaml` - YAML 파서

### 채널 SDK들
- `grammy` - Telegram Bot API
- `@slack/bolt` - Slack Bot
- `@buape/carbon` - Discord Bot
- `@line/bot-sdk` - LINE Bot
- `@whiskeysockets/baileys` - WhatsApp Web

### AI/미디어
- `pdfjs-dist` - PDF 처리
- `sharp` - 이미지 처리
- `playwright-core` - 헤드리스 브라우저
- `node-edge-tts` - Text-to-Speech
- `sqlite-vec` - 벡터 DB (메모리 기능)

### 네트워킹
- `@homebridge/ciao` - mDNS/Bonjour 서비스 디스커버리
- `ipaddr.js` - IP 주소 처리

---

## 7. 개발 시 알아야 할 패턴

### ESM에서 import 시 .js 확장자 필수
```typescript
// 틀린 방법
import { foo } from './utils'

// 올바른 방법 (TypeScript 파일이어도 .js로 씀)
import { foo } from './utils.js'
```

### 왜 .ts가 아니라 .js인가?
TypeScript `NodeNext` 모드에서는 런타임(Node.js)이 실제로 실행할
파일 경로로 import해야 한다. 빌드 후 파일은 `.js`이므로 `.js`를 씀.

---

## 다음: [02-project-structure.md](./02-project-structure.md)
