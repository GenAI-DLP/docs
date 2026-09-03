# specs — 기능별 상세 스펙 색인

각 기능을 문서 하나만 보고 구현할 수 있도록 입력/출력 예시, 판정 로직(룰·프롬프트·임계값), 엣지
케이스를 담는 문서 모음. 아키텍처(설계 의도)는 [`../architecture/`](../architecture/), 데이터 계약은
[`../schemas/`](../schemas/).

상태: ✅ 작성됨 · 🚧 작성 중 · ⬜ 스텁(제목·목차만)

---

## dlp-server — 공통 계약

기능 a~h 를 붙이기 전에 읽는다. 아키텍처 문서 §3/§5 가 설계 의도라면, 이건 구현된 계약.

| 파일 | 내용 | 상태 |
|---|---|---|
| [`dlp-server/pipeline-stage-contract.md`](dlp-server/pipeline-stage-contract.md) | 파이프라인 스테이지 구현 계약 — `Stage` 시그니처, `AnalysisContext` 필드, 스테이지 등록·실행 순서, block/allow/transform 판정 규칙, `reason_obj` 형태, fail-closed | ✅ |

---

## dlp-server (기능 a~h)

기능 개요와 모듈 매핑은 [`../architecture/dlp-server-architecture.md`](../architecture/dlp-server-architecture.md) §6.

| ID | 파일 | 기능 | 우선순위 | 상태 |
|---|---|---|---|---|
| a | [`dlp-server/a_reversible-tokenization.md`](dlp-server/a_reversible-tokenization.md) | 가역적 토큰화 | 필수 | ✅ |
| b | [`dlp-server/b_hybrid-pii-detection.md`](dlp-server/b_hybrid-pii-detection.md) | 하이브리드 PII 탐지 (regex + 사전 + NER) | 필수 | 🚧 (구현·배선됨, 스펙 문서 미완) |
| c | [`dlp-server/c_bidirectional-guardrails.md`](dlp-server/c_bidirectional-guardrails.md) | 양방향 가드레일 (Input / Output) | 필수 | ✅ (Input/Output 구현·배선. eval 튜닝 남음) |
| d | [`dlp-server/d_agent-tool-control.md`](dlp-server/d_agent-tool-control.md) | AI Agent / MCP Tool 통제 | 후순위 | ⬜ |
| e | [`dlp-server/e_multiturn-context-analysis.md`](dlp-server/e_multiturn-context-analysis.md) | Multi-turn Context 분석 | 필수 | ✅ |
| f | [`dlp-server/f_purpose-based-access-control.md`](dlp-server/f_purpose-based-access-control.md) | 목적 기반 동적 데이터 접근 제어 | 필수 | ✅ |
| g | [`dlp-server/g_dynamic-data-transformation.md`](dlp-server/g_dynamic-data-transformation.md) | 동적 데이터 변환 (mask/generalize/…) | 필수 | ✅ |
| h | [`dlp-server/h_agent-behavior-sequence.md`](dlp-server/h_agent-behavior-sequence.md) | Agent 행동 시퀀스 분석 | 후순위 | ⬜ |

## dlp-proxy-server

`dlp-proxy-server/` — 프록시팀 스펙 예정. 현재 없음.

---

## 작성 규칙

- 파일명 접두사 `a_` ~ `h_` 는 아키텍처 문서 §6의 기능 ID와 일치시킨다.
- 각 스펙은 최소한 다음을 포함: **입력 예시 · 출력 예시 · 판정 로직 · 임계값/파라미터 · 엣지 케이스 · 관련 스키마 링크**.
- 후순위(d, h)는 개요 + 인터페이스 스케치까지만 있어도 된다.
