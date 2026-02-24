# OpenClaw 코드 분석 - 05. 채널 플러그인 시스템

---

## 채널이란?

채널 = 사용자와 AI가 메시지를 주고받는 **통신 수단**.

```
Telegram  Discord  Slack  WhatsApp  Signal  iMessage  LINE
    ↓         ↓      ↓       ↓        ↓        ↓       ↓
    └─────────┴──────┴───────┴────────┴────────┘
                     Channel Plugin Interface
                            ↓
                     Gateway 내부에서 통합 처리
```

**핵심 아이디어**: 모든 채널이 동일한 인터페이스(ChannelPlugin)를 구현.
게이트웨이는 인터페이스만 보고 어떤 채널인지 신경 쓰지 않음.

---

## 채널 플러그인 인터페이스

채널 플러그인이 구현해야 하는 인터페이스들:
(`src/channels/plugins/types.ts`)

```typescript
// 채널 플러그인의 핵심 구조
type ChannelPlugin = {
  id: ChannelId;           // 채널 고유 ID (예: "telegram", "discord")
  meta: ChannelMeta;       // 채널 메타정보 (이름, 아이콘, 설명 등)

  // 어댑터들 (각 기능별 인터페이스)
  messaging?: ChannelMessagingAdapter;  // 메시지 송수신
  setup?: ChannelSetupAdapter;          // 초기 설정
  auth?: ChannelAuthAdapter;            // 인증
  status?: ChannelStatusAdapter;        // 상태 확인
  pairing?: ChannelPairingAdapter;      // 기기 페어링
  gateway?: ChannelGatewayAdapter;      // 게이트웨이 연결
  outbound?: ChannelOutboundAdapter;    // 발신 메시지
  streaming?: ChannelStreamingAdapter;  // 스트리밍 응답
  threading?: ChannelThreadingAdapter;  // 스레드/답장
  security?: ChannelSecurityAdapter;    // 보안/허용 목록
  heartbeat?: ChannelHeartbeatAdapter;  // 연결 유지
  command?: ChannelCommandAdapter;      // 명령어 처리
  mention?: ChannelMentionAdapter;      // 멘션 처리
  directory?: ChannelDirectoryAdapter;  // 연락처 목록
  group?: ChannelGroupAdapter;          // 그룹/서버 관리
};
```

### 어댑터 패턴이란?

어댑터 = 채널별로 다른 구현을 같은 인터페이스로 감싸는 패턴.

```
Telegram 봇  →  ChannelMessagingAdapter  →  Gateway
Discord 봇   →  ChannelMessagingAdapter  →  Gateway
Slack 앱     →  ChannelMessagingAdapter  →  Gateway
```

게이트웨이는 항상 `ChannelMessagingAdapter.send()` 만 호출.
실제로 Telegram API를 쓰는지, Discord API를 쓰는지는 어댑터가 알아서 처리.

---

## 채널 플러그인 등록/로드 흐름

```
1. Gateway 시작 시 loadGatewayPlugins() 호출
      ↓
2. src/channels/plugins/load.ts 실행
      ↓
3. 설정 파일에서 활성화된 채널 목록 확인
      ↓
4. 각 채널 플러그인을 동적으로 import
      ↓
5. PluginRegistry에 등록
      ↓
6. listChannelPlugins()로 언제든 조회 가능
```

### PluginRegistry 구조

```typescript
type PluginRegistry = {
  channels: Array<{
    plugin: ChannelPlugin;
    config: ChannelConfig;
  }>;
  // ...
};
```

---

## 채널 구현 예시: Telegram

```
src/telegram/
├── accounts.ts        ← Telegram 계정 관리
├── bot/               ← 봇 구현
│   ├── bot-handlers.ts ← 메시지/명령어 핸들러
│   └── ...
├── api-logging.ts     ← API 로그
├── audit.ts           ← 감사 로그
└── allowed-updates.ts ← 허용된 업데이트 유형
```

Telegram은 **grammy** 라이브러리 사용:
```typescript
import { Bot } from 'grammy';

const bot = new Bot(config.telegram.token);

// 메시지 수신 핸들러
bot.on('message:text', async (ctx) => {
  const text = ctx.message.text;
  const from = ctx.from.id;

  // Gateway로 메시지 전달
  await gateway.handleIncomingMessage({
    channel: 'telegram',
    from: String(from),
    text,
  });
});
```

