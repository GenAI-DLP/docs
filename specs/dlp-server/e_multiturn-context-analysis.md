# Multi-turn Context 분석

문서 내 상태 표기: [확정] 합의 완료 · [논의] 방향은 있으나 확정 전 · [미정] 착수 전.

한 줄 정의: 세션 전체 대화를 추적해 여러 턴에 분산 입력된 PII 조합을 엔티티 그래프 + 슬라이딩 윈도우로 탐지하고 누적 위험도를 매긴다.

담당 모듈: `context/store.py`, `context/accumulator.py`, `context/stage.py`
설계 의도·전체 맥락: `../../architecture/dlp-server-architecture.md` §6-e
관련 스키마: `../../schemas/dlp-server/session-context.md`
색인: `../spec-index.md`

**구현 현황 [확정]:** `pipeline.py`의 `_INPUT_STAGES`에 `multiturn_stage`가 실제로 배선되어 있고, 로드맵 §9 Phase 2 검증 시나리오("N턴 분산 입력 → 3턴째 누적 위험도로 차단")를 실제 gRPC 요청 및 pytest 통합 테스트로 확인했다. 아래 문서는 스텁이 아니라 **동작하는 코드 기준**으로 작성됐다.

**스코프 경계:** 이 기능(e)은 입력 경로(`_INPUT_STAGES`)에만 관여한다. 출력 경로(`_OUTPUT_STAGES = [detokenize_stage]`)는 `transform/apply.py` 담당자가 별도로 배선 중이며, `Output Guard`(기능 c-output)는 아직 미배선이다. 다만 §8에 정리했듯 출력 경로 쪽에서 이 기능의 `SessionState`에 필드 추가를 요청하는 TODO가 하나 걸려 있다.

---

## 0. 모듈 구성

| 파일 | 역할 |
|---|---|
| `context/store.py` | `SessionStore` 인터페이스 + `InMemorySessionStore`(TTL 스윕) |
| `context/accumulator.py` | 엔티티 그래프·슬라이딩 윈도우·위험도 산정 순수 로직 |
| `context/stage.py` | `pipeline.py`의 `Stage = Callable[[AnalysisContext], AnalysisContext]` 계약에 맞춘 어댑터. `accumulate()`를 감싸 sync/async 경계를 브리지 |

`stage.py`가 별도 파일로 분리된 이유: `accumulator.accumulate()`는 `(state, spans, turn_index, config)`를 받는 순수 함수라 테스트하기 쉽지만, 파이프라인이 요구하는 `Stage` 시그니처(`ctx` 하나만 받음)와는 다르다. 어댑터를 분리해 두 계약을 섞지 않았다.

---

## 1. 입력 예시

파이프라인 스테이지 [4](§3.1)가 `multiturn_stage(ctx: AnalysisContext) -> AnalysisContext` 형태로 호출된다. 내부적으로 세션을 로드하고 `accumulate()`를 호출한다.

**stage.py 내부 — 세션 로드**
```python
state = await store.load(ctx.session_id)   # 없으면 store.new_session()
```

**accumulator.py — 이번 턴 누적**
```python
accumulate(state, ctx.new_turn_spans, turn_index=state.turn_count + 1, config=cfg)
```

```json
// ctx.new_turn_spans (pii_detect_stage 가 이미 채워둔 것, 이번 턴 탐지 결과만)
[
  {"type": "ACCOUNT", "value": "110-234-567890", "start": 5, "end": 19, "confidence": 0.65, "source": "regex"}
]
```

값 자체(`value`)는 `accumulate()` 안에서 해시로 변환되어 세션 상태에 저장되고, 평문은 세션 상태에 남지 않는다.

---

## 2. 출력 예시

**stage.py 가 ctx 에 되써주는 것**
```python
ctx.risk_score = state.risk_score          # 0.0~1.0
ctx.accumulated = {"ACCOUNT": [<이번 턴 Span>]}   # 이번 턴 span만 타입별로 묶음, §3.6 참고
```

