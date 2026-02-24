# OpenClaw 코드 분석 - 06. 메시지 라우팅

---

## 라우팅이란?

**어떤 채널에서 온 메시지를 어떤 에이전트로 보낼지 결정하는 시스템.**

```
Telegram user:123  ──→  어느 에이전트?  ──→  agent-A (기본 에이전트)
Discord channel:456 ──→  어느 에이전트?  ──→  agent-B (코딩 전용)
Slack #general     ──→  어느 에이전트?  ──→  agent-C (회사 전용)
```

---

## 핵심 파일

```
src/routing/
├── resolve-route.ts    ← 라우팅 결정 (핵심!)
├── session-key.ts      ← 세션 키 생성
├── bindings.ts         ← 채널-에이전트 바인딩 규칙
└── account-id.ts       ← 계정 ID 정규화
```

---

## 라우팅 결정 (resolve-route.ts)

```typescript
// 입력: 어디서 온 메시지인가?
type ResolveAgentRouteInput = {
  cfg: OpenClawConfig;      // 전체 설정
  channel: string;          // 채널 ID (예: "telegram")
  accountId?: string | null; // 봇 계정 ID
  peer?: RoutePeer | null;  // 메시지 발신자 (상대방)
  parentPeer?: RoutePeer | null; // 부모 스레드 발신자
  guildId?: string | null;  // Discord 서버 ID
  teamId?: string | null;   // Slack 팀 ID
  memberRoleIds?: string[];  // Discord 역할 ID 목록
};

// 출력: 어느 에이전트로 보낼까?
type ResolvedAgentRoute = {
  agentId: string;          // 에이전트 ID
  channel: string;          // 채널 ID
  accountId: string;        // 계정 ID
  sessionKey: string;       // 고유 세션 키
  mainSessionKey: string;   // 메인 세션 키
  matchedBy:                // 어떤 규칙으로 매칭됐나 (디버깅용)
    | "binding.peer"        // 특정 사용자/채팅 바인딩
    | "binding.peer.parent" // 부모 스레드 바인딩
    | "binding.guild+roles" // Discord 역할 기반
    | "binding.guild"       // Discord 서버 기반
    | "binding.team"        // Slack 팀 기반
    | "binding.account"     // 봇 계정 기반
    | "binding.channel"     // 채널 기반
    | "default";            // 기본 에이전트
};
```

### 라우팅 우선순위

바인딩을 찾을 때 **더 구체적인 것이 우선**:

```
1. peer (특정 사용자/채팅) - 가장 구체적
2. peer.parent (부모 스레드)
3. guild + roles (Discord: 서버 + 역할)
4. guild (Discord: 서버)
5. team (Slack: 팀)
6. account (봇 계정)
7. channel (채널 전체)
8. default (기본값) - 가장 일반적
```

---

## 바인딩 (bindings.ts)

바인딩 = "이 채널/사용자는 이 에이전트를 사용해라" 규칙.

```typescript
// bindings.ts에서 바인딩 목록 조회
export function listBindings(config: OpenClawConfig): AgentBinding[] {
  return config.routing?.bindings ?? [];
}

// 바인딩 타입
type AgentBinding = {
  agentId: string;       // 연결할 에이전트

  // 아래 중 하나를 지정:
  channel?: string;      // 채널 전체 (예: "telegram")
  account?: string;      // 특정 봇 계정
  peer?: BindingPeer;    // 특정 사용자/채팅
  guild?: string;        // Discord 서버
  team?: string;         // Slack 팀
  roles?: string[];      // Discord 역할들
};
```

### 바인딩 설정 예시

```json5
// ~/.openclaw/config.json5
{
  "routing": {
    "bindings": [
      // Telegram의 특정 그룹은 코딩 에이전트 사용
      {
        "agentId": "coding-agent",
        "channel": "telegram",
        "peer": { "kind": "group", "id": "-1001234567890" }
      },

      // Discord #general은 지원 에이전트 사용
      {
        "agentId": "support-agent",
        "channel": "discord",
        "peer": { "kind": "channel", "id": "123456789" }
      },

      // Discord "admin" 역할은 관리자 에이전트
      {
        "agentId": "admin-agent",
        "channel": "discord",
        "guild": "server123",
        "roles": ["role-id-for-admin"]
      }
    ]
  }
}
```

---

## 세션 키 (session-key.ts)

세션 키 = 하나의 대화를 식별하는 고유 문자열.

```typescript
// 세션 키 생성 규칙
export function buildAgentMainSessionKey(params: {
  agentId: string;
  channel: string;
  accountId: string;
  peer: RoutePeer;
}): string {
  // 예: "telegram:bot1:default:user:123456"
  return `${params.channel}:${params.accountId}:${params.agentId}:${params.peer.kind}:${params.peer.id}`;
}
```

### 세션 키의 역할

1. **대화 연속성**: 같은 키 → 같은 대화 이어짐
2. **동시성 제어**: 같은 세션에서 동시에 두 요청이 오면 큐에 대기
3. **세션 스토리지**: 파일 시스템에 대화 기록 저장

---

## 계정 ID (account-id.ts)

하나의 채널에 여러 봇 계정이 있을 수 있음.
계정 ID는 어떤 봇 계정인지 식별.

```typescript
// 정규화된 계정 ID 형식
// "telegram:bot_token_prefix" 같은 형식

export function normalizeAccountId(raw: string | null | undefined): string {
  return raw?.trim() || DEFAULT_ACCOUNT_ID;
}

export const DEFAULT_ACCOUNT_ID = "default";
```

---

## 전체 라우팅 흐름 예시

### 시나리오: Telegram 사용자가 메시지 보냄

```
1. Telegram Bot (grammy)이 메시지 수신
   {
     from: { id: 123456, username: "peter" },
     chat: { id: 123456, type: "private" },
     text: "안녕하세요"
   }

2. bot-handlers.ts에서 resolveAgentRoute 호출
   {
     channel: "telegram",
     accountId: "bot1",
     peer: { kind: "dm", id: "123456" }
   }

3. resolve-route.ts에서 바인딩 탐색
   - peer "123456" 바인딩? → 없음
   - account "bot1" 바인딩? → 없음
   - channel "telegram" 바인딩? → 없음
   - default → agentId: "default"

4. 결과:
   {
     agentId: "default",
     sessionKey: "telegram:bot1:default:dm:123456",
     matchedBy: "default"
   }

5. 해당 세션으로 메시지 전달 → AI 에이전트 처리
```

---

## ChatType (채팅 종류)

```typescript
// channels/chat-type.ts
type ChatType =
  | "dm"        // 1:1 다이렉트 메시지
  | "group"     // 그룹 채팅
  | "channel"   // 채널 (Discord, Telegram)
  | "thread"    // 스레드 (Slack, Discord)
  | "voice";    // 음성 채널
```

---

## 다음: [07-agents.md](./07-agents.md)
