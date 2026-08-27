# 가역적 토큰화

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 필수
**한 줄 정의:** 외부 LLM 전송 전 PII를 결정론적 토큰으로 치환하고, 응답 수신 후 인가된 요청자에 한해 원본으로 복원한다.
**담당 모듈:** `transform/vault.py`, `transform/apply.py`(tokenize), Output Guard의 detokenize

설계 의도·전체 맥락: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6
관련 스키마: [`../../schemas/dlp-server/token-vault.md`](../../schemas/dlp-server/token-vault.md)
색인: [`../spec-index.md`](../spec-index.md)

---

## 목차 (작성 예정)

1. 입력 예시
2. 출력 예시
3. 판정 로직 (룰 / 프롬프트 / 임계값)
4. 파라미터 · 설정값
5. 엣지 케이스
6. 테스트 케이스표
