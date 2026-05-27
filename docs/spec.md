# 스펙

## 액션 API (LLM 도구)

모든 좌표는 픽셀 절대값 (ADR-005). 액션 도구는 Windows 클라이언트 내부에서 *실행 후 2초 대기 후* 반환 (ADR-006). 비전·메타 도구는 대기 없음.

### 마우스

| 도구 | 인자 | 설명 |
|------|------|------|
| `mouse_move(x, y)` | x, y: int | 마우스 이동. 호버 효과는 자동 2초 대기로 자연 발생 (ADR-007) |
| `click(x, y, button="left")` | x, y; button ∈ {"left", "right"} | 단일 클릭 |
| `double_click(x, y)` | x, y | 더블 클릭 |
| `drag(x1, y1, x2, y2)` | 시작·끝점 | 드래그 |
| `scroll(amount)` | amount: int (음수 = 아래) | 스크롤 |

### 키보드

| 도구 | 인자 | 설명 |
|------|------|------|
| `press_key(key)` | key: str (PyAutoGUI 키명) | 단일 키 |
| `hotkey(*keys)` | 가변 키 리스트 | 조합 키 (예: Ctrl+Z) |
| `type_text(text)` | text: str | 문자열 입력 |

### 비전 (대기 없음)

| 도구 | 인자 | 설명 |
|------|------|------|
| `get_screenshot()` | — | 현재 전체 화면 |
| `get_crop_image(x1, y1, x2, y2)` | 영역 좌표 | 부분 확대 |

### 메타

| 도구 | 인자 | 설명 |
|------|------|------|
| `end_turn()` | — | 이번 추론 턴 종료, 다음 사이클로 |
| `end_session()` | — | 세션 종료 선언, `finalize()` 트리거 |

**턴당 도구 호출 예산**: 8회 (`end_turn` 포함). 초과 시 시스템이 강제로 `end_turn` 처리 (ADR-003).

### 액션 도구 응답 페이로드 (ADR-012)

마우스·키보드 액션 도구의 tool_result는 다음 페이로드를 반환한다. 스크린샷은 자동 첨부하지 않으며, 필요하면 LLM이 같은 턴에 `get_screenshot()`를 호출한다.

- 성공: `{"ok": true, "executed_at": [x, y], "elapsed_ms": int}`
  - 키보드 액션은 `executed_at` 생략 가능.
- 실패: `{"ok": false, "reason": str}`
  - `reason` 후보: `out_of_bounds`, `failsafe_triggered`, `windows_unreachable`, `retry_exceeded` (ADR-009), `aborted` (ADR-013).

"클릭 후 화면이 변하지 않음" 같은 의미적 실패는 시스템이 판정하지 않는다. 다음 턴 state에서 LLM이 판단 (ADR-009).

### Reflection 턴 전용 도구

`finalize()`로 진입한 reflection 턴에서만 활성화.

| 도구 | 인자 | 설명 |
|------|------|------|
| `get_session_screenshot(turn_n)` | turn 번호 | 이번 세션의 과거 스크린샷 |
| `update_strategy_note(content)` | str | 게임 공략집 갱신 |
| `update_meta_notes(content)` | str | 게임간 공통 노하우 갱신 |

출력 요구: "예상 / 실제 / 갭" 3필드 포함 (ADR-002 안전망 2).

## 세션 lifecycle API (controller 내부)

```python
session_id: str = init(game: str)        # timestamp 반환, 세션 폴더 생성
step(session_id)                          # 한 사이클: screenshot → infer → act
finalize(session_id)                      # 종료 + reflection 강제 트리거
```

- `session_id` 포맷: `YYYY-MM-DDTHH-mm-ss`
- 세션 종료 트리거 3가지:
  1. AI가 `end_session()` 호출
  2. 사용자가 명시적으로 finalize 호출
  3. 시스템이 게임오버 감지 (`prompt.md`에 정의된 조건)
- 세션 폴더는 종료 후에도 보존 (감사·디버깅·미래 reflection 참조)

## 디렉토리 규약

```
games/
├── _shared/
│   └── meta_notes.md            # 게임 공통 노하우 (AI 갱신)
└── {game_name}/
    ├── prompt.md                # 사람 작성, AI read-only
    ├── strategy_note.md         # AI 공략집
    └── sessions/
        └── {YYYY-MM-DDTHH-mm-ss}/
            ├── state.{ext}      # AI 결정 양식·확장자
            ├── log.md           # 판단 근거 append-only (사람 디버깅용)
            ├── metrics.jsonl    # 매 step 메트릭 append (사람 분석용)
            └── screenshots/
                ├── turn_001.png
                └── ...
```

### `prompt.md` 템플릿 필드 (사람이 작성)

각 게임 폴더에 사람이 작성. 예시 파일은 Phase 1에서 `games/_shared/prompt_template.md`로 제공 예정.

