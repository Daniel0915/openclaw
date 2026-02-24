# OpenClaw 클론 코딩 학습 자료

> 이 폴더는 [openclaw/openclaw](https://github.com/openclaw/openclaw) 레포지토리를 분석한 학습 자료입니다.

---

## 파일 목록 (읽는 순서대로)

| 파일 | 내용 |
|------|------|
| [00-overview.md](./00-overview.md) | 프로젝트 개요 - OpenClaw란 무엇인가 |
| [01-tech-stack.md](./01-tech-stack.md) | 기술 스택 - TypeScript ESM, tsdown, Vitest |
| [02-project-structure.md](./02-project-structure.md) | 디렉토리 구조 상세 분석 |
| [03-entry-cli.md](./03-entry-cli.md) | CLI 진입점 흐름 (entry.ts → CLI) |
| [04-gateway.md](./04-gateway.md) | Gateway 서버 아키텍처 |
| [05-channels.md](./05-channels.md) | 채널 플러그인 시스템 |
| [06-routing.md](./06-routing.md) | 메시지 라우팅 |
| [07-agents.md](./07-agents.md) | AI 에이전트 시스템 |
| [08-config.md](./08-config.md) | 설정 시스템 (JSON5 + Zod) |
| [09-clone-guide.md](./09-clone-guide.md) | 클론 코딩 순서 가이드 |

---

## 한 줄 요약

**OpenClaw = 멀티 채널 AI 게이트웨이**
- Telegram, Discord, Slack 등 메시지 채널을 Claude/GPT 등 AI에 연결
- TypeScript ESM + Node.js 22 + pnpm 모노레포
- 채널은 플러그인 형태 → 새 채널 추가 용이

---

## 클론 코딩 시작 순서

```
1단계: TypeScript ESM 프로젝트 세팅
2단계: CLI 골격 (Commander.js)
3단계: 설정 시스템 (JSON5 + Zod)
4단계: HTTP + WebSocket Gateway 서버
5단계: AI 에이전트 연결 (Anthropic SDK)
6단계: 라우팅 시스템
7단계: 첫 채널 플러그인 (Telegram grammy)
```

자세한 내용은 [09-clone-guide.md](./09-clone-guide.md) 참고.
