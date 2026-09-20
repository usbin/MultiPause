# IsEventing 멀티플레이 동기화 수정

- 상태: 진행중 (코드 수정 완료 / 사용자 인게임 검증 대기)
- 대상: MultiPause 1.1.3 / `MultiPause/ModEntry.cs`
- 관련 분석: [../analysis/eventonly-mode-behavior.md](../analysis/eventonly-mode-behavior.md)
- 작성일: 2026-09-20

## 문제

EVENTONLY 모드의 판정 근거인 `PlayerState.IsEventing`이 네트워크로 전파되지 않았다.

- `OnUpdateTicked`에서 **로컬 플레이어의** `IsEventing`만 갱신됨.
- `OnModMessageReceived`의 `STPMessage.Update` 처리가 `IsEventing`을 복사하지 않음.
- `STPMessage.AllPlayerStates` 처리도 복사하지 않음.

결과: 각 클라이언트의 `PlayerStates`에서 `IsEventing`이 true가 될 수 있는 건 자기 자신뿐.
시간은 호스트 기준으로 흐르므로, **팜핸드가 이벤트를 봐도 호스트 시계는 계속 흘렀다.**

추가로, 상태 브로드캐스트 조건이 `isPaused != IsPaused.Value` 뿐이라 이벤트 상태만 바뀌는
경우 메시지가 안 나가는 구멍이 있었다. (예: 이벤트 종료 순간 메뉴가 열려 있으면 `isPaused`가
true로 유지 → 메시지 미발송 → 원격 `IsEventing`이 true로 고착 → 시간 영구 정지)

## 수정 내용

| 위치 | 변경 |
|---|---|
| `OnUpdateTicked` 이벤트 시작/종료 블록 | `bool eventingChanged` 플래그 도입, 두 분기에서 true로 설정 |
| `OnUpdateTicked` 브로드캐스트 조건 | `if (isPaused != IsPaused.Value \|\| eventingChanged)` |
| `OnModMessageReceived` / `STPMessage.Update` | `state.IsEventing = messageState.IsEventing;` 추가 |
| `OnModMessageReceived` / `STPMessage.AllPlayerStates` | `item.Key != Game1.player.UniqueMultiplayerID`일 때만 `IsEventing` 복사 |

`AllPlayerStates`에서 자기 자신을 제외한 이유: 이 메시지는 호스트가 Query 응답으로 한 번
뿌리는 것이라 수신자 본인의 이벤트 상태가 낡은 값일 수 있다. 본인 값은 매 틱 로컬에서
갱신되므로 덮어쓰면 안 된다.

## 검증

자동 테스트 없음(사용자 선택: 인게임 수동 검증). 빌드 확인도 못 함 —
`packages/`(Lib.Harmony, Pathoschild.Stardew.ModBuildConfig) 미복원 + msbuild 미설치.

### 수동 검증 절차
1. 호스트 + 팜핸드 2인 접속, config는 `EVENTONLY`.
2. **팜핸드만** 이벤트(컷신) 진입 → 호스트 화면의 시계가 멈추는지 확인.
3. 이벤트 종료 → 호스트 시계가 다시 흐르는지 확인.
4. SMAPI 콘솔에 `Time is now frozen.` / `passing normally.` 로그와 각 플레이어의
   `eventing: True/False`가 올바르게 찍히는지 확인 (`ModEntry.cs:190-197`).
5. 회귀 확인: 이벤트 종료 직후 인벤토리를 열어둔 채 대기 → 시간이 정상적으로 흐르는지
   (고착 버그 재발 여부).

## 남은 이슈 (미수정)

- `freeze` 콘솔 명령이 무동작 — `IsSetFreezeByCommand`를 아무도 읽지 않음.
- EVENTONLY 분기의 미사용 변수 `int min = Int32.MaxValue;`.
- `LastEvent` 필드만 `PerScreen`이 아님 — 분할화면 사용 시 이벤트 감지가 꼬일 수 있음.
