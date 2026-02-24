# OpenClaw 코드 분석 - 04. Gateway 서버 아키텍처

---

## Gateway란 무엇인가?

Gateway는 OpenClaw의 **심장부**다.

```
                    ┌─────────────────────────────────┐
                    │          Gateway Server          │
  Telegram Bot ────→│                                 │
  Discord Bot  ────→│  HTTP Server (Express v5)       │
  Slack App    ────→│  WebSocket Server (ws)          │────→ AI 모델 (Claude/GPT)
  WhatsApp     ────→│                                 │
  Web UI       ────→│  Channel Manager                │
  iOS/Android App──→│  Agent Event Handler            │
                    │  Hook System                    │
                    │  Cron Scheduler                 │
                    └─────────────────────────────────┘
```

명령으로 시작:
```bash
openclaw gateway run --bind loopback --port 18789
```

---

## Gateway 시작 흐름 (server.impl.ts)

```typescript
export async function startGatewayServer(
  opts: GatewayServerOptions
): Promise<GatewayServer> {

  // 1단계: 설정 로드
  const config = await loadConfig();
  await migrateLegacyConfig(config);

  // 2단계: 인증 확인 (AI 프로바이더 API 키 등)
  await ensureGatewayStartupAuth(config);

  // 3단계: 플러그인 로드 (채널 플러그인들)
  const pluginRegistry = await loadGatewayPlugins(config);

  // 4단계: HTTP + WebSocket 서버 시작
  const httpServer = await startHttpServer(opts);

  // 5단계: 채널 연결
  const channelManager = await createChannelManager(config, pluginRegistry);
  await channelManager.connectAll();

  // 6단계: 각종 부가 서비스 시작
  await startGatewaySidecars(/* ... */);   // 디스커버리, 헬스 체크, Tailscale 등
  startGatewayMaintenanceTimers();         // 자동 업데이트 체크, 정리 작업
  buildGatewayCronService(config);         // 스케줄 작업

  return { close: async () => { /* 종료 로직 */ } };
}
```

---

## Gateway 서버 구성 모듈

```
src/gateway/
├── server.impl.ts          ← 메인 (모든 것을 조율)
│
├── server-http.ts          ← HTTP 서버 설정
├── server-channels.ts      ← 채널 관리자
├── server-chat.ts          ← 채팅/에이전트 이벤트 핸들러
├── server-methods.ts       ← WebSocket RPC 메서드들
├── server-plugins.ts       ← 플러그인 로더
├── server-discovery.ts     ← 네트워크 자동 발견 (mDNS)
├── server-cron.ts          ← 스케줄 작업
├── server-startup.ts       ← 사이드카 서비스 시작
├── server-maintenance.ts   ← 자동 업데이트/정리
├── server-tailscale.ts     ← Tailscale VPN 통합
│
├── auth.ts                 ← 게이트웨이 인증
├── hooks.ts                ← 훅 시스템
├── session-utils.ts        ← 세션 유틸리티
├── credentials.ts          ← 인증 정보 관리
└── control-ui.ts           ← 웹 컨트롤 UI
```

---

## HTTP 엔드포인트

Gateway는 Express v5로 HTTP 서버를 제공한다.

### 주요 엔드포인트

```
GET  /health              ← 헬스 체크
GET  /v1/status           ← 상태 조회
POST /v1/chat/completions ← OpenAI 호환 Chat API (선택적)
POST /v1/responses        ← OpenAI Responses API (선택적)
GET  /                    ← Web Control UI
GET  /api/...             ← 컨트롤 패널 API
```

### OpenAI 호환 API
게이트웨이는 OpenAI API 포맷을 그대로 받을 수 있다.
→ OpenAI 클라이언트를 쓰는 앱이 OpenClaw에 바로 연결 가능!

```typescript
// server.impl.ts에서
const openAiChatCompletionsEnabled =
  opts.openAiChatCompletionsEnabled ??
  config.gateway?.http?.endpoints?.chatCompletions?.enabled ??
  false;
```

---

## WebSocket RPC 시스템

모바일 앱, CLI, Web UI는 WebSocket으로 게이트웨이와 통신한다.

### 메시지 형식 (JSON-RPC 스타일)

```typescript
// 클라이언트 → 게이트웨이 요청
{
  id: "req_123",
  method: "sessions.send",
  params: { sessionId: "...", message: "안녕하세요" }
}

// 게이트웨이 → 클라이언트 응답
{
  id: "req_123",
  result: { success: true }
}

// 게이트웨이 → 클라이언트 이벤트 (단방향)
{
  event: "session.message",
  data: { sessionId: "...", content: "안녕하세요!" }
}
```

