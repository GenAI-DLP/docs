# log-event 스키마

모든 파이프라인 판정의 구조화 감사 로그. 관리자 대시보드와 성능 평가 스크립트가 읽는다.
**감사 계층** — append-only 지향, 운영·볼트 테이블로의 FK 없음(세션 삭제 후에도 보존).

DDL 원본: [`postgres-schema.sql`](postgres-schema.sql) §4
설계 근거: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §3(스테이지 9), §7.2
`reason` 페이로드 형태: [`../../architecture/dlp-proto.md`](../../architecture/dlp-proto.md) §3.2

---

## `log_events`

| 컬럼 | 타입 | 의미 |
|---|---|---|
| `event_id` | BIGSERIAL PK | |
| `session_id` | UUID (**FK 없음**) | 상관관계용 값 컬럼 |
| `direction` | `direction_type` | input / output |
| `provider` | VARCHAR(32) | gateway / openai / anthropic |
| `purpose` | TEXT (FK 없음) | 목적 분류 결과 (감사 시점 스냅샷) |
| `verdict_action` | `verdict_action` | 실제 프록시로 전송된 3값 (allow/block/transform) |
| `transforms` | JSONB | 세부 변환 종류 리스트. 예: `[{"entity":"RRN","action":"tokenize"}]` |
| `entities_summary` | JSONB | `[{type, masked_preview, confidence}]` — **원문 금지, 마스킹 미리보기만** |
| `guardrail_hits` | JSONB | Input/Output Guard 적중 내역 |
| `fail_policy_applied` | BOOLEAN | true = DLP 판정이 아니라 장애 대응(fail-closed)으로 나간 결과 |
| `latency_ms` | INT | 파이프라인 처리 시간 |
| `reason` | JSONB | `Decision.reason_obj` 전체 |
| `created_at` | TIMESTAMPTZ | |

### 인덱스

- `(session_id, created_at)` — 세션별 타임라인
- `(created_at DESC)` — 대시보드 최신순 tail
- `(verdict_action)` — 조치별 집계
- GIN `(entities_summary)` — 엔티티 타입 필터

---

## 규칙 · 불변식

- **원문 무저장.** `entities_summary` 는 `masked_preview` 만. `reason` 에도 원문을 넣지 않는다.
- **append-only 지향.** 운영에서는 `REVOKE UPDATE, DELETE ... FROM app_role`,
  `created_at` 월별 파티셔닝.
- `fail_policy_applied = true` 인 행은 "DLP가 실제로 내린 판단"이 아니라 "장애로 인한 강제 조치"이므로
  탐지율·오탐률 집계에서 구분한다.
- `verdict_action` 은 wire 3값. 세부 전략은 `transforms` 에서 본다
  ([dlp-proto](../../architecture/dlp-proto.md) §3.1).

## 대시보드 · eval 소비

- 대시보드(Streamlit)는 `api.py` 의 `/events` 로 최근 로그를 tail 하고, 세션별 탐지·목적·조치·지연을
  표시한다(원문 미노출).
- `eval/run_eval.py` 는 baseline vs full 두 모드의 로그를 모아 Detection Rate / FPR / latency 를 계산.

## 데모 구현

`logging/events.py` — `LogEvent` 구조화 후 **PostgreSQL `log_events` INSERT** sink. 컬럼 의미는
위와 동일. `/events` 는 이 테이블을 최신순으로 조회한다. append-only 파티셔닝·검색엔진
(Elasticsearch 등) 이관은 확장.
