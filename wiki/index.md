<!-- 최종 수정: 2026-09-20 -->
# MultiPause 위키

스타듀밸리 멀티플레이에서 "싱글플레이였다면 시간이 멈췄을 상황"(인벤토리 열기, NPC 대화,
상점 이용, 수확 애니메이션, 맵 이동 등)에 시간을 멈춰주는 SMAPI 모드. 호스트의 config
설정만 적용되며, 모든 플레이어가 모드를 설치해야 한다.

## 기술 스택

| 항목 | 값 |
|---|---|
| 런타임 | .NET Framework 4.5.2 |
| 모드 로더 | SMAPI 3.8.0 이상 |
| 패치 | Lib.Harmony 1.2.0.1 (`Game1.shouldTimePass()` prefix + transpiler) |
| 빌드 | Pathoschild.Stardew.ModBuildConfig 3.3.0 |
| 버전 | 1.1.3 (UniqueID `usbin.multipause`) |

## 목차

- [architecture.md](architecture.md) — 디렉터리 구조와 모듈 관계
- [components.md](components.md) — 파일·클래스·주요 메서드 역할
- [setup.md](setup.md) — 빌드·설치·설정
- [data-flow.md](data-flow.md) — 틱 처리와 상태 동기화 흐름
- [devnotes.md](devnotes.md) — 알려진 이슈와 설계 결정