### 주요 WebSocket 메서드들 (server-methods-list.ts)

```typescript
// 이런 메서드들이 등록되어 있음:
"sessions.send"          // 메시지 전송
"sessions.list"          // 세션 목록
"sessions.get"           // 특정 세션 조회
"models.list"            // 모델 목록
"channels.status"        // 채널 상태
"config.get"             // 설정 조회
"config.set"             // 설정 변경
"health.get"             // 헬스 체크
"gateway.restart"        // 게이트웨이 재시작
// ...
```

---

## GatewayServerOptions - 게이트웨이 설정

```typescript
type GatewayServerOptions = {
  bind?: 'loopback' | 'lan' | 'tailnet' | 'auto';
  // loopback: 127.0.0.1 (로컬만)
  // lan: 0.0.0.0 (네트워크 전체)
  // tailnet: Tailscale IP만
  // auto: loopback 우선, 없으면 LAN

  host?: string;           // 직접 호스트 지정
  port?: number;           // 기본: 18789
  controlUiEnabled?: boolean;           // 웹 UI 활성화
  openAiChatCompletionsEnabled?: boolean; // OpenAI 호환 API
  openResponsesEnabled?: boolean;         // Responses API
  auth?: GatewayAuthConfig;             // 인증 설정
};
```

---

## Channel Manager (server-channels.ts)

채널 플러그인들을 관리하는 매니저.

```typescript
// createChannelManager 의 역할:
export async function createChannelManager(
  config: OpenClawConfig,
  pluginRegistry: PluginRegistry
) {
  return {
    // 모든 채널 연결
    connectAll: async () => {
      for (const channelPlugin of listChannelPlugins()) {
        await connectChannel(channelPlugin, config);
      }
    },

    // 특정 채널만 재연결
    reconnect: async (channelId: string) => { ... },

    // 채널 상태 확인
    getStatus: (channelId: string) => { ... },
  };
}
```

---

## 훅(Hook) 시스템

훅은 특정 이벤트 발생 시 외부 스크립트/명령을 실행하는 기능.

```typescript
// hooks.ts
// 지원하는 훅 이벤트:
type HookEvent =
  | "message.received"     // 메시지 수신 시
  | "message.sent"         // 메시지 발신 시
  | "session.start"        // 세션 시작 시
  | "session.end"          // 세션 종료 시
  | "agent.response"       // AI 응답 생성 시
  | "gateway.start"        // 게이트웨이 시작 시
  | "gateway.stop";        // 게이트웨이 종료 시
```

설정 파일에서 훅 등록:
```json5
{
  "hooks": {
    "message.received": {
      "command": "/path/to/my-script.sh"
    }
  }
}
```

---

## 인증 시스템 (auth.ts)

게이트웨이는 다단계 인증을 지원한다.

```typescript
// GatewayAuthConfig
type GatewayAuthConfig = {
  enabled: boolean;
  // 지원 인증 방식:
  // - bearer token
  // - API key
  // - IP allowlist
};
```

---

## 헬스 체크

```typescript
// server/health-state.ts
export function getHealthCache(): HealthSnapshot {
  return {
    status: 'healthy' | 'degraded' | 'unhealthy',
    channels: {
      telegram: { connected: true, ... },
      discord:  { connected: false, error: '...' },
    },
    models: { ... },
    version: '2026.2.24',
  };
}
```

---

## 서비스 디스커버리 (server-discovery.ts)

같은 네트워크의 다른 기기에서 게이트웨이를 자동으로 찾을 수 있다.

```typescript
// mDNS/Bonjour를 사용해서 서비스 광고
// @homebridge/ciao 라이브러리 사용
await startGatewayDiscovery({
  port: 18789,
  name: "openclaw-gateway",
  type: "_openclaw._tcp"
});
```

---

## 핵심 패턴 요약

| 패턴 | 설명 |
|------|------|
| 서버 구성 분리 | `server-*.ts` 파일로 기능별 분리 |
| 플러그인 레지스트리 | 채널을 플러그인으로 등록/로드 |
| WebSocket RPC | JSON 메시지로 원격 메서드 호출 |
| 훅 시스템 | 이벤트 기반으로 외부 스크립트 연동 |
| OpenAI 호환 | 기존 OpenAI 클라이언트와 호환되는 API |
| 서비스 디스커버리 | mDNS로 자동 발견 |

---

## 다음: [05-channels.md](./05-channels.md)
