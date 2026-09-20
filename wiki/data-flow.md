<!-- 최종 수정: 2026-09-20 -->
# 데이터 흐름

## 매 틱 (`OnUpdateTicked`)

월드가 로드되어 있고 멀티플레이일 때만 동작한다.

```
1. 누적 정지시간 집계
   PlayerStates의 각 엔트리에서 IsPaused || !IsOnline 이면 TotalTimePaused++

2. 이벤트 진입/종료 감지
   LastEvent == null && Game1.CurrentEvent != null  → 내 IsEventing = true,  eventingChanged = true
   LastEvent != null && Game1.CurrentEvent == null  → 내 IsEventing = false, eventingChanged = true
   LastEvent = Game1.CurrentEvent

3. 내 정지 상태 계산
   ForceSinglePlayerCheck = true
   isPaused = !Game1.shouldTimePass()      // Transpiler 덕에 싱글플레이 경로로 평가됨
   ForceSinglePlayerCheck = false

4. 변화가 있으면 브로드캐스트
   if (isPaused != IsPaused || eventingChanged)
       내 PlayerState 갱신(IsPaused, Ticks) → SendMessage(state, "PlayerStateChanged")
       config 모드가 바뀌었으면 ConfigPauseMode도 함께 실어 보냄

5. 시계 억제
   state = GetTimePassState()
   state != Pass → Game1.gameTimeInterval 을 직전 값(또는 0)으로 되돌림
   state == Pass → prevGameTimeInterval 갱신

6. 최초 1회 Query 브로드캐스트
```

`gameTimeInterval`을 "되돌리는" 방식이라 시계는 멈춰도 게임 루프 자체는 계속 돈다.
`Pause` 상태에서만 Harmony Prefix가 `shouldTimePass()`를 false로 만들어 버프 타이머 등
바닐라 시간 종속 로직까지 멈춘다.

## shouldTimePass() 호출 경로

```
Game1.shouldTimePass()
   │
   ├─ Prefix
   │    멀티 && !ForceSinglePlayerCheck
   │      → __result = (GetTimePassState() != Pause), 원본 skip
   │    그 외 → 원본 실행
   │
   └─ 원본 (Transpiler 적용본)
        Game1.IsMultiplayer 자리에 GetIsMultiplayerForShouldTimePass()
          → ForceSinglePlayerCheck면 false를 돌려 싱글플레이 판정 경로로 유도
```

## 멀티플레이 메시지

| 타입 | 송신자 | 시점 | 내용 | 수신 처리 |
|---|---|---|---|---|
| `QueryStates` | 전원 | 월드 진입 직후 1회 | `true` | 자기 상태를 Update로 회신. 호스트는 추가로 AllPlayerStates 송신 |
| `PlayerStateChanged` | 전원 | 정지 상태 또는 이벤트 상태 변화 시 | 자기 `PlayerState` | `Ticks`가 더 최신일 때만 `IsPaused`/`IsHost`/`ConfigPauseMode`/`IsEventing` 반영. 호스트 발신이면 `PauseMode`도 채택 |
| `AllPauseTimes` | 호스트 | Query 응답 | `Dictionary<long, PlayerState>` 전체 | `TotalTimePaused`/`IsOnline`/`IsHost`/`ConfigPauseMode` 반영. `IsEventing`은 **자기 자신 제외** 후 반영 |

`AllPauseTimes`에서 자기 자신의 `IsEventing`을 제외하는 이유: 이 메시지는 조인 시점에
한 번 뿌려지므로 호스트가 가진 수신자 본인의 이벤트 상태가 낡은 값일 수 있다. 본인 값은
매 틱 로컬에서 갱신되므로 덮어쓰면 안 된다.

## 접속 · 이탈 · 날짜 전환

| 이벤트 | 처리 |
|---|---|
| `SaveLoaded` | `PlayerStates` 초기화, 자기 엔트리 생성(`IsHost`, `ConfigPauseMode`) |
| `PeerContextReceived` | 해당 플레이어 `IsOnline = true`, `IsPaused = InitialPauseState` |
| `PeerDisconnected` | `IsOnline = false`, `IsPaused = true` |
| `DayStarted` | 전원 `TotalTimePaused = 0` |
| `ReturnedToTitle` | Query 재송신 플래그 리셋 |

신규 플레이어의 `TotalTimePaused`는 `GetMinTimePaused()`로 초기화해 AUTO 모드에서
중간 합류자가 불이익/이득을 받지 않도록 한다.
