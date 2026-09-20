# claudedocs 인덱스

## 진행중

| 파일 | 내용 |
|---|---|
| [tasks/fix-iseventing-sync.md](tasks/fix-iseventing-sync.md) | `PlayerState.IsEventing`이 멀티플레이로 동기화되지 않던 버그 수정. 코드 수정 완료, 사용자 인게임 검증 대기. 수동 검증 절차 포함. |
| [tasks/fix-host-config-and-perscreen.md](tasks/fix-host-config-and-perscreen.md) | ① `GetTimePassState()`가 호스트 동기화 값 대신 로컬 config를 읽던 문제 ② `LastEvent`가 화면 간 공유되어 분할화면에서 시간이 영구 정지하던 문제. 코드 수정 완료, 인게임 검증 대기. |

## 완료됨

| 파일 | 내용 |
|---|---|
| [analysis/eventonly-mode-behavior.md](analysis/eventonly-mode-behavior.md) | MultiPause 1.1.3 EVENTONLY 모드가 "전원 일시정지 시 시간 정지" 기능을 포함하는지 검증한 분석. 결론: 미포함. Freeze/Pause 차이와 부수 이슈 기록. |

## 예정됨

(없음)

## 보류

(없음)
