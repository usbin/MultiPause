<!-- 최종 수정: 2026-09-20 -->
# 개발 노트

## 진행 중

### 수정 3건 — 인게임 검증 대기
자동 테스트 없이 수정했으므로 호스트 + 팜핸드 2인 수동 검증이 필요하다.
절차는 `claudedocs/tasks/fix-iseventing-sync.md`,
`claudedocs/tasks/fix-host-config-and-perscreen.md` 참고.

1. **IsEventing 동기화** — 네트워크로 전파되지 않아 EVENTONLY에서 팜핸드가 이벤트를
   봐도 호스트 시계가 멈추지 않던 문제.
2. **호스트 config 미적용** — `GetTimePassState()`가 동기화된 `PauseMode.Value` 대신
   로컬 `Config`를 읽던 문제.
3. **`LastEvent` 화면 공유** — 분할화면에서 이벤트 감지가 꼬여 시간이 영구 정지하던 문제.

## 알려진 이슈

### `freeze` 콘솔 명령이 무동작
`SetFreezeByCommand`가 `PlayerState.IsSetFreezeByCommand`를 세팅하지만
`GetTimePassState()`를 비롯해 어디에서도 이 필드를 읽지 않는다. 또한
`bool.Parse(args[0])`는 인자 누락/오타 시 그대로 예외를 던진다.

### `InitialPauseState`가 매번 새 인스턴스
```csharp
public PerScreen<bool> InitialPauseState => new PerScreen<bool>(() => PauseMode.Value != PAUSE_ANY);
```
프로퍼티라 접근할 때마다 새 `PerScreen`을 만든다. `OnSaveLoaded`의
`IsPaused = InitialPauseState;`는 `IsPaused` 필드 자체를 통째로 교체해 버린다.

### config 값 검증 없음
`PauseMode_ANY_ALL_AUTO_EVENTONLY`에 오타가 있으면 `GetTimePassState()`의 어떤 분기에도
걸리지 않아 `freeze`가 계속 false → 시간이 절대 멈추지 않는다. 경고 로그도 없다.

### EVENTONLY 분기의 미사용 변수
`int min = Int32.MaxValue;` — AUTO 분기에서 복사해 온 잔재.

### config 런타임 재로드가 동작하지 않음
`ReloadConfig()`가 정의되어 있지만 **호출하는 곳이 없다.** `Config.Value`는 `Entry()`에서
한 번만 읽히므로, 게임 실행 중 `config.json`을 고쳐도 반영되지 않는다.
`OnUpdateTicked`의 `PauseMode.Value != Config.Value...` 비교(`:169`)는 이 재로드를
전제로 만들어진 것으로 보이나 현재는 사실상 무의미하다.

## 설계 결정

### Freeze와 Pause를 나눈 이유
`Freeze`는 `Game1.gameTimeInterval`만 고정해 시계를 세우고 `shouldTimePass()`는 **true로
강제**한다. 일부만 정지한 상황에서는 나머지 플레이어가 실제로 플레이 중이므로, 그들의
게임까지 "시간 정지" 취급하면 버프·낚시 미니게임 등 시간 종속 로직이 깨지기 때문이다.
전원이 정지한 `Pause`에서는 아무도 플레이 중이 아니므로 `shouldTimePass()`를 false로
만들어 바닐라식으로 완전히 세운다.

부작용: `Freeze` 중에는 메뉴를 열어둔 플레이어의 버프 지속시간이 계속 닳는다
(바닐라라면 false였을 판정을 모드가 true로 덮어쓰므로).

### EVENTONLY는 `IsPaused`를 보지 않는다
1.1.3에서 추가된 EVENTONLY는 오직 `IsEventing`만 본다. 다른 모드의 "정지 행동" 판정과
합쳐지지 않았으며, 이는 의도된 동작이다 — 인벤토리·건물 출입 등으로는 시간이 멈추지
않고 NPC 컷신에서만 멈춘다. 분석 근거는
`claudedocs/analysis/eventonly-mode-behavior.md` 참고.

### 중앙 권위자 없는 분산 판정
모든 플레이어가 각자의 `PlayerStates` 사본으로 동일한 `GetTimePassState()`를 돌린다.
구현이 단순한 대신, **상태 브로드캐스트가 한 번이라도 누락되면 플레이어마다 다른 결론을
내고 그 상태가 고착된다.** 새 상태 필드를 추가할 때는 반드시
(1) 송신 트리거, (2) `PlayerStateChanged` 수신 처리, (3) `AllPauseTimes` 수신 처리
세 곳을 모두 건드려야 한다. IsEventing 버그가 정확히 이 셋 중 둘을 빠뜨려 생긴 문제였다.

### 판정은 `PauseMode.Value`로만
`GetTimePassState()`는 반드시 호스트로부터 동기화된 `PauseMode.Value`를 읽어야 한다.
로컬 `Config.Value`를 직접 읽으면 플레이어마다 다른 모드로 판정해 위의 "전원 동일 결론"
전제가 깨진다. `Config.Value`는 **자기 설정을 호스트에게 알리는 용도**(`ConfigPauseMode`
필드에 실어 브로드캐스트)로만 쓴다.

`PauseMode`가 `static`인 이유도 이것이다 — `GetTimePassState()`가 Harmony 패치에서
호출되는 정적 메서드라 인스턴스 멤버에 접근할 수 없다.
