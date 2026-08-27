# 하이브리드 PII 탐지

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 필수
**한 줄 정의:** 정규식/체크섬 + 사전(Aho-Corasick) + NER 세 레이어를 병렬 실행하고 병합해 정형·비정형 금융 PII를 동시 탐지한다.
**담당 모듈:** `detect/regex_rules.py`, `detect/dictionary.py`, `detect/ner.py`, `detect/merge.py`

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
