# OpenClaw 코드 분석 - 08. 설정 시스템

---

## 설정 파일 위치

```
~/.openclaw/
├── config.json5        ← 메인 설정 파일 (JSON5 형식)
├── credentials/        ← API 키 등 민감한 정보
│   ├── anthropic.json
│   └── openai.json
├── sessions/           ← 채널 세션 상태
├── agents/             ← 에이전트별 데이터
│   └── <agentId>/
│       └── sessions/
└── ...
```

---

## JSON5란?

JSON + 주석 + 후행 콤마 + 싱글 쿼트 지원.

```json5
// JSON5 예시 - 일반 JSON보다 편하게 작성 가능
{
  // 이렇게 주석 가능!
  "gateway": {
    "port": 18789,  // 포트 번호 (후행 콤마 허용)
    "bind": "loopback",
  },

  // 배열 마지막 요소 뒤에 콤마 허용
  "agents": [
    { "id": "default" },
    { "id": "coder" },
  ]
}
```

---

## 설정 구조 (config/types.ts)

설정 타입은 여러 파일로 분리되어 있음:

```
src/config/
├── types.ts          ← 전체 설정 타입 (모두 re-export)
├── types.agents.ts   ← 에이전트 설정 타입
├── types.auth.ts     ← 인증 설정 타입
├── types.channels.ts ← 채널 설정 타입
├── types.gateway.ts  ← 게이트웨이 설정 타입
├── types.telegram.ts ← Telegram 전용 설정 타입
├── types.discord.ts  ← Discord 전용 설정 타입
├── types.slack.ts    ← Slack 전용 설정 타입
├── types.models.ts   ← 모델 설정 타입
├── types.hooks.ts    ← 훅 설정 타입
├── types.sandbox.ts  ← 샌드박스 설정 타입
├── types.memory.ts   ← 메모리 설정 타입
└── ...
```

### 최상위 설정 타입 (OpenClawConfig)

```typescript
type OpenClawConfig = {
  gateway?: GatewayConfig;       // 게이트웨이 설정
  agents?: AgentConfig[];        // 에이전트 목록
  routing?: RoutingConfig;       // 라우팅 규칙
  models?: ModelConfig;          // AI 모델 설정
  channels?: ChannelsConfig;     // 채널 설정
  plugins?: PluginsConfig;       // 플러그인 설정
  hooks?: HooksConfig;           // 훅 설정
  sandbox?: SandboxConfig;       // 샌드박스 설정
  memory?: MemoryConfig;         // 메모리 설정
  // 각 채널별 설정
  telegram?: TelegramConfig;
  discord?: DiscordConfig;
  slack?: SlackConfig;
  signal?: SignalConfig;
  whatsapp?: WhatsAppConfig;
  // ...
};
```

---

## 설정 파일 예시

```json5
// ~/.openclaw/config.json5
{
  // 게이트웨이 기본 설정
  "gateway": {
    "mode": "local",           // local | remote
    "port": 18789,
    "bind": "loopback",        // loopback | lan | auto
    "controlUi": {
      "enabled": true          // 웹 컨트롤 UI 활성화
    }
  },

  // AI 모델 기본값
  "models": {
    "default": "claude-sonnet-4-5"
  },

  // 에이전트 설정
  "agents": [
    {
      "id": "default",
      "name": "Default Agent",
      "model": "claude-sonnet-4-5"
    },
    {
      "id": "coder",
      "name": "Coding Agent",
      "model": "claude-opus-4-6",
      "tools": ["bash", "files"]
    }
  ],

  // 라우팅 규칙
  "routing": {
    "bindings": [
      {
        "agentId": "coder",
        "channel": "discord",
        "peer": { "kind": "channel", "id": "coding-channel-id" }
      }
    ]
  },

  // Telegram 채널 설정
  "telegram": {
    "accounts": [
      {
        "token": "1234567890:AAFxxxxx",
        "allowFrom": ["user:123456789"]
      }
    ]
  },

  // Discord 채널 설정
  "discord": {
    "accounts": [
      {
        "token": "MTE..."
      }
    ]
  }
}
```

---

## 설정 I/O (config/io.ts)

```typescript
// 설정 읽기
export async function loadConfig(): Promise<OpenClawConfig> {
  const raw = await fs.readFile(CONFIG_PATH, 'utf-8');
  const parsed = parseConfigJson5(raw);    // JSON5 파싱
  await validateConfigObject(parsed);      // Zod 검증
  return parsed;
}

// 설정 쓰기
export async function writeConfigFile(
  config: OpenClawConfig
): Promise<void> {
  const json5 = JSON5.stringify(config, null, 2);
  await fs.writeFile(CONFIG_PATH, json5, 'utf-8');
}
```

---

## 설정 검증 (Zod)

설정을 읽을 때 **Zod**로 타입 검증:

```typescript
// config/zod-schema.ts
import { z } from 'zod';

export const OpenClawSchema = z.object({
  gateway: z.object({
    port: z.number().min(1).max(65535).optional(),
    bind: z.enum(['loopback', 'lan', 'auto', 'tailnet']).optional(),
    // ...
  }).optional(),

  agents: z.array(AgentSchema).optional(),
  // ...
});

// 검증 함수
export async function validateConfigObject(
  config: unknown
): Promise<OpenClawConfig> {
  return OpenClawSchema.parse(config);  // 실패 시 ZodError 던짐
}
```

---

## 설정 마이그레이션 (legacy-migrate.ts)

이전 버전 설정을 자동으로 최신 형식으로 변환:

```typescript
// legacy-migrate.ts
export async function migrateLegacyConfig(
  config: OpenClawConfig
): Promise<void> {
  // 옛날 키 이름을 새 키 이름으로 변환
  // 예: "clawd.token" → "telegram.accounts[0].token"
}
```

---

## 설정 런타임 오버라이드 (runtime-overrides.ts)

환경변수로 설정 일부를 오버라이드 가능:

```bash
# 환경변수로 포트 오버라이드
OPENCLAW_PORT=19000 openclaw gateway run

# 특정 채널 건너뛰기
OPENCLAW_SKIP_CHANNELS=1 openclaw gateway run
```

---

## CLI로 설정 조작

```bash
# 설정 값 조회
openclaw config get gateway.port

# 설정 값 변경
openclaw config set gateway.port 19000

# 설정 값 삭제
openclaw config unset gateway.port

# 설정 파일 직접 열기
openclaw config edit
```

---

## 설정 경로 (config/paths.ts)

```typescript
// config/paths.ts
export const CONFIG_DIR = path.join(os.homedir(), '.openclaw');
export const CONFIG_PATH = path.join(CONFIG_DIR, 'config.json5');
export const CREDENTIALS_DIR = path.join(CONFIG_DIR, 'credentials');
export const SESSIONS_DIR = path.join(CONFIG_DIR, 'sessions');
export const AGENTS_DIR = path.join(CONFIG_DIR, 'agents');
```

---

## 다음: [09-clone-guide.md](./09-clone-guide.md)
