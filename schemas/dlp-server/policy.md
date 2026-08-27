# policy 스키마

목적 기반 동적 데이터 접근 제어(기능 f)의 정책 저장 구조. `(목적 × role × 엔티티 타입) → 조치`
매트릭스와 위험도 오버라이드를 버전 단위로 관리한다.

DDL 원본: [`postgres-schema.sql`](postgres-schema.sql) §3
설계 근거: [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §6-f

---

## 테이블

### `policy_versions`

| 컬럼 | 의미 |
|---|---|
| `policy_version_id` | SERIAL PK |
| `description`, `created_by`, `created_at` | 메타 |
| `is_active` | 활성 버전 플래그 |

- `UNIQUE (is_active) WHERE is_active = true` — **활성 버전은 항상 1개**.
- 정책 변경은 새 버전을 만들고 활성 플래그를 옮기는 방식(롤백 가능).

### `policy_rules`

`(purpose × role × entity_type) → action` 한 줄.

| 컬럼 | 의미 |
|---|---|
| `policy_version_id` | FK, CASCADE |
| `purpose` | TEXT → `purpose_ref`, **NULL = 와일드카드(`*`)** |
| `role` | VARCHAR(64), **NULL = 와일드카드** |
| `entity_type` | TEXT → `entity_type_ref`, **NULL = 와일드카드** |
| `action` | `action_type` (keep/mask/generalize/aggregate/tokenize/synthetic/redact/block) |
| `priority` | INT, 높을수록 우선 |

- 조회 인덱스: `(policy_version_id, purpose, role, entity_type)`.
- 매칭 시 구체 규칙 > 와일드카드, 동점이면 `priority` 큰 것.

### `policy_risk_overrides`

컨텍스트 조건이 맞으면 규칙 결과를 덮어쓰는 항목.

| 컬럼 | 의미 |
|---|---|
| `condition_expr` | 예: `"injection.hit"`, `"risk_score >= 0.8"` |
| `action` | 덮어쓸 조치 (보통 `block`) |
| `priority` | 기본 100 (일반 규칙보다 높게) |

---

## 규칙 · 불변식

- **`condition_expr` 는 앱의 화이트리스트 파서로만 평가한다.** 지원 형태는
  `injection.hit` 와 `risk_score >= N` 두 종류. **`eval()` 절대 금지.** 새 조건이 필요하면 파서에
  케이스를 추가한다.
- `entity_type` / `purpose` 목록은 `entity_type_ref` / `purpose_ref` seed 기준. 확정 시 행만 추가.
- `entity_type_ref.tier`(low/medium/high/critical)는 민감도 등급으로, 정책이 명시 규칙 없이
  tier 기반 기본값을 줄 때 사용할 수 있다.

## 초안 (YAML 표현)

DB 이전 단계에서는 아래 `policy.yaml` 이 같은 내용을 담는다. 컬럼 매핑: `rules[*]` → `policy_rules`,
`risk_overrides[*]` → `policy_risk_overrides`, `defaults` → 와일드카드 최저 우선 규칙.

```yaml
purposes: [customer_support, doc_summarize, code_help, data_analysis, fraud_investigation, unknown]
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

## 데모 구현

`policy/engine.py` + `policy/policy.yaml`. 경량 룰 평가기(~100줄). OPA/Rego, 관리자 CRUD API는 확장.
테스트는 `(purpose × role × entity)` 전 조합 매트릭스.
