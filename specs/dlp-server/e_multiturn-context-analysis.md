# Multi-turn Context 분석

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 필수
**한 줄 정의:** 세션 전체 대화를 추적해 여러 턴에 분산 입력된 PII 조합을 엔티티 그래프 + 슬라이딩 윈도우로 탐지하고 누적 위험도를 매긴다.
**담당 모듈:** `context/store.py`, `context/accumulator.py`

설계 의도·전체 맥락: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6
관련 스키마: [`../../schemas/dlp-server/session-context.md`](../../schemas/dlp-server/session-context.md)
색인: [`../spec-index.md`](../spec-index.md)

---

## 목차 (작성 예정)

1. 입력 예시
2. 출력 예시
3. 판정 로직 (룰 / 프롬프트 / 임계값)
4. 파라미터 · 설정값
5. 엣지 케이스
6. 테스트 케이스표
