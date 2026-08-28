# 가역적 토큰화

> **구현 상태.** 볼트 레포지토리 `transform/vault.py` 는 구현·머지됨(dlp-server #12).
> 파이프라인 배선(입력 [6] 토큰화 / 출력 [2] detokenize)은 후속 — 기능 g·c-output 이 스테이지로
> 연결한다. 이 문서는 완성 기준 계약을 서술한다.

**우선순위:** 필수
**한 줄 정의:** 외부 LLM 전송 전 PII를 결정론적 토큰으로 치환하고, 응답 수신 후 인가된 요청자에 한해 원본으로 복원한다.
**담당 모듈:** `transform/vault.py`(토큰 볼트 레포지토리), `transform/apply.py`(`tokenize` 조치 실행), Output Guard 경로의 detokenize

설계 의도·전체 맥락: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6-a, §7.2
파이프라인 계약: [`pipeline-stage-contract.md`](pipeline-stage-contract.md) §4 (입력 [6] / 출력 [2])
관련 스키마: [`../../schemas/dlp-server/token-vault.md`](../../schemas/dlp-server/token-vault.md)
색인: [`../spec-index.md`](../spec-index.md)

---

## 1. 입력 예시

토큰 볼트는 스테이지가 아니라 다른 스테이지가 호출하는 레포지토리다. 진입점 둘:

**입력 경로 — 토큰화.** 정책 엔진(f)이 엔티티에 `tokenize` 조치를 결정하면 동적 변환(g)이 호출한다.

```python
tokenize(
    session_id="7f3a…-uuid",
    entity_type="RRN",              # 12종 화이트리스트, 밖이면 UNKNOWN 폴백
    value="880101-1234567",
    access_scope={"roles": ["agent_l1"], "purposes": ["customer_support"]},
) -> "<PII:RRN:1>"
```

**출력 경로 — 복원.** detokenize 스테이지가 응답 본문에서 찾은 라벨마다 호출한다.

```python
detokenize_text(
    session_id="7f3a…-uuid",
    text="확인된 주민번호는 <PII:RRN:1> 입니다.",
    role="agent_l1",
    purpose="customer_support",
) -> "확인된 주민번호는 880101-1234567 입니다."
```

## 2. 출력 예시

- **토큰 라벨:** `<PII:{ENTITY_TYPE}:{n}>` — 타입을 포함하는 자기서술 포맷. `n` 은
  `(session_id, entity_type)` 별로 1부터 증가.
- **결정론:** 같은 세션의 같은 원본 값은 재호출해도 항상 같은 라벨(새 행을 만들지 않는다).
- **변환 근거** — 동적 변환(g)이 `reason_obj.transforms` 에 기록:

  ```json
  {"entity": "RRN", "action": "tokenize", "token_label": "<PII:RRN:1>"}
  ```

- **복원 성공:** 원문 문자열. **실패**(미인가·만료·미존재): 라벨을 그대로 반환(치환하지 않음) +
  `token_vault_access_log` 에 `granted=false` 기록.

## 3. 판정 로직

### 3.1 토큰화 (`tokenize`)

1. `entity_type` 정규화 — 대문자·strip. 12종(`RRN FOREIGN_RRN CARD ACCOUNT PASSPORT DRIVER
   CREDIT_INFO PHONE EMAIL BIZNO AMOUNT NAME`) 밖이면 `UNKNOWN`.
2. `value_hash` = `sha256(session_id ‖ 0x1f ‖ entity_type ‖ 0x1f ‖ value)` 앞 32 hex. 로깅하지 않는다.
3. 같은 `(session_id, value_hash)` 의 미revoke 행이 있으면 그 라벨을 반환(재사용).
4. 없으면 `(session_id|entity_type)` 로 `pg_advisory_xact_lock` → **revoked 포함** `count(*)` →
   라벨 번호 `n = count + 1`.
5. AES-GCM 암호화(`nonce(12B) ‖ ciphertext ‖ tag(16B)`) 후
   `INSERT … ON CONFLICT (session_id, value_hash) WHERE revoked_at IS NULL DO NOTHING`.
6. 경합으로 INSERT 가 스킵되면 먼저 커밋된 행의 라벨을 재조회해 반환.

`access_scope` 미지정 → `{"roles": [], "purposes": []}` (아무도 복원할 수 없는 토큰).

### 3.2 복원 (`detokenize` / `detokenize_text`)

1. `(session_id, token_label)` 미revoke 행 조회. 없으면 거부 `unknown_or_revoked`.
2. `expires_at <= now()` → 거부 `expired`.
3. `access_scope` 대조 — `roles` 에 `"*"` 또는 요청 `role` 포함 **그리고** `purposes` 에 `"*"`
   또는 요청 `purpose` 포함해야 통과. 아니면 거부 `role_not_in_scope` / `purpose_not_in_scope`.
4. 통과 시 복호해 원문 반환.

모든 시도(성공·실패)를 `token_vault_access_log` 에 1건 INSERT 한다. `detokenize_text` 는
`<PII:[A-Z_]+:\d+>` 로 라벨을 훑어 각각 `detokenize` 하고, 인가 실패한 라벨은 원문 그대로 둔다.

### 3.3 fail 방향

- `tokenize` 의 DB 오류·키 미설정 → **예외 전파(raise).** 토큰화 실패 시 원문이 그대로 나가면
  안 되므로 파이프라인이 fail-closed(`block`)로 처리하게 한다.
- `detokenize` 의 DB 오류 → **`None`**(토큰 유지). 복원 실패는 라벨 노출일 뿐 유출이 아니다.

## 4. 파라미터 · 설정값

| 키 | 기본 | 의미 |
|---|---|---|
| `DLP_VAULT__KEY` (`config.vault.key`) | *(없음, 필수)* | base64 인코딩 32바이트 AES-GCM 키. 미설정·길이 불일치 시 `RuntimeError`. 저장소에는 암호문만 두고 키는 설정에만 |
| `vault_ttl_sec` (`config.vault_ttl_sec`) | `1800` | 토큰 볼트 레코드 수명(초). 세션 TTL(`session_ttl_sec`)과 분리 |
| `ENTITY_TYPES` (모듈 상수) | 12종 | 허용 엔티티 타입. 밖의 값은 `UNKNOWN` 폴백 |

KMS 키 관리·형식 보존 암호화(FPE)는 이후 확장이다.

## 5. 엣지 케이스

| 상황 | 동작 |
|---|---|
| 미존재·revoke 된 라벨 복원 시도 | `None` + access_log `granted=false, denied_reason="unknown_or_revoked"` (`token_id` = NIL UUID) |
| 만료 토큰 복원 | `None` + `denied_reason="expired"` (`purge_expired` 전이라도 `expires_at` 검사로 즉시 차단) |
| 기본 `access_scope` (`{"roles":[],"purposes":[]}`) | 누구도 복원 불가 |
| 세션 격리 | 값이 같아도 세션마다 별도 행·라벨. 다른 세션 라벨로 복원 시도 → `None` |
| revoke 후 카운터 | `count(*)` 가 revoked 행을 포함 → 라벨 번호를 재사용하지 않음(옛 응답의 `<PII:RRN:1>` 오복원 방지) |
| 동시 tokenize (같은 값) | advisory xact lock + `ON CONFLICT DO NOTHING` + 경합 재조회 → 단일 행·단일 라벨 |
| 알 수 없는 `entity_type` | `UNKNOWN` 폴백 + warning 로그. 토큰화·복원은 정상 동작 |
| 키 미설정 상태에서 `tokenize` | `RuntimeError` 전파. 같은 상태의 `detokenize` 는 `None` |
| 하드 삭제된 토큰의 access_log | dangling `token_id` 유지 (감사 우선의 의도된 결과) |

## 6. 테스트 케이스표

`tests/test_vault.py` — PostgreSQL 필요, 17개.

| 그룹 | 케이스 |
|---|---|
| 왕복·결정론·카운터 | `roundtrip`, `deterministic_reuse`(행 1개), `session_isolation`, `counter_per_session_and_type`, `unknown_entity_type_falls_back` |
| AES-GCM | `cipher_value_is_encrypted`(원문 미포함, 길이 = 12 + ct + 16), `missing_key_fails_closed`(tokenize raise / detokenize None) |
| access_scope 게이팅 | `default_scope_denies_everyone`, `wildcard_scope_allows_any`, `specific_scope_matches_only_listed`, `every_attempt_is_logged`, `access_log_has_no_plaintext` |
| 만료·purge | `expired_token_denies`, `purge_expired_revokes_then_hard_deletes` |
| detokenize_text | `detokenize_text_restores_only_authorized`(미인가 라벨은 원문 유지) |
| 동시성 | `concurrent_same_value_single_row`(100 호출 → 1행), `concurrent_distinct_values_unique_labels`(20 값 → 1~20 라벨) |

## 7. 열린 항목

- `access_scope` 를 볼트 레코드에 고정할지, 복원 시점에 정책 엔진(f)이 재평가할지 — 미정
  (`_resolve_scope` 가 교체 지점). [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §12.
- 형식 보존 암호화(FPE) — 카드번호·전화번호처럼 형식이 고정된 데이터에 한해 이후 확장.
- KMS 연동 — 현재는 앱레벨 AES-GCM + 환경변수 키.
