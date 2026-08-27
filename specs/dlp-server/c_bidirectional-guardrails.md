# 양방향 가드레일 (Input / Output)

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 필수
**한 줄 정의:** LLM 요청(인젝션·탈옥·반출 시도)과 응답(타인 PII 재생성·인젝션 순응·정책 위반)을 모두 검사한다.
**담당 모듈:** `guardrail/injection.py`, `guardrail/output_check.py` (탐지 엔진은 기능 b 재사용)

설계 의도·전체 맥락: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6
관련 스키마: [`../../schemas/dlp-server/log-event.md`](../../schemas/dlp-server/log-event.md)
색인: [`../spec-index.md`](../spec-index.md)

---

## 목차 (작성 예정)

1. 입력 예시
2. 출력 예시
3. 판정 로직 (룰 / 프롬프트 / 임계값)
4. 파라미터 · 설정값
5. 엣지 케이스
6. 테스트 케이스표
