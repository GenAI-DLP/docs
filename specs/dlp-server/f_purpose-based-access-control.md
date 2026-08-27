# 목적 기반 동적 데이터 접근 제어

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 필수
**한 줄 정의:** 요청 목적과 요청자 role, 엔티티 민감도, 누적 위험도를 조합해 엔티티 단위로 조치(ALLOW/MASK/TOKENIZE/BLOCK 등)를 결정한다.
**담당 모듈:** `purpose/classifier.py`, `purpose/role_resolver.py`, `policy/engine.py`, `policy/policy.yaml`

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
