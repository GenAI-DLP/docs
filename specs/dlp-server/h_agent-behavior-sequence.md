# Agent 행동 시퀀스 분석

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 후순위
**한 줄 정의:** 개별 tool call이 아닌 연속 호출 패턴(예: 검색→전체 export→외부 전송)을 상태 기계 또는 n-gram 빈도 분석으로 이상 탐지한다.
**담당 모듈:** `agent_control/sequence_analyzer.py` (기능 e의 세션 이벤트 로그 인프라 재사용)

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
