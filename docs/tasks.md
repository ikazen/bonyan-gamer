# 작업 단계

## Phase 0: 모델 선정 PoC

목표: Mac M1 Pro 32GB에서 실용 가능한 로컬 비전 조합 확정 + Ollama Cloud 추론 모델 확정.

- [ ] 로컬 UI 요소 검출: OmniParser 후보 검증 (스크린샷 → 요소 리스트)
- [ ] 로컬 OCR: PaddleOCR vs Tesseract 비교 (속도·정확도)
- [ ] 로컬 비전 LLM 필요 여부 판단 (필요 시 Qwen2-VL 7B vs MiniCPM-V 2.6 8B)
- [ ] Mac 32GB 메모리 풋프린트 측정 — OmniParser + OCR + 비전 LLM(fp16 / Q8 / Q4) 동시 상주 시 사용량, OS·FastAPI 여유 확인. 한계 초과 시 ADR-014와 별개로 (A) 메모리 대응 ADR 박을 근거
- [ ] 클라우드 추론 모델 라인업 검토 후 1차 선정
- [ ] 한 사이클 end-to-end 토큰 비용 추산

## Phase 1: 인프라 구축

- [ ] uv 기반 프로젝트 셋업 (`pyproject.toml`)
- [ ] FastAPI 컨트롤러 골격 (`controller/main.py`)
- [ ] Windows 클라이언트 골격 (`executor/main.py`)
- [ ] Mac↔Windows HTTP API 정의 및 통신 테스트
- [ ] `init` / `step` / `finalize` lifecycle 구현
- [ ] 메모리 파일 I/O (`controller/memory/`)
- [ ] 액션 도구 → PyAutoGUI 매핑 + 2초 대기 내장
- [ ] 시스템 invariant base prompt 작성 (`controller/prompts/`)
- [ ] `games/_shared/prompt_template.md` 작성 (사용자용)

## Phase 2: Tametsi 1차 통합

목표: 클릭/우클릭 중심의 단순 게임에서 end-to-end 동작 확인.

- [ ] `games/tametsi/prompt.md` 작성 (사람 입력)
- [ ] 콜드 스타트 → 첫 세션 종료까지 동작
- [ ] reflection 강제 트리거 → `strategy_note.md` 자동 갱신 확인
- [ ] 5개 안전망 동작 검증 (특히 스키마 표류, 토큰 예산)
- [ ] 10회 세션 누적 후 학습 효과 정성 평가

## Phase 3: Balatro

목표: 드래그/호버를 통한 카드 조작 + 시너지 학습.

- [ ] `games/balatro/prompt.md` 작성
- [ ] `mouse_move` → 자동 2초 대기 → 다음 스크린샷에 툴팁 잡힘이 의도대로 작동하는지 검증
- [ ] D 액션 모델에서 턴당 도구 호출 패턴 관찰 → 예산 8회 적정성 평가
- [ ] 다중 카드 시너지를 AI가 인지하는지 정성 평가

## Phase 4: Slay the Spire

목표: 복합 조작 + 장기 전략.

- [ ] `games/sts/prompt.md` 작성
- [ ] 1런 완주 시도
- [ ] `meta_notes`에 게임간 공통 노하우 축적 확인

## 미정 / 후속

- 비동기 루프 (현재는 동기, PoC 성공 후 검토)
- 다중 모니터 / 비-창모드 환경 대응
- 테스트 / CI 도입
- 안전장치 (kill switch, 마우스 hijack 방지)
- 세션 종료 reflection 비용 최적화 (스크린샷 도구 호출 패턴)
