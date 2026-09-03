# 동적 데이터 변환

> **구현 상태.** `transform/apply.py` 의 `transform_stage` 와 입력 [6] 배선 완료(#20 이후 확장).
> 8개 action 전부 핸들러가 있다: `keep` / `mask`(타입별) / `redact` / `tokenize` / `synthetic`
> (이름 풀 + 숫자 무작위) / `aggregate`(그룹 합계·평균 요약) / `generalize`(RRN → `<AGE:x대><SEX:y>`,
> 그 외 타입은 `mask` 폴백). 탐지([3])·멀티턴([4])도 배선돼 실 요청에서 `span_actions` 가 채워진다.
> 남은 작업은 §7.

**우선순위:** 필수
**한 줄 정의:** 정책 엔진(f)이 엔티티마다 결정한 조치를 실제로 실행해 전송 본문을 변환한다. 같은 PII 타입이라도 목적·role·위험도 조합에 따라 다른 전략이 런타임에 선택된다(정적 마스킹표가 아님).
**담당 모듈:** `transform/apply.py` (`tokenize` 는 기능 a 의 `transform/vault.py` 호출)

설계 의도·전체 맥락: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6-g
파이프라인 계약: [`pipeline-stage-contract.md`](pipeline-stage-contract.md) §4.1 (입력 [6])
관련 스펙: [`f_purpose-based-access-control.md`](f_purpose-based-access-control.md)(조치 결정), [`a_reversible-tokenization.md`](a_reversible-tokenization.md)(토큰 볼트)
색인: [`../spec-index.md`](../spec-index.md)

---

## 1. 입력 예시

스테이지 `transform_stage(ctx)` 는 `AnalysisContext` 하나를 받는다(`apply_transforms` 는 호환용
별칭). `span_actions` 는 f 가 채운다.

```jsonc
// AnalysisContext (요약)
{
  "direction": "input",
  "session_id": "7f3a…-uuid",
  "role": "agent_l1",
  "purpose": "customer_support",
  "turns": [
    {"role": "user", "text": "고객 연락처 010-1234-5678 로 안내 부탁해요"}
  ],
  "span_actions": [
    // (Span, action) — f 산출. action 은 TRANSFORM_ACTIONS 중 하나.
    [{"type": "PHONE", "value": "010-1234-5678", "start": 6, "end": 19}, "mask"]
  ]
}
```

- `block` action 은 f 가 이미 `ctx.blocked` 로 조기 차단하므로 여기 도달하지 않는다.
- `span_actions` 가 비어 있으면 스테이지는 아무것도 하지 않고 통과한다.

## 2. 출력 예시

스테이지는 `ctx.turns[*].text` 만 바꾼다. 파이프라인이 어댑터로 재조립해 `transform` 판정을 만든다.

```jsonc
// 변환 후 ctx.turns[0].text
"고객 연락처 010-****-5678 로 안내 부탁해요"
```

파이프라인이 `reason_obj` 에 요약을 싣는다(원문 금지):

```json
{
  "verdict": "transform",
  "transforms": [{"entity": "PHONE", "action": "mask"}],
  "entities_summary": [{"type": "PHONE", "confidence": 0.9, "masked_preview": "010-****-5678"}]
}
```

- `tokenize` 조치는 `<PII:PHONE:1>` 라벨로 치환되고 볼트에 매핑이 저장된다. 응답 경로에서
  인가된 요청자에 한해 복원된다([`a_reversible-tokenization.md`](a_reversible-tokenization.md)).

## 3. 판정 로직

이 모듈은 판정하지 않는다. f 의 결정을 집행만 한다.

### 3.1 조치별 실행 (`_render(ctx, span, action)`)

| action | 동작 |
|---|---|
| `keep` | 치환하지 않음 (원본 유지). `reason_obj.transforms` 에도 넣지 않음 |
| `mask` | 타입별 비가역 부분 마스킹 (§3.2) |
| `redact` | `[삭제됨]` 로 치환 |
| `tokenize` | `vault.tokenize(session_id, span.type, span.value, access_scope=…)` → `<PII:{TYPE}:{n}>` (§3.3) |
| `generalize` | **RRN 만 구현** — `<AGE:{연령대}대><SEX:{남\|여}>` (주민번호 앞 6자리 + 성별자리로 계산). 그 외 타입은 `mask` 로 폴백 + info 로그 |
| `aggregate` | **그룹 처리** — 개별 span 은 자리를 비우고, 같은 턴의 `aggregate` 대상 전부를 모아 텍스트 끝에 ` [집계: 합계 N / 평균 M / 건수 K]` 를 덧붙인다. 금액 파싱은 숫자·`.` 만 추출 |
| `synthetic` | `NAME` → 고정 이름 풀에서 무작위. 그 외(숫자 포함) → 자릿수 유지하고 숫자만 무작위. 숫자 없으면 `mask` 폴백 |

### 3.2 타입별 마스킹 (`_mask(entity_type, value)`)

| 타입 | 규칙 | 예 |
|---|---|---|
| `RRN` / `FOREIGN_RRN` | `-` 뒤 전부 `*` (구분자 없으면 앞 6자리만 남김) | `880101-1234567` → `880101-*******` |
| `CARD` / `ACCOUNT` | 뒤 4자리만 남기고 숫자 `*` (구분자 보존) | `4111-1111-1111-1111` → `****-****-****-1111` |
| `PHONE` | 3파트면 가운데 파트 `*` (아니면 카드 규칙) | `010-1234-5678` → `010-****-5678` |
| `EMAIL` | 로컬파트 첫 글자 + `***` + `@도메인` | `test@x.co.kr` → `t***@x.co.kr` |
| `NAME` | 가운데 글자 `*` (2자면 `홍*`, 1자면 `*`) | `홍길동` → `홍*동` |
| 그 외 | 첫 글자 + 가운데 `*` + 끝 글자 (`_mask_middle`) | `1000000` → `1*****0` |

### 3.3 토큰화 시 `access_scope` 조립 (`_access_scope(ctx)`)

정책 매트릭스는 *조치* 만 주고 *복원 권한* 은 주지 않으므로 g 가 조립한다.

```python
{"roles": [ctx.role] if ctx.role else ["*"], "purposes": [ctx.purpose] if ctx.purpose else ["*"]}
```

- 토큰화한 role·목적이 응답 경로에서 돌아올 때만 복원되고, 이후 턴에서 목적이 바뀌면 복원이 차단된다.
- `role` 헤더가 없으면 `["*"]` 로 완화(로컬 개발 편의). 실배포는 role 헤더 필수.
- 정책이 복원 범위를 직접 지정하는 확장(`policy_rules` 에 `restore_roles` / `restore_purposes`
  컬럼, 직무분리·목적제한 설정화)은 §7.

### 3.4 치환 순서

- 조치는 **이번 요청의 마지막 `user` 턴** 텍스트에 적용한다.
- 한 턴 안에서 span 을 `start` **역순(뒤→앞)** 으로 치환한다. 앞 span 의 offset 이 밀리지 않도록.

### 3.5 fail 방향

**목표: fail-closed** — 변환에 실패한 원문이 그대로 나가면 안 된다.

- `tokenize` 오류(볼트 예외 · `DLP_VAULT__KEY` 미설정 등) → `_tokenize` 가 예외를 잡아
  `[삭제됨]`(redact)로 대체한다. 원문은 안 나가지만 요청 자체는 계속 진행된다.
- 알 수 없는 action → `keep`(원본 유지) + warning 로그.
- 그 외 스테이지 예외(마스킹 함수 버그 등)는 `transform_stage` 에 자체 `try/except` 가 아직 없어
  `pipeline.analyze()` 상위 처리기로 전파된다 → `fail_action`(기본 `block`)으로 잡힌다. 단
  `DLP_FAIL_ACTION=allow` 시연 스위치에서는 원문이 통과할 수 있다.

> **알려진 갭:** `transform_stage` 에 스테이지 레벨 fail-closed(`try/except → ctx.blocked`)가 없다.
> 마스킹 버그가 상위 `fail_action` 에 의존하는 구조라, `allow` 스위치와 조합되면 유출 경로가 된다.
> 스테이지에서 직접 `ctx.blocked` + `block_reason={"type": "transform", "note": "stage_error"}` 를
> 세팅하도록 보강 예정(§7).

## 4. 파라미터 · 설정값

| 키 | 기본 | 의미 |
|---|---|---|
| `DLP_VAULT__KEY` (`config.vault.key`) | *(없음, 필수)* | `tokenize` 조치가 볼트 `cipher_value` 를 AES-GCM 하는 키. 미설정 시 `tokenize` 는 `RuntimeError` → fail-closed |
| `vault_ttl_sec` | `1800` | 토큰 볼트 레코드 수명(초). `tokenize` 가 만드는 행에 적용 |

마스킹 포맷·조치 전략은 코드 상수. 정책/설정화는 이후 확장.

## 5. 엣지 케이스

| 상황 | 동작 |
|---|---|
| `span_actions` 비어 있음 | 스테이지 무동작 통과 |
| `direction == "output"` | 무동작 통과 (변환은 입력 경로 전용) |
| `keep` 조치 | 텍스트 무변경 |
| 한 턴에 span 여럿 | `start` 역순 치환 → 길이가 바뀌어도 앞 span offset 유지 |
| `user` 턴 없음 | 무동작 통과 |
| 과거 `user` 턴 | 건드리지 않음 (마지막 user 턴만) |
| 알 수 없는 `entity_type` 마스킹 | 첫/끝 글자만 남기고 가운데 `*` (`_mask_middle`) |
| `tokenize` 인데 `DLP_VAULT__KEY` 없음 | `_tokenize` 가 예외를 잡아 `[삭제됨]`(redact)로 대체. 원문 미유출, 요청은 진행 (§3.5) |
| 같은 원본 값 span 여럿 | 각각 치환. `tokenize` 는 볼트가 `value_hash` 로 같은 라벨 재사용(결정론) |

## 6. 테스트 케이스표

`tests/test_transform_apply.py` — `_mask` · `apply_transforms` 는 DB 불필요(`tokenize` 는 monkeypatch),
실 볼트 왕복만 `db` fixture.

| 그룹 | 케이스 |
|---|---|
| 타입별 마스킹 | RRN/CARD/PHONE/EMAIL/NAME + 규칙 없는 타입, 구분자 유무 |
| 조치 실행 | `keep` 무변경, `mask`+`redact` 혼합, 뒤→앞 치환(길이 변화), 마지막 user 턴만 적용 |
| `tokenize` | `vault.tokenize` 호출 시 `access_scope` 가 요청 role·목적 형태, role 없으면 `["*"]` |
| fail-closed | 볼트 예외 → `ctx.blocked` + `block_reason.note == "stage_error"` |
| 볼트 왕복 (DB) | `tokenize` → 같은 role·목적 복원 O, 다른 role 거부 |
| 파이프라인 배선 | 합성 `span_actions` → `Decision.transform`, `transformed_body` 에 원문 없음, `reason_obj.transforms`/`entities_summary` 채워짐 |

## 7. 현재 구현 상태 · 남은 작업

- **동작 중:** `keep` / `mask`(타입별 §3.2) / `redact` / `tokenize`(`access_scope` = 요청 맥락 §3.3) /
  `synthetic` / `aggregate`(그룹 요약) / `generalize`(RRN 만). span 은 마지막 `user` 턴 대상.
  `_INPUT_STAGES` [6] 배선. 탐지([3])·멀티턴([4]) 배선돼 실 `span_actions` 로 관통된다.

**남은 작업:**

- **`transform_stage` fail-closed 보강** — §3.5 갱. 스테이지 레벨 `try/except → ctx.blocked` 부재.
- `generalize` — RRN 외 타입(나이·주소 등) 규칙. 현재는 `mask` 폴백.
- `access_scope` 정책화 — `policy_rules` 에 `restore_roles` / `restore_purposes`(nullable) 컬럼 추가.
- span↔turn 매핑 — 현재 "마지막 `user` 턴" 가정. 멀티턴 요청의 다른 턴 PII 는 미처리 → `Span.turn_index` 도입(탐지 모듈과 협의).
- `reason_obj.transforms` 상세 — 현재 `{entity, action}` 만. `token_label` / `masked_preview` 를 담으려면 `AnalysisContext.transforms` 필드.
- 배치 토큰화 `tokenize_many` — 요청당 PII 다수일 때 1 트랜잭션.
- 형식 보존 암호화(FPE) — 카드·전화처럼 형식이 고정된 데이터의 마스킹 대안.
