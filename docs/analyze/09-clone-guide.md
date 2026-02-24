# OpenClaw 코드 분석 - 09. 클론 코딩 순서 가이드

---

## 전략: 레이어별로 작은 것부터 큰 것으로

복잡한 시스템을 처음부터 만들려면 **아래에서 위로** 쌓아야 한다.
가장 단순한 것부터 시작해서 점점 기능을 추가해 나가는 방식.

```
[7단계] 채널 플러그인 (Telegram, Discord 등)
         ↑
[6단계] 라우팅 시스템
         ↑
[5단계] AI 에이전트 연결
         ↑
[4단계] Gateway HTTP + WebSocket 서버
         ↑
[3단계] 설정 시스템
         ↑
[2단계] CLI 골격
         ↑
[1단계] 프로젝트 기반 (TypeScript ESM + 빌드)
```

---

## 1단계: 프로젝트 기반 만들기

### 목표
- TypeScript ESM 프로젝트 생성
- 빌드 도구 설정
- 기본 테스트 환경

### 할 일

```bash
# 1. 프로젝트 생성
mkdir my-gateway && cd my-gateway
pnpm init

# 2. package.json 설정
# - "type": "module" 추가
# - TypeScript, tsx, tsdown 설치

# 3. tsconfig.json 생성
# - module: "NodeNext"
# - strict: true

# 4. 빌드 스크립트 추가
```

### 핵심 파일 구조

```
my-gateway/
├── package.json
├── tsconfig.json
└── src/
    └── index.ts   ← Hello World
```

### 체크리스트
- [ ] `pnpm build` 성공
- [ ] `pnpm test` 성공 (빈 테스트 파일)
- [ ] ESM import 동작 확인

---

## 2단계: CLI 골격

### 목표
- `openclaw` 명령어 만들기
- `openclaw gateway run` 같은 서브커맨드 구조

### 핵심 라이브러리
```bash
pnpm add commander
```

### 구현 순서

```typescript
// 1. bin 파일 만들기
// src/entry.ts
import('./cli/run-main.js').then(({ runCli }) => runCli(process.argv));

// 2. CLI 프로그램 빌드
// src/cli/program.ts
import { Command } from 'commander';

const program = new Command('my-gateway');
program.version('1.0.0');

// 3. 서브커맨드 등록
// src/cli/commands/gateway.ts
program
  .command('gateway')
  .command('run')
  .action(async () => {
    console.log('Gateway starting...');
  });
```

### 체크리스트
- [ ] `node bin.mjs --help` 동작
- [ ] `node bin.mjs gateway run` 동작
- [ ] `node bin.mjs config get` 동작

---

## 3단계: 설정 시스템

### 목표
- JSON5 설정 파일 읽기/쓰기
- Zod로 검증
- `config get/set` 명령어

### 핵심 라이브러리
```bash
pnpm add json5 zod
```

### 구현 순서

```typescript
// 1. 설정 타입 정의
// src/config/types.ts
import { z } from 'zod';

export const ConfigSchema = z.object({
  gateway: z.object({
    port: z.number().default(18789),
    bind: z.enum(['loopback', 'lan']).default('loopback'),
  }).optional(),
});

export type Config = z.infer<typeof ConfigSchema>;

// 2. 설정 읽기/쓰기
// src/config/io.ts
import JSON5 from 'json5';
import fs from 'fs/promises';
import os from 'os';
import path from 'path';

const CONFIG_PATH = path.join(os.homedir(), '.my-gateway', 'config.json5');

export async function loadConfig(): Promise<Config> {
  const raw = await fs.readFile(CONFIG_PATH, 'utf-8');
  const parsed = JSON5.parse(raw);
  return ConfigSchema.parse(parsed);  // Zod 검증
}

// 3. config CLI 명령어
// src/commands/config.ts
program
  .command('config get <key>')
  .action(async (key) => {
    const config = await loadConfig();
    console.log(get(config, key));  // lodash get 또는 직접 구현
  });
```

### 체크리스트
- [ ] `~/.my-gateway/config.json5` 생성/읽기
- [ ] `config get gateway.port` 동작
- [ ] `config set gateway.port 19000` 동작
- [ ] 잘못된 값 입력 시 Zod 에러 출력

---

## 4단계: Gateway HTTP + WebSocket 서버

### 목표
- Express HTTP 서버 시작
- WebSocket 서버 추가
- `/health` 엔드포인트

### 핵심 라이브러리
```bash
pnpm add express ws
pnpm add -D @types/express @types/ws
```

### 구현 순서

