# 동적 데이터 변환

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 필수
**한 줄 정의:** 정책 엔진의 결정을 keep/mask/generalize/aggregate/tokenize/synthetic/redact/block 전략으로 실행한다. 같은 PII라도 목적·role·위험도에 따라 다른 전략이 런타임 선택된다.
**담당 모듈:** `transform/apply.py` (tokenize는 기능 a와 공유)

설계 의도·전체 맥락: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6
관련 스키마: [`../../schemas/dlp-server/policy.md`](../../schemas/dlp-server/policy.md)
색인: [`../spec-index.md`](../spec-index.md)

---

## 목차 (작성 예정)

1. 입력 예시
2. 출력 예시
3. 판정 로직 (룰 / 프롬프트 / 임계값)
4. 파라미터 · 설정값
5. 엣지 케이스
6. 테스트 케이스표