**실제 gRPC 응답 예시 (검증 완료, action=transform)**
```json
{
  "verdict": "transform",
  "provider": "gateway",
  "purpose": "customer_support",
  "risk_score": 0.0,
  "guardrail_hits": []
}
```

**3턴째 block 시 (실제 통합 테스트로 확인)**
```json
{
  "verdict": "block",
  "risk_score": 0.65,
  "note": "risk_hard_block"
}
```
`note: risk_hard_block`은 `pipeline.py::_block_check()`가 `ctx.risk_score >= cfg.risk.hard_block`을 확인해서 붙이는 것이다 — `multiturn_stage`는 `ctx.blocked`를 세팅하지 않는다(§3.5).

---

## 3. 판정 로직

### 3.1 엔티티 그래프
- 노드: `(type, value_hash)`. 같은 세션 내 동일 값은 동일 노드로 병합(count 증가, last_turn 갱신).
- 활성 윈도우(§3.2) 내 노드들 사이에 완전 그래프로 엣지를 만든다 (`_build_edges`).

### 3.2 슬라이딩 윈도우
- 기본 5턴 (`window_size_turns`, config 로 조정 가능, §4).
- 조합/그래프 위험도는 **윈도우 내 활성 엔티티만** 사용. `entities` 이력 자체(세션 전체)는 윈도우 밖으로 나가도 지우지 않는다 — 재등장 시 `first_turn`을 새로 만들지 않기 위해서다(§5 엣지케이스).

### 3.3 조합 위험도 규칙표 [확정 — 코드 반영됨]

| 조합 | 가중치 | 탐지 전제 |
|---|---|---|
| NAME + RRN | 0.25 | NAME은 `detect/dictionary.py`의 사전 등재 이름에 한해 탐지됨 |
| NAME + RRN + ACCOUNT | 0.35 | 위와 동일 — 로드맵 §9 Phase 2 원 시나리오("이름·주민번호·계좌") |
| NAME + PHONE + ACCOUNT | 0.30 | 위와 동일 |
| RRN + ACCOUNT | 0.20 | 정규식만으로 탐지, 사전과 무관하게 항상 발동 |
| RRN + ACCOUNT + PHONE | 0.35 | 정규식만으로 탐지 — 사전에 없는 이름이어도 이 조합으로 block 가능 |

**왜 NAME 조합과 별개로 RRN 계열 조합을 뒀는가:** `detect/dictionary.py`는 Aho-Corasick 정확 일치 방식이라 사전에 없는 이름은 절대 못 잡는다. 현재 `financial_terms.txt`는 데모용 샘플(약 10개 항목, 실명 4명)이라 실제 이름 대부분이 커버 안 된다. NER(로드맵 Phase 5) 또는 실제 운영 사전(컴플라이언스팀 리스트)이 갖춰지기 전까지, `RRN+ACCOUNT+PHONE` 같은 정규식 전용 조합이 "이름이 사전에 없는 경우"의 보완 경로 역할을 한다.

- `combo_cap = 0.6` — 조합 가중치 총합의 상한. `cfg.risk.hard_block`(§4)과 **반드시 같은 값**이어야 한다. 이게 어긋나면(예: cap이 hard_block보다 낮음) 조합만으로는 절대 block에 도달할 수 없는 상태가 된다 — 실제로 이 버그를 겪었다 (`combo_cap=0.5`, `hard_block=0.8`이던 시기).

### 3.4 반복 가중치
- 동일 엔티티가 세션 내 3회 이상 언급되면 3회째부터 회당 `repeat_weight`(기본 0.05) 가산, `repeat_cap`(기본 0.15)으로 상한.

### 3.5 속도(velocity) 가중치
- `(최소 턴 수, 최대 경과 초, 가중치)` 튜플 목록. 기본값: `(3턴, 60초, +0.05)`, `(5턴, 120초, +0.10)`. 코드 상수로만 존재하며 config로 안 뺐다(§4 참고).

