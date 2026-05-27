# Bonyan Gamer

범용 게임 에이전트

키보드와 마우스만으로 인간의 조작을 모방하고, 매 세션 종료마다 자기 성찰을 통해 공략집을 스스로 갱신하는 AI 에이전트.

## 핵심 아이디어

- 조작 매체는 오직 물리 키보드/마우스 (PyAutoGUI)
- 매 턴 LLM은 stateless — 모든 기억은 파일에 둠 (Voyager / MemGPT 류 패턴)
- 상태 표현 양식과 갱신 시점까지 AI가 스스로 결정
- 사람은 게임별 최종 목표와 세션 정의만 제공

## 시스템 구성

세 머신 분리. V/L/A를 칼같이 나누지 않음 (ADR-001).

- **Mac (controller)**: FastAPI 서버, 게임 루프 구동, 로컬 비전·OCR·라벨링, 좌표 그라운딩, 메모리 관리, 클라우드 추론 라우팅. *두뇌 + 척수 + 눈*.
- **Windows (executor)**: HTTP 클라이언트, PyAutoGUI 액션 실행, 스크린샷 캡처. *손*.
- **Ollama Cloud**: 복잡 추론(L)만. 좌표는 보지 않음.

자세한 흐름은 [docs/architecture.md](docs/architecture.md) 참조.

## 폴더 구조

```
.
├── README.md
├── docs/                # 기획·설계·결정 이력
├── pyproject.toml       # (Phase 1에서 도입)
├── .env.example
├── controller/          # Mac 쪽 코드 (서버 + 루프 + 비전 + 추론)
├── executor/            # Windows 쪽 코드 (HTTP 클라이언트 + PyAutoGUI)
├── shared/              # 양쪽 공통 (액션 스키마)
└── games/               # AI 메모리 영역 (게임별 분리)
    ├── _shared/
    │   └── meta_notes.md
    └── {game}/
        ├── prompt.md
        ├── strategy_note.md
        └── sessions/{timestamp}/
```

## 진입점

- 컨트롤러 서버 (Mac): `controller/main.py` *(Phase 1)*
- 실행기 클라이언트 (Windows): `executor/main.py` *(Phase 1)*

## 문서

- [docs/architecture.md](docs/architecture.md) — 3대 머신 구조, 데이터 흐름, 메모리 계층
- [docs/decisions.md](docs/decisions.md) — ADR 누적
- [docs/spec.md](docs/spec.md) — 액션 API, 세션 lifecycle, 디렉토리 규약
- [docs/tasks.md](docs/tasks.md) — 단계별 작업 진행 상태

## 상태

기획 단계. 다음 작업: Phase 0 (모델 선정 PoC).
