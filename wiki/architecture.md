<!-- 최종 수정: 2026-09-20 -->
# 아키텍처

## 디렉터리 구조

```
MultiPause-master/
  MultiPause.sln
  README.md              — 사용자용 설명 + 모드별 동작 + 버전 히스토리
  claudedocs/            — 분석·작업 문서 (소스 아님)
  wiki/                  — 이 문서
  MultiPause/
    MultiPause.csproj    — .NET Framework 4.5.2, ModBuildConfig
    manifest.json        — SMAPI 매니페스트 (버전/UniqueID/UpdateKeys)
    packages.config      — Lib.Harmony, ModBuildConfig
    ModEntry.cs          — 모드 전체 로직 (단일 파일)
    Config.cs            — config.json 매핑 클래스
    Properties/AssemblyInfo.cs
```

소스는 사실상 `ModEntry.cs` 하나다. 별도 레이어 분리 없이 SMAPI 이벤트 핸들러와
Harmony 패치, 상태 모델이 한 파일에 들어 있다.

## 모듈 관계

```
          config.json
              │ ReadConfig
              ▼
        Config (PauseMode_ANY_ALL_AUTO_EVENTONLY)
              │
              ▼
  ┌───────────────────────────────┐
  │           ModEntry            │
  │  SMAPI 이벤트 핸들러           │
  │   - GameLoop.UpdateTicked     │──► 로컬 상태 갱신 + 시계 억제
  │   - Multiplayer.*             │──► PlayerStates 동기화
  │                               │
  │  PlayerStates                 │
  │   Dictionary<long,PlayerState>│◄─┐
  └──────────┬────────────────────┘  │ 메시지(Query/Update/AllPlayerStates)
             │ GetTimePassState()    │
             ▼                       ▼
     ShouldTimePassPatch      다른 플레이어의 ModEntry
     (Harmony prefix/transpiler)
             │
             ▼
     StardewValley.Game1.shouldTimePass()
```

### 핵심 설계

1. **모든 플레이어가 같은 판정을 독립 계산한다.** 중앙 권위자가 없고, 각자 자기
   `PlayerStates` 사본으로 `GetTimePassState()`를 돌린다. 그래서 상태 브로드캐스트가
   누락되면 플레이어마다 다른 결론을 내게 된다.
2. **Harmony 이중 패치.** Prefix는 멀티플레이일 때 `shouldTimePass()`를 통째로 대체하고,
   Transpiler는 원본 IL의 `Game1.IsMultiplayer` 호출을
   `ModEntry.GetIsMultiplayerForShouldTimePass()`로 바꿔치기한다. 후자는 "멀티인데
   싱글인 척" 원본 로직을 돌려 내 정지 상태를 알아내기 위한 장치다
   (`ForceSinglePlayerCheck` 플래그).
3. **PerScreen.** 분할화면 대응을 위해 모든 가변 상태를 `PerScreen<T>`로 감쌌다.
   화면 간에 공유되는 일반 필드가 하나라도 있으면 화면별 판정이 서로를 덮어쓴다.
