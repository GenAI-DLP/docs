# docs — 생성형 AI Dynamic DLP Gateway 공식 문서

설계·계약·기능 스펙 문서 모음. 이 README는 **어디에 무엇이 있는지 알려주는 지도**다.

## 읽는 순서

각 폴더의 **색인 문서를 먼저** 읽어라. 색인이 폴더 구성과 각 문서의 역할을 정리한다.

| 폴더 | 먼저 읽을 색인 | 폴더의 내용 |
|---|---|---|
| `architecture/` | `architecture/architecture-index.md` | 컴포넌트 구조, 모듈 경계, 데이터 흐름, gRPC 계약 |
| `schemas/` | `schemas/schema-index.md` | DB 테이블·계약 스키마와 설계 근거 |
| `specs/` | `specs/spec-index.md` | 기능 a~h의 입출력 예시·판정 로직·임계값 |

## 목적별 진입점

| 하려는 것 | 파일 |
|---|---|
| 프로젝트 전체 처음 파악 | `architecture/architecture-index.md` |
| dlp-server 전체 구성 (파이프라인·모듈·기능 개요·저장소·로드맵) | `architecture/dlp-server-architecture.md` |
| Go 프록시 ↔ dlp-server gRPC 연동 | `architecture/dlp-proto.md` — 계약이므로 임의 변경 금지 |
| 기능 하나(a~h) 구현 | `specs/spec-index.md` → `specs/dlp-server/` 의 해당 파일 |
| DB 테이블·컬럼 확인/변경 | `schemas/dlp-server/postgres-schema.sql` (SSOT) + `schemas/dlp-server/*.md` |
| 판정 결과 / `reason` JSON 형태 | `architecture/dlp-proto.md` §3.2 |
| 대시보드가 읽는 로그 포맷 | `schemas/dlp-server/log-event.md` |
| dlp-proxy-server 설계 | `architecture/dlp-proxy-server-architecture.md` (스텁) |

## 파일 찾기 (Tool 사용 시)

- `specs/dlp-server/` 에는 `a_` ~ `h_` 접두사 파일이 8개 있다. 접두사는 아키텍처 문서 §6의 기능 ID와 일치.
- 예시 파일명: `specs/dlp-server/a_reversible-tokenization.md`, `specs/dlp-server/e_multiturn-context-analysis.md`
- Glob 패턴: `specs/dlp-server/*.md` (전체) · `specs/dlp-server/e_*.md` (기능 e)

## 규칙

- **SSOT:** gRPC 계약 = `architecture/dlp-proto.md` + 공유 `.proto` · DB 스키마 = `schemas/dlp-server/postgres-schema.sql`. 변경 시 참조 문서도 함께 갱신.
- `architecture/`·`schemas/`·`specs/` 는 컴포넌트(`dlp-server` / `dlp-proxy-server`)별로 분리.
