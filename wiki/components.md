<!-- 최종 수정: 2026-09-20 -->
# 구성 요소

## 파일

| 파일 | 역할 |
|---|---|
| `MultiPause/ModEntry.cs` | 모드 전체 로직. SMAPI 이벤트 핸들러, 상태 모델, Harmony 패치 |
| `MultiPause/Config.cs` | `config.json` 매핑. 필드 1개 |
| `MultiPause/manifest.json` | SMAPI 매니페스트 |

## Config

```csharp
public string PauseMode_ANY_ALL_AUTO_EVENTONLY { get; set; } = "EVENTONLY";
```
유효값: `ANY` / `ALL` / `AUTO` / `EVENTONLY`. 대소문자 무시(`ToUpper()` 후 비교).
오타 시 어떤 분기에도 걸리지 않아 **시간이 절대 멈추지 않는다**(검증 없음).

## ModEntry 주요 멤버

### 상태

| 멤버 | 설명 |
|---|---|
| `PlayerStates` | `Dictionary<long, PlayerState>` — 모든 플레이어의 정지/이벤트 상태 |
| `IsPaused` | 로컬 플레이어의 직전 정지 상태(변화 감지용) |
| `PauseMode` | **판정에 실제로 쓰이는 모드.** 호스트 값이 전파됨. `GetTimePassState()`가 정적 메서드라 `static` |
| `ForceSinglePlayerCheck` | true면 `shouldTimePass()`가 싱글플레이처럼 동작 |
| `LastEvent` | 직전 틱의 `Game1.CurrentEvent`(`PerScreen`) — 이벤트 시작/종료 감지용 |
| `prevGameTimeInterval` | 시계를 되돌릴 때 쓰는 직전 `Game1.gameTimeInterval` |

### 메서드

| 메서드 | 역할 |
|---|---|
| `Entry` | config 로드, `freeze` 콘솔 명령 등록, 이벤트 구독 |
| `applyPatch` | `Game1.shouldTimePass`에 Harmony prefix + transpiler 적용 |
| `OnUpdateTicked` | 매 틱: 누적 정지시간 증가 → 이벤트 진입/종료 감지 → 내 정지 상태 계산 → 변화 시 브로드캐스트 → 시계 억제 |
| `OnModMessageReceived` | `Query` / `Update` / `AllPlayerStates` 메시지 처리 |
| `OnPeerContextReceived` / `OnPeerDisconnected` | 접속·이탈 시 `IsOnline` 갱신 |
| `ShouldTimePassForCurrentPlayer` | 바닐라 판정 로직에서 멀티 체크만 뺀 복제본 |
| `GetTimePassState` | **핵심.** 모드별로 `Pass` / `Freeze` / `Pause` 결정 |
| `GetMinTimePaused` | AUTO 모드용 최소 누적 정지시간 |
| `SetFreezeByCommand` | `freeze <true\|false>` 콘솔 명령 (현재 무동작 — devnotes 참고) |

### TimePassState

| 값 | 게임 내 시계 | `Game1.shouldTimePass()` | 의미 |
|---|---|---|---|
| `Pass` | 흐름 | 바닐라 판정 | 평상시 |
| `Freeze` | 멈춤 | **강제 `true`** | 일부만 정지. 시계만 세우고 나머지는 정상 진행 |
| `Pause` | 멈춤 | `false` | 전원 정지. 버프 타이머 등까지 바닐라식으로 정지 |

`Freeze`에서 `shouldTimePass()`를 true로 강제하는 이유: 지금 실제로 플레이 중인
플레이어가 있으므로 그들의 게임까지 "시간 정지" 취급하면 안 되기 때문.

## PauseMode별 판정 (`GetTimePassState`)

| 모드 | freeze 조건 | 비고 |
|---|---|---|
| `ANY` | 한 명이라도 `IsPaused` | 악용 쉬움 |
| `ALL` | 전원 `IsPaused` | |
| `AUTO` | 누적 정지시간이 가장 적은 플레이어가 `IsPaused` | 시간 손실 자동 보정 |
| `EVENTONLY` | 한 명이라도 `IsEventing` | **`IsPaused`를 전혀 보지 않음.** 전원이 인벤토리를 열어도 시간은 흐른다 |

최종 반환: `freeze ? (allPaused ? Pause : Freeze) : Pass`.
`EVENTONLY` 분기에서는 `allPaused`가 "전원 **이벤트 중**"을 뜻하도록 의미가 바뀐다.

## PlayerState

| 필드 | 설명 | 동기화 |
|---|---|---|
| `IsPaused` | 싱글 기준 정지 상태 | O |
| `Ticks` | 메시지 순서 판별용 | O |
| `TotalTimePaused` | 오늘 누적 정지 틱 (AUTO용, 매일 0으로 초기화) | 호스트 → 전체 |
| `IsOnline` / `IsHost` | 접속·호스트 여부 | O |
| `ConfigPauseMode` | 해당 플레이어의 config 모드 (호스트 것만 채택) | O |
| `IsEventing` | 컷신 진행 중 여부 (EVENTONLY용) | O |
| `IsSetFreezeByCommand` | `freeze` 명령 값. 아무도 읽지 않음 | X |

## Harmony 패치 (`ShouldTimePassPatch`)

- `Prefix`: 멀티플레이이고 `ForceSinglePlayerCheck`가 아니면
  `__result = GetTimePassState() != TimePassState.Pause`로 설정하고 원본을 건너뛴다.
- `Transpiler`: 원본 IL의 `Game1.get_IsMultiplayer` 호출을
  `ModEntry.GetIsMultiplayerForShouldTimePass()`로 치환. `ForceSinglePlayerCheck`가
  true일 때 false를 반환해 원본이 싱글플레이 경로를 타게 만든다.
