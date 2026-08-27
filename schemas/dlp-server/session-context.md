# session-context 스키마

멀티턴 분석(기능 e)이 세션 단위로 누적하는 대화 상태. **운영 계층**이며 `expires_at` TTL 도달 시
하드 삭제된다(하위 `session_turns` / `session_entities` 는 CASCADE).

DDL 원본: [`postgres-schema.sql`](postgres-schema.sql) §1
설계 근거: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §3(파이프라인), §6-e, §7

---

## 테이블

### `sessions`

| 컬럼 | 타입 | 의미 |
|---|---|---|
| `session_id` | UUID PK | 프록시가 넘긴 `session_id`([dlp-proto](../../architecture/dlp-proto.md) §2.1)를 그대로 사용. 신규면 생성 |
| `provider` | VARCHAR(32) | `gateway` / `openai` / `anthropic` — 어댑터 판별 결과 |
| `role` | VARCHAR(64) | 헤더에서 해석한 요청자 role (`role_resolver`). 정책 엔진 입력 |
| `purpose` | TEXT → `purpose_ref` | 목적 분류 결과. 기본 `unknown` |
| `purpose_confidence` | REAL | 목적 분류 신뢰도 |
| `turn_count` | INT | 세션 내 턴 수 |
| `risk_score` | REAL 0.0~1.0 | 누적 위험도. `CHECK` 제약으로 범위 강제 |
| `created_at` / `last_seen_at` | TIMESTAMPTZ | 생성·최근 접근 시각 |
| `expires_at` | TIMESTAMPTZ | TTL. `purge_expired()` 가 이 값 기준 삭제 |

### `session_turns`

턴 단위 메타데이터. **원문은 저장하지 않는다** — `text_hash`(SHA-256)와 `text_length`만.

- `UNIQUE (session_id, turn_index)` — 턴 순번은 세션 내 0부터, 중복 불가.
- `direction` — `input` / `output`.

### `session_entities`

턴별로 탐지된 PII 엔티티의 누적 기록. 원본 값은 여기 없고(볼트가 관리) `value_hash`만.

- `turn_id` FK로 어느 턴에서 나왔는지 추적.
- 엔티티 그래프의 엣지는 **저장하지 않고**, 같은 `turn_id`를 공유하는 행들로 쿼리 시점에 계산.
- 인덱스: `(session_id)`, `(session_id, entity_type)`, `(session_id, value_hash)` — 누적 조회와
  "같은 값이 여러 턴에 걸쳐 나왔는지" 판정용.

---

## 불변식 · 규칙

- **원문 무저장.** `session_turns.text_hash`, `session_entities.value_hash` 모두 해시. 원본이 필요한
  경우는 토큰 볼트를 통해서만.
- `risk_score` 는 `accumulator` 가 조합 가중으로 갱신(예: NAME+RRN 상승, +ACCOUNT 추가 상승).
  하드 블록 임계값을 넘으면 파이프라인이 `block`.
- 세션 TTL 만료 시 해당 세션의 **토큰 볼트 레코드도 함께 정리**된다(볼트는 별도 `expires_at`을 갖지만
  `purge_expired()` 가 세션 만료와 정합을 맞춤). [`token-vault.md`](token-vault.md) 참조.
- 감사 로그(`log_events`)는 이 테이블로의 FK가 없으므로 세션 삭제 후에도 남는다.

## 데모 구현

`context/store.py` 의 InMemory 구현이 이 구조를 딕셔너리로 흉내낸다(TTL 스윕 포함). 컬럼 의미·불변식은
동일하게 지킨다. Redis/PostgreSQL 승격은 [dlp-server-architecture](../../architecture/dlp-server-architecture.md) §8.
