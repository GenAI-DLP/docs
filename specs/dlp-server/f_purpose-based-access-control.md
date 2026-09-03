# 목적 기반 동적 데이터 접근 제어

> **구현 상태.** `purpose/` · `policy/` 모듈과 입력 [5] 스테이지 구현·배선됨(dlp-server #17).
> 탐지([3])·멀티턴([4])도 배선 완료 → 실 요청에서 `span_actions` 가 채워지고 `risk_score` 도
> 세션 누적을 반영한다. 조치 실행·`access_scope` 조립은 기능 g. 남은 작업은 §7.

**우선순위:** 필수
**한 줄 정의:** 요청 목적·요청자 role·엔티티 타입·누적 위험도를 조합해 탐지된 엔티티마다 조치(keep/mask/generalize/aggregate/tokenize/synthetic/redact/block)를 결정한다.
**담당 모듈:** `purpose/role_resolver.py`, `purpose/classifier.py`, `policy/engine.py`, `policy/policy.yaml`

설계 의도·전체 맥락: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6-f
파이프라인 계약: [`pipeline-stage-contract.md`](pipeline-stage-contract.md) §4.1 (입력 [5])
관련 스펙: [`g_dynamic-data-transformation.md`](g_dynamic-data-transformation.md)(조치 실행)
관련 스키마: [`../../schemas/dlp-server/policy.md`](../../schemas/dlp-server/policy.md)
색인: [`../spec-index.md`](../spec-index.md)

---

## 1. 입력 예시

스테이지 `purpose_policy_stage(ctx)` 는 `AnalysisContext` 하나를 받는다. role 은 스테이지가
아니라 **파이프라인이** `_analyze_input` / `_analyze_output` 에서 `role_resolver.resolve(headers)`
로 미리 채운다(raw 헤더를 컨텍스트에 싣지 않는다).

```jsonc
// AnalysisContext (요약)
{
  "direction": "input",
  "role": "agent_l1",                 // 파이프라인이 헤더에서 resolve
  "turns": [
    {"role": "user", "text": "이 고객 민원 건 요약해줘. 연락처 010-1234-5678"}
  ],
  "new_turn_spans": [                 // 기능 b 산출 (pii_detect_stage 가 채움)
    {"type": "PHONE", "value": "010-1234-5678", "start": 20, "end": 33, "confidence": 0.9, "source": "regex"}
  ],
  "risk_score": 0.0,                  // 기능 e 산출 (multiturn_stage 가 세션 누적으로 갱신)
  "injection": {"hit": false, "score": 0.0, "pattern": null}
}
```

## 2. 출력 예시

스테이지는 `ctx` 필드만 갱신한다. `Decision` 은 파이프라인이 만든다.

```jsonc
{
  "purpose": "customer_support",
  "purpose_confidence": 0.8,
  "span_actions": [
    // (Span, action) 쌍. action 은 TRANSFORM_ACTIONS 중 하나.
    [{"type": "PHONE", "value": "010-1234-5678", "...": "..."}, "mask"]
  ]
}
```

- 어떤 span 의 action 이 `block` 이면 스테이지가 `ctx.blocked = true`,
  `ctx.block_reason = {"type": "policy", "entity": "<타입>", "purpose": "<목적>"}` 를 세팅한다
  → 파이프라인이 요청 전체를 `block`.
- `span_actions` 의 실제 실행(치환·토큰화)은 기능 g(`transform/apply.py`)가 한다. 이 모듈은
  **결정까지만** 책임진다 (`guardrail/injection.py` 와 같은 규약).

## 3. 판정 로직

### 3.1 role 해석 (`role_resolver.resolve(headers)`)

1. `X-Corp-User-Role` 헤더 값(대소문자 무시, strip) → 그대로 role.
2. 없으면 `X-Corp-User-Id` → `_ROLE_MAP`(자리표시, 사내 IAM 연동 지점) 조회.
3. 그것도 없으면 `None` — 정책은 role 와일드카드 규칙만 매칭한다.

### 3.2 목적 분류 (`classifier.classify(text)`)

- 대상 텍스트 = **이번 요청의 마지막 `user` 턴** (Input Guard 와 같은 기준).
- `RuleClassifier` — 키워드 정규식을 위에서부터 훑어 **첫 매칭** purpose 를 채택(confidence 0.8).
  카테고리: `fraud_investigation` → `data_analysis` → `code_help` → `doc_summarize` →
  `customer_support` 순으로 먼저 평가(구체·민감한 것 우선).
- 미매칭 → `("unknown", 0.0)`.
- `config.purpose.backend == "llm"` 이면 `_classifier` 를 모델 구현으로 교체(타임아웃 + 규칙
  fallback + 세션 캐시는 그쪽 구현). 현재 기본은 규칙.

### 3.3 조치 결정 (`engine.decide(purpose, role, entity_type, *, risk_score, injection_hit)`)

활성 정책 버전(`policy_versions.is_active`)의 규칙을 PostgreSQL 에서 읽어 **프로세스당 1회 캐시**한다.

1. **risk_override 먼저.** `policy_risk_overrides` 를 순회해 `condition_expr` 가 참인 것 중
   `priority` 가 가장 큰 항목의 `action` 을 채택하고 종료.
2. **매트릭스 매칭.** `policy_rules` 에서 `(purpose|NULL) ∧ (role|NULL) ∧ (entity_type|NULL)`
   가 맞는 규칙을 모아, **구체성**(NULL 아닌 매칭 컬럼 수)이 큰 것 → 동점이면 `priority` 큰 것.
3. 매칭 규칙이 하나도 없으면 `defaults` 규칙(시드가 `purpose=role=entity=NULL, priority=-1` 로 삽입).
4. 활성 정책 버전 자체가 없으면 `tokenize` 로 폴백(가역적이라 보수적).

### 3.4 조건식 파서 (`eval_condition`)

**화이트리스트만.** `eval()` 을 절대 쓰지 않는다.

| 지원 형태 | 의미 |
|---|---|
| `injection.hit` | `ctx.injection.hit` 이 참 |
| `risk_score >= N` | `N` 은 `0`~`1` 리터럴(`0`, `0.8`, `1.0`) |

그 외 문자열(`risk_score > 0.8`, `1 == 1`, `__import__(...)`, 빈 문자열 등)은 **예외 없이 `False`**
+ warning 로그. 새 조건이 필요하면 파서에 케이스를 추가한다.

### 3.5 fail 방향

- `purpose_policy_stage` 내부 오류(분류·정책 조회·DB) → **fail-closed.** `ctx.blocked = true`,
  `ctx.block_reason = {"type": "policy", "note": "stage_error"}`. PII 보호 경로이므로 통과시키지 않는다.
  (Input Guard 는 fail-open 이지만 그건 PII 보호를 b~g 가 따로 하기 때문 — f 는 그 보호 자체다.)

## 4. 파라미터 · 설정값

| 키 | 기본 | 의미 |
|---|---|---|
| `config.purpose.backend` (`DLP_PURPOSE__BACKEND`) | `rule` | `rule` \| `llm`. `llm` 이면 `_classifier` 를 모델 구현으로 교체 |
| `config.purpose.llm_timeout_sec` | `1.5` | LLM 분류기 타임아웃(초). 규칙 fallback 과 함께 씀 |
| `config.risk.hard_block` (`DLP_RISK__HARD_BLOCK`) | `0.6` | 파이프라인 내장 위험도 차단 임계. 정책 시드의 `risk_score >= 0.8` 오버라이드와 값이 다른 건 의도 — 내장 컷오프가 더 낮게(먼저) 걸린다. `multiturn.combo_cap` 과 같은 값이어야 함(e 스펙 §3.3) |
| `app/policy/policy.yaml` | — | `(목적×role×엔티티)→조치` 매트릭스 + `defaults` + `risk_overrides` 시드 |

정책 데이터는 `scripts/seed_policy.py` 가 `policy.yaml` → `policy_versions` / `policy_rules` /
`policy_risk_overrides` 로 적재한다(활성 버전 1개, `--reset` 으로 재적재). 런타임은 DB 에서 읽는다.
관리자 CRUD API·OPA 연동은 이후 확장.

**정책 초안 (`policy.yaml`):**

```yaml
rules:
  - {purpose: doc_summarize,       role: "*",      entity: "*",    action: tokenize}
  - {purpose: doc_summarize,       role: "*",      entity: CARD,   action: block}
  - {purpose: data_analysis,       role: "*",      entity: RRN,    action: generalize}
  - {purpose: data_analysis,       role: "*",      entity: AMOUNT, action: aggregate}
  - {purpose: code_help,           role: "*",      entity: "*",    action: block}
  - {purpose: customer_support,    role: agent_l1, entity: PHONE,  action: mask}
  - {purpose: fraud_investigation, role: agent_l2, entity: PHONE,  action: keep}
  - {purpose: unknown,             role: "*",      entity: "*",    action: tokenize}
defaults: {action: tokenize}
risk_overrides:
  - {when: "injection.hit",     action: block}
  - {when: "risk_score >= 0.8", action: block}
```

## 5. 엣지 케이스

| 상황 | 동작 |
|---|---|
| role 헤더 없음 | `role = None`. 구체 role 규칙은 매칭 안 되고 role 와일드카드 규칙만 적용 |
| 목적 키워드 미매칭 | `purpose = "unknown"` (confidence 0.0). `unknown/*/*` 규칙 → `tokenize` |
| `user` 턴이 없음 | 빈 문자열로 분류 → `unknown` |
| 같은 구간에 규칙 여럿 (와일드카드 vs 구체) | 구체성 큰 규칙 우선. 예: `doc_summarize/*/CARD`(block) 가 `doc_summarize/*/*`(tokenize) 를 이김 |
| 매칭 규칙 없음 | `defaults` 규칙(`tokenize`). 그것도 없으면 `_FALLBACK_ACTION = tokenize` + warning |
| 활성 정책 버전 없음 (시드 누락) | 빈 규칙셋 → 모든 엔티티 `tokenize` 폴백. 서버는 뜨지만 정책 미적용 |
| `injection.hit` 또는 `risk_score >= 0.8` | risk_override 가 규칙 결과를 덮어 `block` (파이프라인 내장 차단과 중복 — 방어 깊이로 둘 다 유지) |
| 알 수 없는 `condition_expr` | `False` (발동 안 함), 예외 없음, warning 로그 |
| 스테이지 내부 예외 | fail-closed — `ctx.blocked = true`, `block_reason.note = "stage_error"` |
| `direction == "output"` | 스테이지는 아무것도 하지 않고 통과 (목적·정책은 입력 경로 전용) |

## 6. 테스트 케이스표

`tests/test_purpose.py` (DB 불필요) + `tests/test_policy.py` (`pick_action`·`eval_condition` 은
DB 불필요, 시드→`decide` 는 PostgreSQL 필요).

| 그룹 | 케이스 |
|---|---|
| 목적 분류 | 5개 목적별 한/영 키워드 hit, 미매칭 → `unknown`, 첫 매칭 규칙 우선 |
| role 해석 | `X-Corp-User-Role` 채택, 대소문자 무시, 헤더 없음 → `None`, 공백 role → `None` |
| 매트릭스 | `policy.yaml` 규칙별 기대 action 전수, 구체 > 와일드카드, `priority` 타이브레이크, 빈 규칙셋 → `tokenize` |
| risk_override | `injection.hit` / `risk_score >= 0.8` 가 규칙을 덮음, 임계 미만이면 안 덮음 |
| 조건 파서 | `injection.hit` / `risk_score >= N` 정상, 악의·미지원 문자열 → `False` (예외 없음) |
| 시드 (DB) | `seed()` → `decide()` 가 매트릭스대로, 재시드 시 활성 버전 1개 유지 |
| 파이프라인 배선 | `purpose_policy_stage` 가 `span_actions` 채움, `block` action → `Decision.block` |

## 7. 현재 구현 상태 · 남은 작업

**구현됨 (dlp-server #17):** `role_resolver` · `RuleClassifier` · `policy/engine.py`(`decide` +
화이트리스트 조건 파서) · `scripts/seed_policy.py` · 입력 [5] 스테이지 배선. `tests/test_purpose.py`
+ `tests/test_policy.py`.

**남은 작업:**

- **`access_scope` 정책화** — `tokenize` 복원 범위는 g 가 요청 맥락으로 조립
  ([`g_dynamic-data-transformation.md`](g_dynamic-data-transformation.md) §3.3). 정책이 직접 지정하려면
  `policy_rules` 에 `restore_roles` / `restore_purposes` 컬럼 추가(직무분리·목적제한 설정화).
- **LLM 분류기** — `config.purpose.backend == "llm"` seam 만 있고 규칙이 기본 구현. 모델 백엔드는 후속.
- **복원 시점 정책 재평가** — 볼트 `_resolve_scope` 를 정책 엔진 재평가로 바꿀지 미정
  ([`a_reversible-tokenization.md`](a_reversible-tokenization.md) §7, 아키텍처 §12).
- **정책 관리 UI** — 현재는 `policy.yaml` + 시드 스크립트. 관리자 CRUD·버전 롤백 API 는 확장.

**완료된 항목:** 탐지([3])·멀티턴([4]) 배선 → `span_actions`·`risk_score` 실 데이터로 채워짐.
`AnalysisContext` 의 `purpose` / `purpose_confidence` / `span_actions` 는
[`pipeline-stage-contract.md`](pipeline-stage-contract.md) §3 표에 반영됨.
