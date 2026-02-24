# OpenClaw 코드 분석 - 03. CLI 진입점 흐름

---

## CLI 실행 흐름 전체 지도

```
사용자가 터미널에서 실행
    openclaw gateway run
         ↓
    openclaw.mjs          ← bin 파일 (npm 설치 시 PATH에 등록)
         ↓
    src/entry.ts          ← 진입점 (환경 세팅, respawn)
         ↓
    src/cli/run-main.ts   ← runCli() 호출
         ↓
    src/cli/route.ts      ← 서브커맨드 라우팅
         ↓
    src/cli/program/      ← Commander 프로그램 빌드
         ↓
    src/commands/         ← 실제 커맨드 핸들러 실행
```

---

## Step 1: openclaw.mjs (bin 파일)

`package.json`에서:
```json
{
  "bin": {
    "openclaw": "openclaw.mjs"
  }
}
```

npm으로 설치하면 `openclaw` 명령이 `openclaw.mjs`를 가리킴.

`openclaw.mjs`는 단순한 래퍼:
```js
#!/usr/bin/env node
// dist/entry.js를 import해서 실행
import './dist/entry.js'
```

---

## Step 2: src/entry.ts (진입점)

**역할**: 환경 초기화 + respawn (재시작) 처리

```typescript
// entry.ts의 핵심 로직

// 1. 프로세스 제목 설정
process.title = "openclaw";

// 2. 환경변수 정규화
normalizeEnv();

// 3. 실험적 경고 억제를 위한 respawn
// Node.js에서 "ExperimentalWarning" 메시지가 뜨는데,
// 이를 숨기려면 --disable-warning 플래그가 필요하다.
// 근데 NODE_OPTIONS에는 이 플래그를 넣을 수 없어서
// 자식 프로세스를 새로 spawn하는 방식으로 해결!
if (!ensureExperimentalWarningSuppressed()) {
  // respawn이 필요 없으면 바로 CLI 실행
  import('./cli/run-main.js')
    .then(({ runCli }) => runCli(process.argv))
}
```

### Respawn 패턴이란?
```
process A (부모)
  ├─ Node.js 실험적 경고가 억제 안 된 상태
  └─ spawn → process B (자식)
              ├─ --disable-warning 플래그 포함
              └─ 실제 CLI 실행
```

부모 프로세스는 자식을 시작하고 자신은 종료.
자식 프로세스가 실제 작업을 수행.

---

## Step 3: src/cli/run-main.ts

**역할**: CLI 실행의 실제 시작점

```typescript
export async function runCli(argv: string[]): Promise<void> {
  // 1. 환경 검증 (Node.js 버전 체크 등)
  assertSupportedRuntime();

  // 2. .env 파일 로드
  loadDotEnv();

  // 3. 환경변수 정규화
  normalizeEnv();

  // 4. CLI PATH 설정 (openclaw 바이너리가 PATH에 있도록)
  ensureOpenClawCliOnPath();

  // 5. 서브커맨드 라우팅
  await tryRouteCli(argv);
}
```

---

## Step 4: src/cli/route.ts

**역할**: 어떤 서브커맨드인지 파악해서 적절한 처리기로 분기

```typescript
export async function tryRouteCli(argv: string[]): Promise<void> {
  // argv에서 primary 커맨드 추출
  // 예: ["openclaw", "gateway", "run"] → primary = "gateway"
  const primary = getPrimaryCommand(argv);

  // Commander 프로그램 빌드 (전체 커맨드 트리 등록)
  const { program } = await buildProgram({ argv, primary });

  // 실행!
  await program.parseAsync(argv);
}
```

---

## Step 5: src/cli/program/ (Commander 프로그램 빌더)

### build-program.ts

Commander.js 프로그램을 구성하는 곳.

