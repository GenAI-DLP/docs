# schemas — 데이터 계약 색인

dlp-server가 소유하는 데이터 저장소·계약 스키마. 프록시는 DB가 없고 wire 계약만 가지므로
([`../architecture/dlp-proto.md`](../architecture/dlp-proto.md)), 현재 스키마는 모두 `dlp-server/` 아래에 있다.

---

## 3계층 데이터 모델

| 계층 | 테이블 | 수명 관리 | 다른 계층으로의 FK |
|---|---|---|---|
| **운영** | `sessions`, `session_turns`, `session_entities` | `expires_at` TTL → 하드 삭제(CASCADE) | — |
| **볼트** | `token_vault` | 자체 `expires_at` (세션과 분리) | 없음 |
| **감사** | `log_events`, `token_vault_access_log` | 장기 보존, append-only 지향 | 없음 (세션 삭제 후에도 보존) |

핵심: **감사·볼트 테이블은 운영 테이블로의 FK를 두지 않는다.** 세션 TTL 삭제가 CASCADE로 감사 로그를
지우는 문제를 원천 차단하기 위함. 상세 근거 8가지는
[`../architecture/dlp-server-architecture.md`](../architecture/dlp-server-architecture.md) §7.2.

---

## 파일

| 파일 | 내용 | 대응 기능 |
|---|---|---|
| [`dlp-server/postgres-schema.sql`](dlp-server/postgres-schema.sql) | **실행 가능한 DDL 원본 (SSOT).** 4개 계약 전체 + ENUM + 조회 테이블 + `purge_expired()` | — |
| [`dlp-server/session-context.md`](dlp-server/session-context.md) | 세션·턴·누적 엔티티. 원문 무저장 | e (멀티턴) |
| [`dlp-server/token-vault.md`](dlp-server/token-vault.md) | 토큰 ↔ 원문 매핑, `access_scope`, soft revoke | a (토큰화) |
| [`dlp-server/policy.md`](dlp-server/policy.md) | `(목적 × role × 엔티티) → 조치` 버전 테이블, 위험도 오버라이드 | f (접근 제어) |
| [`dlp-server/log-event.md`](dlp-server/log-event.md) | 판정 감사 로그. 대시보드·eval 소비 | 전체 |

---

## 변경 규칙

- `postgres-schema.sql` 이 단일 기준. 컬럼·제약·인덱스를 바꾸면 이 파일을 먼저 고치고,
  해당 `*.md` 와 `../architecture/dlp-server-architecture.md` §7 을 함께 갱신한다.
- `entity_type_ref` / `purpose_ref` 는 seed로 시작. 목록 확정 시 `INSERT` 행만 추가.
  코드(`enums`)와 DB 중 하나를 단일 기준으로 정한다 (미정).
- `verdict_action` ENUM 은 wire 계약([dlp-proto](../architecture/dlp-proto.md))과 3값이 항상 일치해야 한다.
