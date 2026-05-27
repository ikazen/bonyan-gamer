# 아키텍처

## 머신 배치

```
+---------------+         +-------------------+         +------------------+
| Windows       |  HTTP   | Mac M1 Pro 32GB   |  HTTPS  | Ollama Cloud     |
| (executor)    | <-----> | (controller)      | <-----> | (brain - L)      |
|               |         |                   |         |                  |
| - PyAutoGUI   |         | - FastAPI 서버    |         | - 복잡 추론만    |
| - 스크린샷    |         | - 게임 루프 구동  |         | - 좌표 미관여    |
|               |         | - 로컬 비전·OCR   |         |                  |
|               |         | - UI 라벨링       |         |                  |
|               |         | - 좌표 그라운딩   |         |                  |
|               |         | - 메모리 관리     |         |                  |
+---------------+         +-------------------+         +------------------+
   pure actuator              eyes + spine                  reasoning (L)
   "hands"                   (V + orchestration)            "brain"
```

VLA의 V/L/A를 머신 단위로 칼같이 나누지 않는다. 이유는 ADR-001 참조.

## 한 사이클 (일반 턴)

동기 단방향. Windows가 Mac을 호출, Mac이 응답.

1. Windows: 직전 액션 종료 후 *내부 2초 대기* → 스크린샷 캡처 (ADR-006)
2. Windows → Mac: 스크린샷 + 세션 ID HTTP POST
3. Mac: 로컬 비전 — UI 요소 파싱, OCR, 라벨 ↔ 좌표 사전 구축
4. Mac: 메모리 파일 로드
   - `games/_shared/meta_notes.md`
   - `games/{game}/prompt.md`
   - `games/{game}/strategy_note.md`
   - `games/{game}/sessions/{current}/state.{ext}`
5. Mac → Ollama Cloud: 시각 라벨 + 메모리 파일 + (필요시 스크린샷) → 추론 요청
6. Cloud: 도구 호출 반복 (D 모델, ADR-003). `end_turn` 부를 때까지. 턴당 최대 8회.
7. Mac: 추상 액션을 픽셀 좌표로 그라운딩 → Windows에 응답
8. Windows: PyAutoGUI 실행 → 2초 대기 후 1번으로

## 세션 lifecycle

`init() → step() × N → finalize()`. 세션 종료 트리거:

- AI가 `end_session()` 도구 호출
- 사람이 명시적으로 finalize 호출
- 시스템이 게임오버 감지 (`prompt.md`에 정의된 조건)

`finalize()` 시점에 **별도 reflection 턴**이 강제로 실행됨 (ADR-002 안전망 3).

## 메모리 계층

매 턴 LLM은 파일에서 새로 읽는다 (롤링 히스토리 없음, ADR-004).

| 계층 | 위치 | 작성자 | LLM 가시성 |
|------|------|--------|------------|
| 시스템 invariant | controller 코드 (base prompt) | 사람 | 매 호출 |
| 사람 지시 (목표·세션 정의) | `games/{game}/prompt.md` | 사람 (AI read-only) | 매 호출 |
| 게임간 공통 노하우 | `games/_shared/meta_notes.md` | AI | 매 호출 |
| 게임별 공략집 | `games/{game}/strategy_note.md` | AI | 매 호출 |
| 현재 세션 상태 | `games/{game}/sessions/{ts}/state.{ext}` | AI (양식 자유) | 매 호출 |
| 판단 근거 로그 | `games/{game}/sessions/{ts}/log.md` | AI | **아니오** (사람 디버깅용) |
| 세션 메트릭 | `games/{game}/sessions/{ts}/metrics.jsonl` | 시스템 | **아니오** (사람 분석용) |
| 과거 스크린샷 | `games/{game}/sessions/{ts}/screenshots/` | 시스템 | 일반 턴 X, **reflection 턴에서 도구로 접근** |

## 비전 파이프라인 (분담)

- **로컬 (Mac)**: 작은 비전 LLM + OCR + UI 요소 검출. 후보군은 Phase 0 PoC에서 비교 (OmniParser, PaddleOCR/Tesseract, Qwen2-VL 7B, MiniCPM-V 2.6 등).
- **클라우드**: 좌표 미관여. 의미 단위(`element_id`, 라벨)로 액션 결정.
- **좌표 그라운딩** (라벨 → 픽셀)은 Mac에서 일어남.

## LLM 입력 invariant

### 일반 턴

```
[system]
  - 액션 API 스펙
  - 시스템 invariant (2초 대기 / mouse_move == hover 효과 / 등)

[user]
  - games/_shared/meta_notes.md
  - games/{game}/prompt.md
  - games/{game}/strategy_note.md
  - games/{game}/sessions/{current}/state.{ext}
  - 로컬 비전 추출 결과 (라벨 + 좌표 사전)
  - 현재 스크린샷
```

이전 턴의 대화·도구 호출은 다음 호출에 포함되지 **않는다**.

### 세션 종료 reflection 턴

위와 동일하되 추가:

- 별도 reflection 시스템 메시지
- 도구 `get_session_screenshot(turn_n)` 활성화 — 이번 세션 과거 스크린샷 접근
- 도구 `update_strategy_note(...)`, `update_meta_notes(...)` 활성화
- 출력 요구: "예상 / 실제 / 갭" 3필드 (ADR-002 안전망 2)

## 디버그 / 감사

- 세션 폴더는 종료 후에도 보존
- `log.md`는 LLM에 안 들어가지만 사람이 매 턴의 thought·tool_call·결과를 추적할 수 있게 append-only로 기록