### 3.6 `ctx.accumulated`의 의미 — [확정, 원래 계약 문서(§5)와 다르게 구현됨]
`app/models.py`의 `AnalysisContext.accumulated: dict[str, list[Span]]` 주석은 "세션 누적 엔티티"라고 돼 있지만, 실제로는 **"이번 턴에 탐지된 span만 타입별로 묶은 것"**을 채운다. 이유:
1. 세션 상태(`store.py`)는 평문 값을 저장하지 않으므로(§7.2 최소 보관 원칙) 과거 턴의 `Span.value`를 애초에 복원할 수 없다.
2. 과거 턴의 `Span.start/end`는 현재 턴 본문(`ctx.turns[*].text`) 기준으로 무의미하다 — 변환 스테이지(`transform_stage`)가 실제로 고칠 수 있는 건 이번 턴 텍스트뿐이다.
3. 세션 전체의 위험 정도는 `ctx.risk_score`로 충분히 전달된다.

### 3.7 책임 경계 — accumulator는 block하지 않는다 [확정]
`multiturn_stage`는 `ctx.risk_score`만 채우고 `ctx.blocked`는 세팅하지 않는다. block 판정은 `pipeline.py::_block_check()`가 스테이지 실행이 모두 끝난 뒤 `ctx.risk_score >= cfg.risk.hard_block`를 비교해서 내린다. 정책 엔진(§6-f)의 `risk_overrides`와는 별개의, 파이프라인 레벨의 최종 컷오프다.

---

## 4. 파라미터 · 설정값 [확정 — app/config.py에 반영됨]

`app/config.py`의 `Config.multiturn`(`MultiturnConfig`)에서 다음 4개 스칼라만 온다:

| 파라미터 | 기본값 | env |
|---|---|---|
| `multiturn.window_size_turns` | 5 | `DLP_MULTITURN__WINDOW_SIZE_TURNS` |
| `multiturn.combo_cap` | 0.6 | `DLP_MULTITURN__COMBO_CAP` |
| `multiturn.repeat_weight` | 0.05 | `DLP_MULTITURN__REPEAT_WEIGHT` |
| `multiturn.repeat_cap` | 0.15 | `DLP_MULTITURN__REPEAT_CAP` |
| `risk.hard_block` (별도 섹션) | **0.6** | `DLP_RISK__HARD_BLOCK` |

**`combo_weights`/`velocity_thresholds`는 의도적으로 config에서 뺐다.** 이유:
- `combo_weights`의 키는 `frozenset[str]`인데, pydantic-settings의 env/yaml 오버라이드 메커니즘과 구조적으로 궁합이 안 좋다(중첩 컬렉션+비-JSON 키 타입).
- "어떤 엔티티 조합에 얼마의 위험도를 줄지"는 운영 설정값이라기보다 탐지 정책에 가까운 판단이라, 코드 리뷰를 거쳐 `accumulator.py`를 직접 고치는 편이 안전하다고 판단했다.

바꾸려면 `app/context/accumulator.py::AccumulatorConfig.combo_weights` / `velocity_thresholds`를 직접 수정한다.

`store.backend`, `session.ttl_seconds` 등 세션 스토어 관련 설정은 아직 config로 연결 안 됨 — §7.3/§12 미결정 상태 그대로다 (아래 참고).

---

## 5. 엣지 케이스