```typescript
// 1. HTTP 서버
// src/gateway/server.ts
import express from 'express';
import { WebSocketServer } from 'ws';
import http from 'http';

export async function startGatewayServer(opts: { port: number; bind: string }) {
  const app = express();
  app.use(express.json());

  // 헬스 체크
  app.get('/health', (req, res) => {
    res.json({ status: 'healthy', version: '1.0.0' });
  });

  const httpServer = http.createServer(app);

  // 2. WebSocket 서버
  const wss = new WebSocketServer({ server: httpServer });
  wss.on('connection', (ws) => {
    ws.on('message', (data) => {
      const msg = JSON.parse(data.toString());
      handleWsMessage(ws, msg);
    });
  });

  // 3. 서버 시작
  const host = opts.bind === 'loopback' ? '127.0.0.1' : '0.0.0.0';
  httpServer.listen(opts.port, host, () => {
    console.log(`Gateway running on ${host}:${opts.port}`);
  });

  return { close: () => httpServer.close() };
}

// 4. WebSocket 메시지 핸들러
function handleWsMessage(ws: WebSocket, msg: any) {
  const { id, method, params } = msg;

  if (method === 'health.get') {
    ws.send(JSON.stringify({ id, result: { status: 'healthy' } }));
  }
}
```

### 체크리스트
- [ ] `openclaw gateway run` 으로 서버 시작
- [ ] `curl localhost:18789/health` 동작
- [ ] WebSocket 연결 및 메시지 수신 확인
- [ ] `openclaw gateway stop` 으로 서버 종료

---

## 5단계: AI 에이전트 연결

### 목표
- Anthropic Claude API 연결
- 메시지 전송 → AI 응답 수신

### 핵심 라이브러리
```bash
pnpm add @anthropic-ai/sdk
```

### 구현 순서

```typescript
// 1. AI 클라이언트
// src/agents/ai-client.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function callAI(messages: Message[]): Promise<string> {
  const response = await client.messages.create({
    model: 'claude-sonnet-4-5',
    max_tokens: 1024,
    messages,
  });
  return response.content[0].text;
}

// 2. 에이전트 핸들러
// src/agents/agent.ts
type Message = { role: 'user' | 'assistant'; content: string };

export class Agent {
  private history: Message[] = [];

  async handleMessage(userMessage: string): Promise<string> {
    this.history.push({ role: 'user', content: userMessage });
    const reply = await callAI(this.history);
    this.history.push({ role: 'assistant', content: reply });
    return reply;
  }
}

// 3. Gateway에 에이전트 연결
// WebSocket에서 메시지 수신 시 에이전트 호출
const agent = new Agent();

if (method === 'session.send') {
  const reply = await agent.handleMessage(params.message);
  ws.send(JSON.stringify({ id, result: { reply } }));
}
```

### 체크리스트
- [ ] 환경변수 `ANTHROPIC_API_KEY` 설정
- [ ] WebSocket으로 메시지 보내면 AI 응답 수신
- [ ] 대화 맥락 유지 (멀티턴 대화)

---

## 6단계: 라우팅 시스템

### 목표
- 여러 에이전트 관리
- 어떤 채널/사용자가 어느 에이전트를 쓸지 결정

### 구현 순서

```typescript
// 1. 에이전트 레지스트리
// src/agents/registry.ts
const agents = new Map<string, Agent>();

export function registerAgent(id: string, config: AgentConfig) {
  agents.set(id, new Agent(config));
}

export function getAgent(id: string): Agent | undefined {
  return agents.get(id);
}

// 2. 라우팅 로직
// src/routing/resolve-route.ts
type RouteInput = {
  channel: string;
  userId: string;
};

export function resolveRoute(input: RouteInput, config: Config): string {
  const bindings = config.routing?.bindings ?? [];

  // 바인딩에서 매칭 탐색 (구체적인 것 우선)
  for (const binding of bindings) {
    if (binding.channel === input.channel &&
        binding.userId === input.userId) {
      return binding.agentId;
    }
  }

  // 채널 전체 바인딩
  for (const binding of bindings) {
    if (binding.channel === input.channel && !binding.userId) {
      return binding.agentId;
    }
  }

  return 'default';  // 기본 에이전트
}

// 3. 메시지 처리 시 라우팅 적용
const agentId = resolveRoute({ channel: 'telegram', userId: '123' }, config);
const agent = getAgent(agentId) ?? getAgent('default');
const reply = await agent.handleMessage(message);
```

### 체크리스트
- [ ] 여러 에이전트 등록 가능
- [ ] 바인딩 규칙에 따라 다른 에이전트 사용
- [ ] 기본 에이전트 폴백

---

## 7단계: 채널 플러그인

### 목표
- Telegram 봇 추가
- 메시지 수신 → 에이전트 → 응답 전송

### 핵심 라이브러리
```bash
pnpm add grammy
```

### 구현 순서