```typescript
export async function buildProgram(opts: BuildProgramOptions) {
  const program = new Command('openclaw');

  // 각 커맨드 그룹 등록
  await registerAgentCommands(program);      // agent 관련
  await registerConfigureCommands(program);  // configure 관련
  await registerMessageCommands(program);    // message 관련
  await registerOnboardCommands(program);    // onboard 관련
  await registerStatusHealthSessions(program); // status/health/sessions
  await registerSubClis(program);            // gateway, nodes 등
  await registerSetupCommands(program);      // setup
  await registerMaintenanceCommands(program); // maintenance

  return { program };
}
```

### register.*.ts 파일들

각 커맨드 그룹을 등록하는 파일들:

```
program/
├── register.agent.ts          ← openclaw agent [...]
├── register.configure.ts      ← openclaw configure [...]
├── register.message.ts        ← openclaw message [...]
├── register.onboard.ts        ← openclaw onboard [...]
├── register.status-health-sessions.ts ← openclaw status/health/sessions
├── register.setup.ts          ← openclaw setup [...]
├── register.subclis.ts        ← openclaw gateway/nodes/...
└── register.maintenance.ts    ← openclaw update/doctor/...
```

---

## 주요 CLI 커맨드 구조

### 최상위 커맨드 트리

```
openclaw
├── gateway         ← 게이트웨이 서버 관련
│   ├── run         ← 게이트웨이 시작
│   ├── stop        ← 게이트웨이 정지
│   └── status      ← 게이트웨이 상태
│
├── agent           ← AI 에이전트 관련
│   ├── run         ← 에이전트 실행
│   ├── list        ← 에이전트 목록
│   └── add/delete
│
├── channels        ← 메시징 채널 관련
│   ├── add         ← 채널 추가
│   ├── status      ← 채널 상태
│   └── remove
│
├── message         ← 메시지 전송
│   └── send
│
├── models          ← AI 모델 관리
│   ├── list        ← 사용 가능한 모델 목록
│   └── set         ← 기본 모델 설정
│
├── status          ← 전체 시스템 상태
├── health          ← 헬스 체크
├── sessions        ← 세션 관리
│
├── configure       ← 설정 마법사
├── onboard         ← 초기 설정
├── setup           ← 설치 설정
│
├── config          ← 설정 파일 직접 조작
│   ├── get
│   ├── set
│   └── unset
│
├── doctor          ← 문제 진단 및 자동 수정
├── update          ← 버전 업데이트
└── ...
```

---

## src/cli/deps.ts - 의존성 주입

OpenClaw는 **의존성 주입(DI)** 패턴을 사용한다.
테스트 시 가짜 의존성을 주입할 수 있도록.

```typescript
// createDefaultDeps: 실제 프로덕션 의존성 생성
export function createDefaultDeps(): CliDeps {
  return {
    config: loadConfig,
    writeConfig: writeConfigFile,
    log: createSubsystemLogger('cli'),
    // ...
  };
}

// 커맨드에서 사용 방법:
async function runGateway(opts: GatewayOpts, deps = createDefaultDeps()) {
  const config = await deps.config();
  // ...
}

// 테스트에서:
test('gateway starts', () => {
  const mockDeps = { config: () => mockConfig };
  await runGateway(opts, mockDeps);  // 가짜 의존성 주입
});
```

---

## src/cli/progress.ts - 프로그레스 UI

```typescript
// osc-progress + @clack/prompts 사용
// 터미널에서 예쁜 프로그레스/스피너 표시
import { spinner } from '@clack/prompts';

const s = spinner();
s.start('연결 중...');
await doSomething();
s.stop('완료!');
```

---

## 핵심 패턴 요약

| 패턴 | 설명 |
|------|------|
| Respawn | 부모가 자식 프로세스를 spawn해서 플래그 전달 |
| Lazy import | `import()` 동적 임포트로 필요할 때만 모듈 로드 |
| 의존성 주입 | `createDefaultDeps()`로 테스트 가능한 구조 |
| Commander | CLI 커맨드 트리를 선언적으로 정의 |
| 레지스트리 패턴 | 커맨드를 `register.*.ts`로 분리해서 등록 |

---

## 다음: [04-gateway.md](./04-gateway.md)
