# AI Agent / MCP Tool 통제

> **스텁.** 제목·목적·목차만. 구현 착수 전 채운다.

**우선순위:** 후순위
**한 줄 정의:** MCP 클라이언트↔서버 사이에서 tool call의 입력/출력에 탐지·정책 파이프라인을 동일 적용하고 tool별 최소 권한을 강제한다.
**담당 모듈:** `agent_control/mcp_proxy.py` (스켈레톤)

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
