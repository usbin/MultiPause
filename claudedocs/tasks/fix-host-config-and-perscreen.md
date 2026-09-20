# 호스트 config 미적용 · LastEvent 화면 공유 수정

- 상태: 진행중 (코드 수정 완료 / 사용자 인게임 검증 대기)
- 대상: MultiPause 1.1.3 / `MultiPause/ModEntry.cs`
- 관련: [fix-iseventing-sync.md](fix-iseventing-sync.md)
- 작성일: 2026-09-20

---

## 이슈 1 — 호스트 config가 실제 판정에 반영되지 않음

### 문제
README와 코드 의도는 "호스트 설정만 사용"이고 이를 위해 `PauseMode`를 호스트 →
클라이언트로 동기화한다(`:170`, `:237`). 그런데 `GetTimePassState()`는 `PauseMode.Value`가
아니라 **로컬** `Config.Value.PauseMode_ANY_ALL_AUTO_EVENTONLY`를 직접 읽고 있었다.

결과: 각 플레이어가 자기 config로 따로 판정한다. 이 모드는 중앙 권위자 없이 "전원이
같은 결론을 낸다"는 전제로 설계됐기 때문에, 모드가 서로 다르면 버프 타이머·조작 가능
여부가 사람마다 달라진다.

### 수정
`GetTimePassState()`의 네 분기 비교 대상을 `PauseMode.Value`로 교체.

```csharp
- if (Config.Value.PauseMode_ANY_ALL_AUTO_EVENTONLY.ToUpper() == PAUSE_ALL)
+ if (PauseMode.Value == PAUSE_ALL)
```
(ANY / AUTO / EVENTONLY 분기도 동일)

`PauseMode`를 `static`으로 변경 — `GetTimePassState()`가 Harmony 패치에서 호출되는
정적 메서드라 인스턴스 멤버에 접근할 수 없다.

초기값 팩토리도 함께 변경했다.
```csharp
- new PerScreen<string>(() => string.Empty)
+ new PerScreen<string>(() => Config.Value.PauseMode_ANY_ALL_AUTO_EVENTONLY.ToUpper())
```
`Entry()`는 화면 0에서만 돌기 때문에, 분할화면의 두 번째 화면은 `PauseMode.Value`가
빈 문자열로 남아 어떤 분기에도 걸리지 않는다(= 시간이 절대 안 멈춤). 자기 `Config` 값을
기본값으로 삼아 이를 막는다. 람다는 지연 평가되므로 `Config`와의 정적 초기화 순서
문제는 없다.

`OnUpdateTicked`의 `PauseMode.Value != Config.Value...` 비교(`:169`)는 "자기 config를
호스트에게 알리는" 경로라 그대로 뒀다.

### 부수 효과
`GetTimePassState()`는 틱당 여러 번 호출되는데(`OnUpdateTicked` 1회 + `shouldTimePass()`
호출 횟수), 호출마다 최대 4번 돌던 `.ToUpper()`가 사라진다.

---

## 이슈 2 — `LastEvent`가 화면 간 공유되어 시간 영구 정지

### 문제
`private Event LastEvent;`만 `PerScreen`이 아니었다. 분할화면은 한 프로세스에서 화면
수만큼 `UpdateTicked`가 도는데 이 필드는 하나를 공유한다.

화면0만 컷신 중일 때:

| 틱 | `LastEvent` 읽기 | `CurrentEvent` | 결과 |
|---|---|---|---|
| 화면0 | `null` | `E` | 시작 분기 → P0 `IsEventing=true`, `LastEvent=E` |
| 화면1 | `E` | `null` | 종료 분기 → `LastEvent=null` |
| 화면0 | `null` | `E` | 시작 분기 재진입 → `LastEvent=E` |
| … | | | 매 틱 반복 |

값 자체는 각자 자기 `PlayerState`에 쓰므로 틀리지 않지만 **변화 감지가 매 틱 오작동**한다.
IsEventing 동기화 수정으로 브로드캐스트가 `eventingChanged`에 묶이면서, 이 오작동이
매 틱 네트워크 메시지 송신으로 증폭된다.

치명적인 건 컷신 종료 시점이다. 직전 화면1 틱이 `LastEvent`를 `null`로 만들어 두면
화면0 틱에서 `LastEvent == null && CurrentEvent == null`이 되어 **종료 분기가 실행되지
않고**, P0의 `IsEventing`이 true로 고착 → EVENTONLY에서 시간이 영구 정지한다.

### 수정
```csharp
- private Event LastEvent;
+ private PerScreen<Event> LastEvent = new PerScreen<Event>(() => null);
```
사용처 3곳(`:141`, `:149`, `:157`)을 `.Value`로.

---

## 검증

자동 테스트 없음(사용자 선택: 인게임 수동 검증). 빌드 확인 불가 —
`packages/` 미복원 + msbuild 미설치.

### 이슈 1 검증
1. 호스트 config `ALL`, 팜핸드 config `EVENTONLY`로 서로 다르게 설정하고 접속.
2. 팜핸드가 인벤토리를 열었을 때 **양쪽 다** 시간이 안 멈추는지 확인
   (`ALL`이므로 전원 정지가 아니면 안 멈춰야 함). 수정 전에는 팜핸드 쪽만
   EVENTONLY로 판정해 동작이 갈렸다.
3. 양쪽 모두 인벤토리를 열면 시간이 멈추는지 확인.
4. SMAPI 콘솔의 `Time is now ...` 로그가 호스트/팜핸드에서 같은 타이밍에 찍히는지 확인.

### 이슈 2 검증 (분할화면 필요)
1. 로컬 분할화면 2인 + config `EVENTONLY`.
2. 화면0만 컷신 진입 → 시간 정지 확인.
3. **컷신 종료 → 시간이 다시 흐르는지 확인** (수정 전에는 영구 정지).
4. 컷신 중 SMAPI 콘솔에 상태 로그가 매 틱 쏟아지지 않는지 확인.

분할화면을 쓰지 않는다면 이슈 2는 회귀만 확인하면 된다 — 일반 멀티에서 컷신 진입/종료가
정상 동작하는지.

## 남은 이슈 (미수정)

`wiki/devnotes.md`의 "알려진 이슈" 참고 — `freeze` 명령 무동작, `InitialPauseState`
프로퍼티, config 값 검증 없음, EVENTONLY 미사용 변수, config 런타임 재로드 미동작.