- `## 최종 목표` — 사람이 정한 게임별 성공 기준 (예: "죽지 않고 5스테이지 클리어")
- `## 세션 단위` — 무엇이 세션 종료를 의미하는지 (사람 설명, 자연어)
- `## 세션 종료 조건` — 시스템이 자동 감지에 사용할 OCR 매칭 문자열 (아래 형식 참조)
- `## 게임 규칙` — AI에게 명시적으로 알려주고 싶은 규칙·주의사항
- `## 입력 특이사항` — 키 매핑·특수 조작 (있으면)

#### `## 세션 종료 조건` 형식 (PoC)

OCR 부분 문자열 매칭으로 단순화. or 관계 — 하나라도 화면에 매칭되면 `finalize()` 자동 트리거.

```markdown
## 세션 종료 조건
- "Game Over"
- "Defeat"
- "You Died"
```

시스템 처리: 매 step 후 로컬 OCR 결과 전체 텍스트에서 위 문자열들을 검색. 매칭 시 ADR-002 안전망 3(세션 종료 강제 consolidation) 경로로 진입.

대소문자 구분 없음. 정규식·영역 제한·비전 LLM 기반 판정은 PoC 이후 확장 여지 (Phase 3+).

## LLM 입력 구성

### 일반 턴

```
system:
  - 액션 API 스펙
  - 시스템 invariant (ADR-011: controller/prompts/ 코드 상수)

user:
  games/_shared/meta_notes.md
  games/{game}/prompt.md
  games/{game}/strategy_note.md
  games/{game}/sessions/{current}/state.{ext}
  로컬 비전 추출 결과 (라벨 ↔ 좌표 사전, ADR-012 형식)
  현재 스크린샷
```

라벨 ↔ 좌표 사전 형식 (ADR-012):

```
element_id  label          box(x1,y1,x2,y2)    click_at(x,y)
cell_0_0    "빈 칸"         (10,40,50,80)       (30,60)
tooltip     "마인 3"        (200,300,360,340)   (280,320)
```

LLM은 사전에서 좌표를 골라 액션 인자에 넣는다. 사전 외 자유 좌표 사용도 허용.

이전 턴의 대화·도구 호출은 **포함되지 않음** (ADR-004).

### Reflection 턴

위와 동일하되 다음 추가:
- 별도 reflection 시스템 메시지
- Reflection 턴 전용 도구 활성화
- "예상 / 실제 / 갭" 출력 강제

## HTTP API (Windows ↔ Mac)

방향: Windows = HTTP 클라이언트, Mac = FastAPI 서버.

세부 엔드포인트는 구현 단계(Phase 1)에서 확정 후 본 문서에 추가. 대략:

- Windows → Mac: 스크린샷 + 세션 ID POST → 응답으로 다음 액션 시퀀스
- 동기 단방향 (ADR-008)

### 안전장치 (ADR-013)

- Windows executor 부팅 시 `pyautogui.FAILSAFE = True`. 사용자가 마우스를 좌상단(0,0)으로 던지면 즉시 정지.
- Mac controller: `POST /abort` (전체 정지·다음 step 차단), `POST /resume` (재개).
- Windows executor: `POST /abort` (이후 액션 요청에 `aborted` 응답).
- 모든 abort/failsafe 사유는 ADR-012 tool_result 페이로드의 `reason`으로 LLM에 전달.

## 메모리 파일 사이즈 가이드

PoC 초기값. 운영 중 측정해 조정. 토큰 카운팅은 controller가 일관된 토크나이저로 수행.

| 파일 | 권장 상한 (토큰) | 초과 시 동작 |
|------|------------------|--------------|
| `_shared/meta_notes.md` | 2000 | 다음 프롬프트에 압축 권고 한 줄 끼움. AI 자율 압축 (ADR-002 안전망 5와 같은 정신). |
| `{game}/prompt.md` | 2000 | 사람 작성. 시스템은 경고만, 자동 압축 안 함. |
| `{game}/strategy_note.md` | 4000 | 압축 권고. AI 자율 압축. |
| `{game}/sessions/{ts}/state.{ext}` | 1500 | 압축 권고. AI 자율 압축. |

## 세션 메트릭 기록

매 step 후 `games/{game}/sessions/{ts}/metrics.jsonl`에 한 줄 append:

```json
{"ts": "2026-05-27T18-30-01", "turn": 12, "tool_calls": 5, "input_tokens": 3421, "output_tokens": 187, "latency_ms": 4210, "model": "qwen3-coder"}
```

LLM 컨텍스트에 들어가지 않음. 사람 분석·디버깅용. ADR-002 안전망 2("예상/실제/갭")의 정량 보조.

## 시크릿 / 환경변수

`.env` (gitignore), 템플릿은 `.env.example`.

```
OLLAMA_CLOUD_API_KEY=
OLLAMA_CLOUD_ENDPOINT=
CONTROLLER_HOST=0.0.0.0
CONTROLLER_PORT=8000
```

## 미정 (구현 단계에서 채울 항목)

- 로컬 비전 모델 조합 (Phase 0 PoC 결과)
- 클라우드 추론 모델 선정 (Phase 0 PoC 결과)
- HTTP 엔드포인트 상세 시그니처 (Phase 1)
