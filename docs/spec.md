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
            └── screenshots/
                ├── turn_001.png
                └── ...
```

### `prompt.md` 템플릿 필드 (사람이 작성)

각 게임 폴더에 사람이 작성. 예시 파일은 Phase 1에서 `games/_shared/prompt_template.md`로 제공 예정.

- `## 최종 목표` — 사람이 정한 게임별 성공 기준 (예: "죽지 않고 5스테이지 클리어")
- `## 세션 단위` — 무엇이 세션 종료를 의미하는지 (예: "게임오버 화면 감지", "한 런 완주")
- `## 게임 규칙` — AI에게 명시적으로 알려주고 싶은 규칙·주의사항
- `## 입력 특이사항` — 키 매핑·특수 조작 (있으면)

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
  로컬 비전 추출 결과 (라벨 + 좌표 사전)
  현재 스크린샷
```

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
- state 파일 토큰 예산 N값 (운영하며 조정)
