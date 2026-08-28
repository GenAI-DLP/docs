# token-vault 스키마

가역적 토큰화(기능 a)의 토큰 ↔ 원문 매핑과 복원 인가 정보. **볼트 계층**이며 세션과 수명을 분리해
자체 `expires_at`으로만 관리한다.

DDL 원본: [`postgres-schema.sql`](postgres-schema.sql) §2
설계 근거: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6-a, §7.2

---

## 테이블

### `token_vault`

| 컬럼 | 타입 | 의미 |
|---|---|---|
| `token_id` | UUID PK | |
| `session_id` | UUID (**FK 없음**, 값 컬럼) | 세션 스코프. `sessions` 를 참조하지 않아 세션 CASCADE 삭제의 영향을 받지 않음 |
| `entity_type` | TEXT → `entity_type_ref` | RRN / CARD / PHONE ... |
| `token_label` | VARCHAR(64) | 응답에 노출되는 문자열. 예: `<PII:RRN:1>`. 타입을 포함하는 자체 서술 포맷 |
| `cipher_value` | BYTEA | 앱레벨 AES-GCM 암호문(KMS 키). 저장소에 키를 두지 않음 |
| `value_hash` | VARCHAR(64) | 결정론적 재사용 매칭용 해시 |
| `access_scope` | JSONB | `{"roles": [...], "purposes": [...]}` — 복원 허용 조건 |
| `created_at` / `expires_at` | TIMESTAMPTZ | 생성·만료 |
| `revoked_at` | TIMESTAMPTZ NULL | soft delete. 세팅되면 즉시 복원 차단 |

### `token_vault_access_log`

복원(detokenize) 시도 이력. **감사 계층** — 운영·볼트 테이블로의 FK 없음.

| 컬럼 | 의미 |
|---|---|
| `token_id`, `session_id` | 값 컬럼(FK 없음). `session_id` 는 조회 편의 denormalize |
| `token_label` | 어떤 토큰을 복원하려 했는지 |
| `requested_role`, `requested_purpose` | 요청자 속성 |
| `granted` | `access_scope` 통과 여부 |
| `denied_reason` | 거부 사유 |
| `accessed_at` | 시각 |

---

## 인덱스 · 규칙

- **입력 경로(토큰 재사용):** `UNIQUE (session_id, value_hash) WHERE revoked_at IS NULL`.
  같은 세션에서 같은 원본 값은 항상 같은 토큰 라벨을 받는다.
- **출력 경로(복원 조회):** `(session_id, token_label) WHERE revoked_at IS NULL`.
  응답 본문에서 찾은 `<PII:...>` 라벨로 원문을 역조회.
- **만료 조회:** `(expires_at) WHERE revoked_at IS NULL`.

## 수명 주기

1. 입력 파이프라인이 토큰화 → `token_vault` 행 생성(암호문 + `access_scope`).
2. 출력 파이프라인의 detokenize → `access_scope` 대 요청자 role·purpose 대조.
   - 통과: 원문 복원, `token_vault_access_log(granted=true)` 기록.
   - 실패: 토큰 유지, `granted=false` + `denied_reason` 기록. 판정 `reason`에도 남김.
3. `purge_expired()`:
   - `expires_at < now()` → `revoked_at = now()` (soft revoke, 즉시 복원 차단).
   - `revoked_at < now() - 1 day` → 하드 삭제(`cipher_value` 완전 파기).
   - 세션 만료 삭제와 정합을 맞춰 개인정보 보관을 최소화.

## 열린 항목

- `access_scope` 를 볼트 레코드에 고정할지, 복원 시점에 정책 엔진이 재평가할지 — 미정
  ([dlp-server-architecture](../../architecture/dlp-server-architecture.md) §12).
- 하드 삭제 후 `token_vault_access_log` 는 dangling `token_id` 를 갖는다(감사 우선의 의도된 결과).
  필요 시 access log에 `entity_type` 등 스냅샷 컬럼 추가.

## 데모 구현

`transform/vault.py` 가 이 스키마의 `token_vault` / `token_vault_access_log` 에 PostgreSQL로 직접
읽고 쓴다. `tokenize(session_id, type, value)` 는 해시 기반 결정론적이며, 같은 세션의 같은 값
재사용은 `UNIQUE (session_id, value_hash) WHERE revoked_at IS NULL` 로 강제한다.
`cipher_value` 는 dlp-server가 INSERT 전 앱레벨 AES-GCM으로 암호화한 값이고(키는 설정에만,
DB는 암호문만 본다), 복원은 `detokenize` 가 복호한다. `access_scope` 게이팅과 `denied_reason`
기록은 스키마 그대로 유지하며, 복원 시도(성공·실패)는 매번 `token_vault_access_log` 에 INSERT
한다. 수명·정합 규칙은 위 "수명 주기"·"인덱스" 절을 따른다. 세션 스토어(인메모리/PostgreSQL)는
미정이나 볼트와 FK가 없어 무관하다. KMS 연동·FPE는 이후 확장이다.