```typescript
// 1. 채널 인터페이스 정의
// src/channels/types.ts
interface ChannelPlugin {
  id: string;
  connect(config: any): Promise<void>;
  disconnect(): Promise<void>;
}

// 2. Telegram 플러그인
// src/telegram/plugin.ts
import { Bot } from 'grammy';

export class TelegramPlugin implements ChannelPlugin {
  id = 'telegram';
  private bot: Bot;

  async connect(config: { token: string }) {
    this.bot = new Bot(config.token);

    // 메시지 수신 핸들러
    this.bot.on('message:text', async (ctx) => {
      const message = ctx.message.text;
      const userId = String(ctx.from.id);

      // 라우팅 → 에이전트 → 응답
      const agentId = resolveRoute({
        channel: 'telegram',
        userId,
      }, config);

      const agent = getAgent(agentId);
      const reply = await agent.handleMessage(message);

      // 응답 전송
      await ctx.reply(reply);
    });

    await this.bot.start();
  }

  async disconnect() {
    await this.bot.stop();
  }
}

// 3. 채널 매니저
// src/channels/manager.ts
const plugins: ChannelPlugin[] = [];

export async function connectAllChannels(config: Config) {
  if (config.telegram?.token) {
    const telegram = new TelegramPlugin();
    await telegram.connect(config.telegram);
    plugins.push(telegram);
  }
  // Discord, Slack 등 추가 가능
}
```

### 체크리스트
- [ ] Telegram Bot 토큰 설정 후 메시지 수신
- [ ] AI 응답이 Telegram으로 전송
- [ ] 봇 연결/해제 가능

---

## 추가 단계 (고급)

이 기본 구조가 완성되면 아래를 추가할 수 있음:

| 단계 | 기능 | 난이도 |
|------|------|--------|
| 8 | Discord 채널 추가 | ⭐⭐ |
| 9 | Slack 채널 추가 | ⭐⭐ |
| 10 | 대화 히스토리 파일 저장 | ⭐⭐ |
| 11 | 여러 API 키 라운드 로빈 | ⭐⭐⭐ |
| 12 | 스트리밍 응답 (타이핑 효과) | ⭐⭐⭐ |
| 13 | 훅 시스템 | ⭐⭐⭐ |
| 14 | 웹 컨트롤 UI | ⭐⭐⭐⭐ |
| 15 | 이미지/파일 처리 | ⭐⭐⭐⭐ |
| 16 | Bash 도구 + 샌드박스 | ⭐⭐⭐⭐⭐ |
| 17 | 메모리/벡터 검색 | ⭐⭐⭐⭐⭐ |

---

## 학습 순서 추천

1. **먼저 읽어야 할 파일들** (실제 코드):

   | 순서 | 파일 | 이유 |
   |------|------|------|
   | 1 | `src/entry.ts` | 시작점, 짧고 명확 |
   | 2 | `src/cli/run-main.ts` | CLI 흐름 파악 |
   | 3 | `src/config/io.ts` | 설정 읽기 패턴 |
   | 4 | `src/config/zod-schema.ts` | Zod 스키마 패턴 |
   | 5 | `src/routing/resolve-route.ts` | 라우팅 로직 |
   | 6 | `src/routing/session-key.ts` | 세션 키 생성 |
   | 7 | `src/gateway/server.impl.ts` | 게이트웨이 전체 조율 |
   | 8 | `src/channels/plugins/types.ts` | 채널 인터페이스 |
   | 9 | `src/telegram/bot/bot-handlers.ts` | Telegram 구현 |

2. **테스트 파일도 함께 읽기**
   - 테스트가 사용 방법을 가장 잘 보여줌
   - 예: `resolve-route.test.ts` → `resolve-route.ts` 이해 도움

3. **작은 단위로 클론**
   - 전체를 한번에 복사하지 말고
   - 기능 하나씩 직접 구현해보기

---

## 핵심 패턴 정리

| 패턴 | 설명 | 예시 파일 |
|------|------|-----------|
| 플러그인 시스템 | 채널을 런타임에 동적 로드 | `channels/plugins/load.ts` |
| 어댑터 패턴 | 채널별 다른 구현을 같은 인터페이스로 | `channels/plugins/types.ts` |
| 의존성 주입 | `createDefaultDeps()`로 테스트 가능 | `cli/deps.ts` |
| 레지스트리 패턴 | 플러그인/에이전트를 Map으로 관리 | `plugins/registry.ts` |
| 세션 키 | 채널+계정+사용자로 고유 세션 식별 | `routing/session-key.ts` |
| 타입 안전 스키마 | Zod + TypeBox 이중 검증 | `config/zod-schema.ts` |
| JSONL 로그 | 대화 기록을 줄 단위 JSON으로 저장 | `sessions/` |