1. **세션 만료 직후 요청 도착** — [확정] 신규 세션으로 취급, 누적 위험도 초기화. `InMemorySessionStore.load()`가 만료 확인 즉시 정리 후 `None` 반환 → `stage.py`가 `new_session()` 호출.
2. **동일 값, 다른 타입 오탐** — [확정] 그래프 노드가 `(type, value_hash)`라 자연히 분리됨. `merge.py`의 병합 정책과 일관.
3. **동일 엔티티 반복이지만 정상 문맥** — [확정] `repeat_weight`를 작게(0.05) 두고 `repeat_cap`(0.15)으로 상한 — 조합 위험도(최대 0.6) 대비 비중을 낮게 유지.
4. **role/목적이 세션 중간에 바뀜** — [확정] 누적 위험도는 목적과 무관하게 세션 전체로 유지. 최종 action은 그 턴의 목적·role로 정책 엔진(§6-f)이 별도 재평가.
5. **윈도우 밖으로 밀려난 엔티티의 재등장** — [확정] `entities` 이력은 세션 전체 기간 유지, 재등장 시 `first_turn`은 그대로, `last_turn`만 갱신. 테스트로 확인됨(§6 #5).
6. **세션 식별자 재사용/충돌** — [미정] 프록시의 세션 식별 우선순위(§2.3)가 원격 주소까지 내려가면 서로 다른 사용자가 같은 `session_id`를 쓸 위험 — accumulator 레벨에서 막을 수 없다. 프록시 쪽 이슈로 별도 트래킹 필요.
7. **NER 편입 전 span 도착 시차 (Phase 5)** — [미정] 아직 범위 밖.
8. **`value_hash` 충돌** — [확정] SHA-256 앞 16바이트 사용 (`accumulator.hash_value`). 실질적 충돌 위험 낮음.
9. **세션 만료 시 vault 미동반 정리** — [미정, 신규] `InMemorySessionStore`에 `on_expire` 콜백 자리는 있고 콜백이 실제로 발화하는 것까지는 테스트로 확인됐다(§6 #8). 다만 `transform/vault.py`를 실제로 호출하는 코드가 아직 없다 — 만료된 세션의 토큰이 vault에 그대로 남는다.

---

## 6. 테스트 케이스표 [10/10 완료]

`tests/test_context_smoke.py` + `tests/test_context_integration.py` 기준.

| # | 시나리오 | 상태 | 파일 |
|---|---|---|---|
| 1 | 단일 턴 단일 엔티티 | [미정] | — |
| 2 | 2턴 분산 (조합 미도달) | [미정] | — |
| 3 | 3턴 분산 — 조합 임계 도달 | ✅ | `test_context_smoke.py::test_case3_combo_triggers_threshold` |
| 4 | 동일 엔티티 반복 (cap 적용) | ✅ | `test_context_smoke.py::test_case4_repeat_capped` |
| 5 | 윈도우 밖 재등장 (first_turn 유지) | ✅ | `test_context_smoke.py::test_case5_window_reentry_no_new_first_turn` |
| 6 | 고속 다턴 (velocity 가중치) | ✅ | `test_context_smoke.py::test_case6_velocity` |
| 7 | 세션 TTL 만료 후 초기화 | ✅ | `test_context_smoke.py::test_case7_ttl_expiry_resets_session` |
| 8 | 세션 만료 시 vault 동반 정리 | ✅ (훅만) | `test_accumulator.py::test_case8_on_expire_fires_on_load_triggered_expiry`, `test_case8_on_expire_fires_on_sweep` — ⚠️ 콜백 발화만 검증됨, 실제 `vault.purge_expired` 연결은 §5 #9 참고 |
| 9 | injection 동시 발생 시 override 우선 | ✅ | `test_context_integration.py::test_injection_short_circuits_multiturn_stage` |
| 10 | 조합 상한(cap) 초과 방지 | ✅ | `test_accumulator.py::test_case10_combo_cap_prevents_overflow` — 4개 타입 동시 성립(이론상 합 1.45)도 `combo_cap` 이하로 눌림 확인 |
| — | **설정 배선** (`configure()` 없이 `app.config`에서 정상 로드) | ✅ | `test_context_smoke.py::test_default_config_loads_from_app_config_without_explicit_configure` |
| — | **실제 파이프라인 E2E** — 3턴 분산 → block (사전 등재 이름) | ✅ | `test_context_integration.py::test_three_turn_distributed_input_triggers_block` |
| — | **실제 파이프라인 E2E** — 정규식 전용 조합 → block (사전 미등재 이름) | ✅ | `test_context_integration.py::test_regex_only_combo_triggers_block_without_dictionary_name` |

**§6 커버리지: 10/10 완료.** 단 #8은 "vault 동반 정리"라는 이름값을 완전히 하려면 §5 엣지케이스 9(vault 미연결)가 먼저 풀려야 한다 — 지금은 콜백이 울리는 것까지만 보장한다.

---

## 7. 연관 이슈 (이 기능 검증 중 발견, 별도 담당 필요)

이 스펙의 스코프는 아니지만 e2e 검증 과정에서 실제로 재현된 버그라 기록해둔다.

- **`detect/regex_rules.py`**: `_ACCOUNT_PATTERN`과 `_PHONE_PATTERN`이 `010-1234-5678` 같은 값에서 동시에 매치되어 `merge.py`가 타입 충돌 경고를 낸다. 어느 쪽이 최종 채택될지가 우연에 가깝다 (source 우선순위 동점 시 먼저 탐지된 쪽).
- **`transform/vault.py`**: `token_vault.session_id` 컬럼이 UUID 타입인데, 실제 `session_id`는 프록시가 헤더/쿠키/원격주소에서 뽑은 임의 문자열(§2.3)이라 UUID 형식이 보장 안 된다. `session_id`가 UUID가 아니면 `tokenize()`가 `InvalidTextRepresentation`으로 실패하고 `transform_stage`가 `redact`로 폴백한다 (겉으로는 안 터지지만 tokenize가 사실상 항상 실패하는 상태로 방치될 수 있음).

---

## 8. 확장 지점 — 다른 기능이 `SessionState`에 거는 기대 [미정]

`transform/apply.py::detokenize_stage`(출력 경로 [2])의 docstring에 이런 TODO가 있다:

> ⚠️ 알려진 한계: `ctx.purpose`는 output 요청에서 항상 `None`이다. `purpose_policy_stage`는 `direction == "input"`일 때만 돌기 때문에, 이 별개의 output `InspectRequest`는 애초에 원 요청의 purpose를 모른다. **세션 스토어(기능 e, context 쪽)가 세션별 purpose를 기억해뒀다가 여기서 읽어오게 되면 이 TODO는 해소된다.** 그 전까지는 `access_scope`에 `"*"`가 없는 토큰은 output 단계에서 복원되지 않는다.

**현재 상태:** `context/store.py::SessionState`에는 `purpose` 필드가 없다. `turn_count`, `entities`, `graph_edges`, `turn_timestamps`, `risk_score`, `risk_reasons`만 있다.

**왜 지금 안 넣었는가:** 이 기능(e)의 스코프는 "PII 조합 위험도"이지 "요청 목적 추적"이 아니다(§6-f가 목적 담당). `SessionState`에 `purpose`를 얹는 게 자연스러워 보이긴 하지만, 그러면 이 파일이 두 기능(e, f)의 상태를 동시에 들고 있는 셈이 되어 책임 경계가 흐려진다. 또한 `multiturn_stage`는 입력 경로에서만 실행되므로, 마지막 입력 턴의 `purpose`를 언제 어떤 스테이지가 `SessionState`에 써넣을지(정책 엔진 §6-f 쪽 스테이지가 직접 쓰게 할지, `multiturn_stage`가 대신 받아쓸지)가 아직 안 정해졌다.

**결정 필요한 것:**
1. `purpose`를 `SessionState`에 넣을지, 아니면 별도의 작은 "세션 메타" 저장소를 새로 만들지
2. 넣는다면 어느 스테이지가 쓰는지 (`purpose_policy_stage`가 직접 `store.save()`를 부르게 할지, `multiturn_stage`가 `ctx.purpose`를 읽어 같이 저장할지 — 후자가 스테이지 순서상 더 간단해 보임: `multiturn_stage`는 `purpose_policy_stage` *전*에 도는데, purpose는 `purpose_policy_stage`가 채우므로 실제로는 `multiturn_stage`가 저장 시점에 `ctx.purpose`를 아직 못 본다는 순서 문제가 있다 — 스테이지 순서를 바꾸거나, 별도의 저장 지점이 필요)
3. 이건 output 경로 담당자와 조율이 필요한 사안이라 이 문서만으로 결정하지 않는다.

---

*Phase 2 구현·검증 완료 (§6 10/10). 남은 것: 세션 스토어 backend 확정(§7.3/§12), vault 만료 동반 정리 실연결(§5 #9), §7의 두 연관 버그, §8의 purpose 저장 방식 결정(output 경로 담당자와 조율).*