# architecture — 색인 & 전체 구조

생성형 AI Dynamic DLP Gateway의 컴포넌트 구성과 각 아키텍처 문서로의 링크.
"무엇이 어디에 있는가"에 대한 답은 이 파일, "어떻게 설계했는가"는 각 컴포넌트 문서.

---

## 1. 컴포넌트 & 레포

| 컴포넌트 | 레포 | 언어/스택 | 역할 |
|---|---|---|---|
| **dlp-proxy-server** | `dlp-proxy-server` | Go | 직원 PC ↔ 외부 LLM 트래픽을 SNI allowlist로 거르고, 대상만 TLS 종단(MITM). HTTP 파싱 후 gRPC로 판정 요청, 판정대로 외부 LLM에 중계 |
| **dlp-server** | `dlp-server` | Python / FastAPI | 의미 기반 검사 백엔드. Input/Output Guard, 멀티턴 분석, PII 탐지, 목적·정책, 동적 변환·토큰화 |
| **dlp-proto** | 공유 소스 (서브모듈) | protobuf | 위 둘 사이의 gRPC 계약. 양쪽이 공유, 임의 변경 금지 |
| 대시보드 | `dashboard` | React | 감사 로그를 조회(`/events`)·실시간 tail(`/events/stream` SSE)해 탐지·차단 현황 표시 |

---

## 2. 데이터 흐름

```
[직원 PC]
   │  HTTPS(TCP)
   ▼
[dlp-proxy-server]  SNI allowlist → 대상만 TLS 종단(MITM) → HTTP 파싱
   │  gRPC  Inspect(InspectRequest, direction="input")   ── 트래픽 Hold
   ▼
[dlp-server]  입력 파이프라인 (Input Guard → 탐지 → 멀티턴 → 목적·정책 → 변환)
   │  Verdict(allow | block | transform)
   ▼
[dlp-proxy-server]  판정대로 외부 LLM에 중계
   ▼
[외부 LLM]
   │  응답
   ▼
[dlp-proxy-server]  다시 Hold → gRPC Inspect(direction="output")
   ▼
[dlp-server]  출력 파이프라인 (Output Guard 재스캔 → detokenize → 재조립)
   │  Verdict
   ▼
[dlp-proxy-server] → [직원 PC]
                       │
[dlp-server] ── 구조화 감사 로그 ──▶ [대시보드]
```

배포 형태: 사내망 내부. 프록시-백엔드 간 gRPC 는 평문(사내망 전제), deadline 3초.
프록시가 백엔드 응답을 못 받으면 fail-closed(차단).

---

## 3. 문서

| 파일 | 내용 | 상태 |
|---|---|---|
| [`dlp-server-architecture.md`](dlp-server-architecture.md) | dlp-server 전체: 파이프라인, 모듈 구조, 계약 타입, 기능 a~h, 저장소, 로드맵 | 작성됨 |
| [`dlp-proto.md`](dlp-proto.md) | Proxy ↔ dlp-server gRPC 계약. `InspectRequest`/`Verdict`, action 3값 규칙, 에러/타임아웃 | 작성됨 |
| [`dlp-proxy-server-architecture.md`](dlp-proxy-server-architecture.md) | dlp-proxy-server 전체: 투명 프록시, MITM, TCP/UDP | 스텁 (프록시팀 작성 예정) |

관련: 데이터 스키마는 [`../schemas/`](../schemas/), 기능별 상세 구현 스펙은 [`../specs/`](../specs/).