---

## 채널 구현 예시: Discord

```
src/discord/
├── accounts.ts     ← Discord 계정/서버 관리
├── api.ts          ← Discord API 래퍼
├── audit.ts        ← 감사 로그
├── chunk.ts        ← 긴 메시지 분할
├── client.ts       ← Discord 클라이언트
├── components.ts   ← UI 컴포넌트 (버튼, 선택 메뉴)
└── components-registry.ts
```

Discord는 **@buape/carbon** 라이브러리 사용.
(Discord.js 대신 더 가벼운 라이브러리 선택)

---

## 채널 구현 예시: WhatsApp

```
src/whatsapp/  (또는 extensions/whatsapp/)
```

WhatsApp은 **@whiskeysockets/baileys** 사용.
(공식 API 없어서 WhatsApp Web 리버스 엔지니어링 기반)

QR 코드로 기기 페어링:
```bash
openclaw channels add whatsapp  # QR 코드 표시
# 폰으로 QR 스캔
```

---

## 채널 설정 구조 (config/types.channels.ts)

```typescript
// 각 채널의 설정 타입
type ChannelConfig = {
  enabled: boolean;
  accounts: ChannelAccountConfig[];  // 여러 계정 지원!
};

// 예: Telegram 설정
type TelegramConfig = {
  accounts: Array<{
    token: string;          // 봇 토큰
    allowFrom?: string[];   // 허용된 사용자 목록
    agentId?: string;       // 연결할 에이전트 ID
  }>;
};
```

**중요**: 하나의 채널에 **여러 계정** 연결 가능!
예: Telegram 봇 3개를 동시에 운영 가능.

---

## 메시지 수신 흐름

```
1. Telegram에서 사용자 메시지 도착
      ↓
2. grammy Bot이 메시지 수신
      ↓
3. bot-handlers.ts에서 처리
      ↓
4. channel-session.ts: 어느 세션으로 라우팅할지 결정
      ↓
5. Gateway의 handleIncomingMessage() 호출
      ↓
6. routing/resolve-route.ts: 어느 에이전트로 보낼지 결정
      ↓
7. agents/: AI 에이전트에게 전달
      ↓
8. AI 응답 받음
      ↓
9. ChannelMessagingAdapter.send()로 응답 전송
```

---

## 채널 보안: allow-from (수신 허용 목록)

모든 채널은 누가 메시지를 보낼 수 있는지 제어 가능.

```typescript
// channels/allow-from.ts
type AllowFromConfig = {
  // 특정 사용자 ID만 허용
  userIds?: string[];

  // 특정 채팅/채널만 허용
  chatIds?: string[];

  // 아무나 허용 (주의!)
  everyone?: boolean;
};
```

설정 예시:
```json5
{
  "telegram": {
    "accounts": [{
      "token": "...",
      "allowFrom": ["user:123456789", "group:-1001234567"]
    }]
  }
}
```

---

## 채널 세션 (channels/session.ts)

각 채널의 대화는 **세션** 단위로 관리된다.

```typescript
// 세션 키 생성 규칙
// 채널ID + 계정ID + 상대방ID → 고유한 세션 키
function buildSessionKey(channel: string, accountId: string, peerId: string): string {
  return `${channel}:${accountId}:${peerId}`;
}

// 예시:
// telegram:bot1:user123456  ← 봇1과 user123456의 대화
// discord:server1:channel789 ← 서버1의 channel789
```

---

## 스트리밍 응답 (Streaming)

AI 응답을 실시간으로 스트리밍 가능 (타이핑 효과).

```typescript
// ChannelStreamingAdapter
type ChannelStreamingAdapter = {
  supportsStreaming: boolean;
  streamMessage: (ctx: StreamContext, chunk: string) => Promise<void>;
};

// 지원 채널: Discord, Slack 등
// 미지원 채널: WhatsApp, Signal 등 (완성된 메시지만 전송)
```

---

## 다음: [06-routing.md](./06-routing.md)
