<!-- 최종 수정: 2026-09-20 -->
# 빌드 · 설치 · 설정

## 사전 요구사항

- Stardew Valley 설치 (ModBuildConfig가 설치 경로를 자동 탐지해 게임 DLL을 참조)
- SMAPI 3.8.0 이상
- MSBuild / Visual Studio (.NET Framework 4.5.2 타깃)

## 빌드

```powershell
nuget restore MultiPause.sln     # packages/ 복원 (Lib.Harmony, ModBuildConfig)
msbuild MultiPause.sln /p:Configuration=Release
```

`packages/` 폴더가 없으면 `EnsureNuGetPackageBuildImports` 타깃이 빌드를 막는다.
ModBuildConfig가 빌드 산출물(`MultiPause.dll` + `manifest.json`)을 게임의
`Mods/MultiPause/`로 자동 복사한다.

게임 설치 경로 자동 탐지가 실패하면 `.csproj`에 `<GamePath>`를 지정한다.

## 설정 (config.json)

첫 실행 시 `Mods/MultiPause/config.json`이 생성된다.

```json
{
  "PauseMode_ANY_ALL_AUTO_EVENTONLY": "EVENTONLY"
}
```

| 값 | 동작 |
|---|---|
| `ANY` | 누구든 한 명이라도 정지 상태면 전체 정지 |
| `ALL` | 전원이 정지 상태일 때만 정지 |
| `AUTO` | 누적 정지시간이 가장 적은 플레이어 기준 |
| `EVENTONLY` | NPC 이벤트(컷신) 중일 때만 정지 (기본값, 1.1.3 추가) |

**호스트의 설정만 적용된다.** 다만 모든 플레이어가 모드를 설치해야 상태가 공유된다.
호스트의 모드 값은 접속 시 `PlayerStateChanged` 메시지로 클라이언트에 전파되어
`PauseMode`에 채택되고, 판정은 이 값으로만 이루어진다.

config는 게임 시작 시 한 번만 읽힌다. 값을 바꾸려면 게임을 재시작해야 한다.
(`ReloadConfig()`가 있지만 호출되지 않는다 — devnotes 참고)

## 콘솔 명령

| 명령 | 설명 |
|---|---|
| `freeze <true\|false>` | 현재 아무 동작도 하지 않음 (devnotes 참고) |

## 환경변수

없음.
