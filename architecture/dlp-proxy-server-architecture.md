# dlp-proxy-server 아키텍처

> **스텁.** dlp-proxy-server(Go) 팀이 작성 예정. 현재는 목차 골격과 확정된 계약만 기록한다.

네트워크 레벨 컴포넌트. 직원 PC ↔ 외부 LLM 트래픽을 가로채 TLS를 종단(MITM)하고, HTTP를 파싱해
dlp-server에 검사를 맡긴 뒤 판정대로 중계한다.

전체 시스템에서의 위치: [`architecture-index.md`](architecture-index.md)
dlp-server와의 계약: [`dlp-proto.md`](dlp-proto.md) — **확정**, 임의 변경 금지

---

## 확정 사항 (계약 관련)

- gRPC `Inspect` **unary** 호출. 요청·응답 각 1회. 판정까지 트래픽 Hold.
- `Verdict.action` 3값(`allow` / `block` / `transform`)만 처리.
- 스트리밍 응답은 **전체 버퍼링** 후 한 번에 검사.
- dlp-server 무응답(타임아웃/장애) 시 **fail-closed(차단)**, 로그에 `fail_policy_applied` 플래그.
- `session_id` 추출 우선순위: `X-Corp-User-Id` 헤더 → 서비스 세션 쿠키 → 원격 주소.

---

## 목차 (작성 예정)

1. 배포 모델 — 투명 프록시 / SNI allowlist / 회사 루트 CA 배포
2. TCP 경로 — TLS 종단(MITM), HTTP/1.1 파싱, 업스트림 재연결
3. UDP/QUIC 경로 — HTTP/3 대응 범위
4. 인증서 발급 — SNI 기반 leaf 인증서 on-the-fly, 캐싱
5. dlp-server 연동 — gRPC 클라이언트, deadline, fail 정책, 재연결 백오프
6. 로깅 — 감사 로그 포맷([`../schemas/dlp-server/log-event.md`](../schemas/dlp-server/log-event.md))
7. 설정 — allowlist, 리스닝 포트, DLP 주소/타임아웃, CA 경로
8. 로컬 테스트 — mock DLP 서버, CA 생성, 스모크 테스트
