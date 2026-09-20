# EVENTONLY 모드 동작 분석

- 상태: 완료됨 (분석만, 코드 변경 없음)
- 대상: MultiPause 1.1.3 / `MultiPause/ModEntry.cs`
- 작성일: 2026-09-20

## 질문

EVENTONLY 모드가 "1명이라도 이벤트 중이면 정지"에 더해서 "모든 플레이어가 시간 정지 행동(인벤토리 열기 등)을 하고 있으면 정지"까지 포함하는가?

## 결론

**아니다. 포함되어 있지 않다.** EVENTONLY는 오직 `IsEventing`만 본다.

## 근거

`ModEntry.GetTimePassState()` (ModEntry.cs:382-402)

```csharp
else if (... == PAUSE_EVENTONLY)
{
    int min = Int32.MaxValue;            // 사용되지 않는 변수
    foreach (Farmer farmer in Game1.getOnlineFarmers())
    {
        PlayerStates.Value.TryGetValue(farmer.UniqueMultiplayerID, out PlayerState state);
        if (state != null && state.IsOnline)
        {
            if (state.IsEventing) freeze = true;
            else                  allPaused = false;
        }
    }
}
return freeze ? (allPaused ? TimePassState.Pause : TimePassState.Freeze) : TimePassState.Pass;
```

- 이 분기에서 `state.IsPaused`(메뉴/인벤토리 등으로 인한 정지 상태)를 **한 번도 참조하지 않는다.**
- 이벤트 중인 사람이 아무도 없으면 `freeze == false` → 전원이 인벤토리를 열고 있어도 `TimePassState.Pass` (시간 흐름).
- 따라서 ALL 모드의 "전원 정지 시 멈춤" 기능은 EVENTONLY에 합쳐져 있지 않다.

### allPaused 변수의 의미 변질

다른 모드에서 `allPaused`는 "모든 플레이어가 IsPaused"를 뜻하지만, EVENTONLY 분기에서는 "모든 플레이어가 **이벤트 중**"을 뜻하게 된다.
결과적으로 EVENTONLY의 반환값은:

| 상황 | 결과 |
|---|---|
| 아무도 이벤트 중이 아님 | `Pass` (인벤토리 전원 오픈이어도 시간 흐름) |
| 일부만 이벤트 중 | `Freeze` (`gameTimeInterval`만 되돌려 시계만 멈춤) |
| 전원 이벤트 중 | `Pause` (`shouldTimePass() == false`) |

`ShouldTimePassPatch.Prefix`는 `TimePassState.Pause`일 때만 `shouldTimePass()`를 false로 만들기 때문에 Freeze와 Pause는 체감이 다르다(Freeze는 시계만 정지).

## 부수적으로 발견한 이슈 (수정하지 않음)

1. **`IsEventing`이 네트워크로 동기화되지 않는다.**
   - `OnUpdateTicked`에서 로컬 플레이어의 `IsEventing`만 갱신된다 (ModEntry.cs:137-151).
   - `OnModMessageReceived`의 `STPMessage.Update` 처리는 `Ticks / IsPaused / IsHost / ConfigPauseMode`만 복사하고 `IsEventing`은 복사하지 않는다 (ModEntry.cs:231-239).
   - `STPMessage.AllPlayerStates` 처리도 마찬가지로 `IsEventing`을 복사하지 않는다 (ModEntry.cs:256-260).
   - 즉 각 클라이언트의 `PlayerStates`에서 `IsEventing`이 true가 될 수 있는 건 **자기 자신뿐**이다. 호스트 기준으로 판정되는 시간 흐름 특성상, 팜핸드가 이벤트를 봐도 호스트 시계는 계속 흐를 가능성이 높다.
2. **`freeze` 콘솔 명령이 무동작.** `SetFreezeByCommand`가 `PlayerState.IsSetFreezeByCommand`를 세팅하지만 `GetTimePassState()` 어디에서도 이 필드를 읽지 않는다 (ModEntry.cs:268-274, 429).
3. **미사용 변수** `int min = Int32.MaxValue;` (ModEntry.cs:384) — AUTO 분기에서 복사해 온 잔재.
