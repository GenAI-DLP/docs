# Multi-turn Context 분석

문서 내 상태 표기: [확정] 합의 완료 · [논의] 방향은 있으나 확정 전 · [미정] 착수 전.

한 줄 정의: 세션 전체 대화를 추적해 여러 턴에 분산 입력된 PII 조합을 엔티티 그래프 + 슬라이딩 윈도우로 탐지하고 누적 위험도를 매긴다.

담당 모듈: `context/store.py`, `context/accumulator.py`
설계 의도·전체 맥락: `../../architecture/dlp-server-architecture.md` §6-e
관련 스키마: `../../schemas/dlp-server/session-context.md`
색인: `../spec-index.md`

---

## 1. 입력 예시

파이프라인 스테이지 4(세션 컨텍스트 누적, §3.1)가 이 모듈을 호출할 때 넘기는 입력이다.

**store.py — 세션 로드**

```
SessionStore.load(session_id: str) -> SessionState | None
```

```json
// 기존 세션 상태 (2턴째까지 누적된 상태, 3턴째 요청 직전)
{
  "session_id": "sess_8f2a...",
  "created_at": "2026-09-03T09:12:01Z",
  "expires_at": "2026-09-03T09:42:01Z",
  "turn_count": 2,
  "entities": [
    {"type": "NAME", "value_hash": "a1b2...", "first_turn": 1, "last_turn": 1, "count": 1},
    {"type": "RRN",  "value_hash": "c3d4...", "first_turn": 2, "last_turn": 2, "count": 1}
  ],
  "risk_score": 0.42
}
```

**accumulator.py — 이번 턴 누적**

```
accumulate(state: SessionState, new_turn_spans: list[Span], turn_index: int) -> SessionState
```

```json
// 이번 턴(3턴째)에 새로 탐지된 span
{
  "new_turn_spans": [
    {"type": "ACCOUNT", "value": "110-***-******", "start": 12, "end": 24, "confidence": 0.93, "source": "regex"}
  ],
  "turn_index": 3
}
```

값 자체(`value`)는 누적기에 들어가는 순간 해시로 변환되며, 평문 값은 세션 상태에 저장하지 않는다(§7.2 감사 로그 원칙과 동일한 최소 보관 원칙 적용).

---

## 2. 출력 예시

**accumulator.py — 누적 결과**

```json
{
  "session_id": "sess_8f2a...",
  "turn_count": 3,
  "entities": [
    {"type": "NAME",    "value_hash": "a1b2...", "first_turn": 1, "last_turn": 1, "count": 1},
    {"type": "RRN",     "value_hash": "c3d4...", "first_turn": 2, "last_turn": 2, "count": 1},
    {"type": "ACCOUNT", "value_hash": "e5f6...", "first_turn": 3, "last_turn": 3, "count": 1}
  ],
  "entity_graph_edges": [
    {"from": "NAME:a1b2", "to": "RRN:c3d4",     "co_occur_window": true},
    {"from": "RRN:c3d4",  "to": "ACCOUNT:e5f6", "co_occur_window": true}
  ],
  "risk_score": 0.87,
  "risk_delta_reason": [
    "combo:NAME+RRN(+0.25)",
    "combo:RRN+ACCOUNT(+0.20)",
    "velocity: 3 turns / 40s (+0.05)"
  ]
}
```

**AnalysisContext 반영 (§5 계약 타입)**

```json
{
  "accumulated": { "NAME": [...], "RRN": [...], "ACCOUNT": [...] },
  "risk_score": 0.87
}
```

**정책 엔진으로의 판정 트리거 (risk_overrides, §6-f)**

```json
{"action": "block", "reason_obj": {"trigger": "risk_score >= 0.8", "risk_score": 0.87, "stage": "multiturn_accumulator"}}
```

---

## 3. 판정 로직 (룰 / 프롬프트 / 임계값)

[논의] 아래는 초안이며 Phase 2 착수 시 확정한다.

### 3.1 엔티티 그래프
- 노드: `(type, value_hash)`. 같은 세션 내 동일 값은 동일 노드로 병합된다(count 증가, last_turn 갱신).
- 엣지: 슬라이딩 윈도우 내에서 함께 언급된 두 노드 사이에 생성된다. 엣지 자체에 가중치를 두지 않고, 위험도 계산은 §3.3 조합 규칙표를 따른다.
- 그래프는 세션 상태에 인메모리로만 유지하고 평문 값은 저장하지 않는다(해시만).

### 3.2 슬라이딩 윈도우
- 윈도우 크기 N턴(기본값 §4 참고)을 넘어간 엔티티는 그래프 신규 엣지 생성 대상에서 제외되지만, `entities` 누적 목록(전체 세션 이력)에서는 제거하지 않는다 — 위험도 계산의 "조합" 판정은 윈도우 기준, 감사·차단 이력 판정은 세션 전체 기준으로 이원화한다.
- 근거: 대화가 길어지며 화제가 바뀐 뒤에도 초반 PII 조합을 계속 위험으로 잡으면 오탐이 누적되므로, "최근에 실제로 함께 쓰이려는 조합"만 그래프 위험도에 반영한다.

### 3.3 조합 위험도 규칙표 (초안)
| 조합 | 가중치 | 비고 |
|---|---|---|
| NAME + RRN | +0.25 | 신원 확정 조합 |
| RRN + ACCOUNT | +0.20 | 금융 사기 활용 가능 조합 |
| NAME + PHONE + ACCOUNT | +0.30 | 3종 조합 시 개별 합산이 아닌 별도 규칙으로 상한 부여 |
| 동일 타입 반복 언급 (count ≥ 3) | +0.05 / 회 (최대 +0.15) | 같은 PII를 여러 번 되풀이 입력 |
| 턴 속도(velocity): 짧은 시간에 다수 턴 | +0.05 ~ +0.10 | §3.4 참고 |

- 조합 규칙은 상한(cap)을 둔다 — 단순 합산이 1.0을 쉽게 넘겨 임계값의 변별력이 사라지는 것을 방지.
- 규칙표는 `policy/policy.yaml`과 별개로 `context/` 설정에 둔다(정책 엔진의 목적×role 규칙과는 계층이 다름 — 이건 "이 세션이 얼마나 위험한 정보를 모으고 있는가"이고 정책 엔진은 "이 정보를 이 목적·role에 줘도 되는가").

### 3.4 턴 간격·속도 신호
- 짧은 시간에 여러 턴이 이어지며 서로 다른 타입의 PII가 연달아 들어오는 패턴("우회 분산 입력"의 전형)에 소폭 가중치를 더한다.
- 순수 규칙: `turn_count >= 3 and elapsed_seconds <= 60` → `+0.05`, `turn_count >= 5 and elapsed_seconds <= 120` → `+0.10` (초안 수치, 튜닝 대상).

### 3.5 임계값 처리
- `risk_score >= BLOCK_THRESHOLD` (§6-f `risk_overrides`의 `risk_score >= 0.8`과 동일 값을 공유) → block.
- block 미도달이어도 `risk_score`는 정책 엔진(§6-f)의 컨텍스트 입력으로 전달되어 엔티티별 조치 강도(mask vs tokenize vs block)에 영향을 준다 — accumulator는 직접 block하지 않고 위험도만 계산하며, 최종 action 결정은 정책 엔진이 한다는 원칙을 유지한다. 단, `risk_overrides`에 정의된 하드 컷오프(§6-f `policy.yaml`)는 예외적으로 accumulator 단계에서도 조기 종료(short-circuit) 가능하도록 열어둔다 — 성능·명확성 트레이드오프는 Phase 2에서 결정.

---

## 4. 파라미터 · 설정값

[논의] `context/config.yaml` (또는 전역 `config.yaml`의 하위 섹션) 초안.

| 파라미터 | 기본값(초안) | 설명 |
|---|---|---|
| `window.size_turns` | 5 | 엔티티 그래프 엣지 생성에 사용하는 슬라이딩 윈도우 턴 수 |
| `session.ttl_seconds` | 1800 (30분) | 세션 만료 시간. 만료 시 store + vault 함께 정리 |
| `risk.block_threshold` | 0.8 | 누적 위험도 차단 임계값 (`policy.yaml`의 `risk_score >= 0.8`과 동일 값 유지) |
| `risk.combo_weights` | §3.3 표 | 조합별 가중치 (엔티티 타입 쌍 → float) |
| `risk.combo_cap` | 0.5 | 단일 판정에서 조합 가중치 합산 상한 |
| `risk.repeat_weight` | 0.05 | 동일 엔티티 반복 언급 1회당 가중치 |
| `risk.repeat_cap` | 0.15 | 반복 가중치 합산 상한 |
| `velocity.thresholds` | §3.4 표 | (턴 수, 경과 시간) → 가중치 매핑 |
| `store.backend` | `memory` | `memory` \| `postgres` — §7.3 미정 사항과 연동, 인터페이스 뒤에서 전환 |
| `store.sweep_interval_seconds` | 300 | 인메모리 TTL 스윕 주기 |

---

## 5. 엣지 케이스

[논의] 구현 시 테스트 케이스(§6)로 구체화한다.

1. **세션 만료 직후 요청 도착** — TTL이 막 지난 세션에 새 턴이 들어오면 신규 세션으로 취급할지, 잠시(grace period) 이전 상태를 이어받을지 결정 필요. 기본안: 신규 세션으로 취급(누적 위험도 초기화), vault는 별도 TTL로 관리되므로 영향 없음(§7.2).
2. **동일 값, 다른 타입으로 오탐** — 같은 문자열이 정규식/사전/NER에서 서로 다른 타입으로 탐지되는 경우, 그래프 노드를 `(type, value_hash)`로 분리해 이중 계산되지 않게 병합 규칙(§6-b merge.py)과 일관성을 맞춘다.
3. **동일 엔티티 반복이지만 다른 세션 문맥** — 같은 사람 이름이 상담 목적으로 반복 언급되는 정상 케이스(예: 상담원이 같은 고객명을 여러 번 부름)와 반출 시도용 반복 입력을 구분하기 어려움 — `repeat_weight`를 작게 두고 조합 가중치 위주로 판정하는 이유.
4. **role/목적이 세션 중간에 바뀌는 경우** — 목적 분류(§6-f)는 턴별로 재평가되므로, 누적 위험도는 목적과 무관하게 세션 전체로 유지하되 최종 action은 그 턴의 목적·role로 정책 엔진이 재평가. 즉 위험도 누적과 접근 제어 판정을 분리.
5. **윈도우 밖으로 밀려난 엔티티의 재등장** — 1턴에 나온 엔티티가 10턴 뒤 다시 언급되면 새 엣지로 취급(윈도우 기준)하되, `entities` 이력에는 이미 존재하므로 신규 first_turn을 만들지 않고 `last_turn`만 갱신.
6. **세션 식별자 재사용/충돌** — 프록시의 세션 식별 우선순위(§2.3: 헤더 → 쿠키 → 원격 주소)가 원격 주소까지 내려가는 경우 서로 다른 사용자가 같은 session_id로 섞일 위험 — accumulator 레벨에서 막을 수 없으므로 문서화만 하고, 프록시 측 이슈로 별도 트래킹.
7. **NER 지연으로 인한 부분 span 도착(후반 단계)** — NER이 편입되기 전(Phase 2)에는 정규식/사전 span만으로 누적하므로 문제 없으나, Phase 5에서 NER이 붙으면 같은 턴 내 span이 시차를 두고 도착할 수 있어 누적 시점 처리 필요(당장은 범위 밖으로 기록만).
8. **value_hash 충돌** — 해시 함수 선택 시 충돌 가능성 고려(예: SHA-256 등 안전한 해시 사용을 전제, 별도 명시 필요).

---

## 6. 테스트 케이스표

[미정] 착수 시 `tests/test_accumulator.py`에 구체 케이스로 반영한다. 아래는 표 골격.

| # | 시나리오 | 입력 턴 시퀀스 | 기대 risk_score 방향 | 기대 action |
|---|---|---|---|---|
| 1 | 단일 턴 단일 엔티티 | T1: RRN | 낮음 (조합 없음) | 정책 엔진 개별 판정 |
| 2 | 2턴 분산 — 이름+주민번호 | T1: NAME / T2: RRN | combo 가중치 반영 상승 | risk 상승, 임계 미만이면 정책 엔진 판정 |
| 3 | 3턴 분산 — 이름+주민번호+계좌 | T1: NAME / T2: RRN / T3: ACCOUNT | 임계값(0.8) 초과 | block |
| 4 | 동일 엔티티 반복 (동일 세션) | T1~T4: 같은 RRN 반복 | repeat_weight만큼 소폭 상승, cap 이내 | 정책 엔진 판정 |
| 5 | 윈도우 밖 재등장 | T1: NAME / T7: RRN (window=5) | 낮음 (엣지 미생성) | 정책 엔진 판정 |
| 6 | 고속 다턴(velocity) | T1~T5: 5턴을 100초 내 입력, 서로 다른 타입 | velocity 가중치 추가 반영 | risk 상승 |
| 7 | 세션 TTL 만료 후 재요청 | T1 이후 TTL 경과 → 새 요청 | 신규 세션으로 초기화 | 이전 이력 미반영 확인 |
| 8 | 세션 삭제 시 vault 동반 정리 | 세션 만료 트리거 | — | vault 레코드 revoked/삭제 확인(§7.2) |
| 9 | injection 동시 발생 | T1: 정상 PII / T2: 인젝션 패턴 적중 | risk_overrides(`injection.hit`) 우선 | block (accumulator 결과와 무관) |
| 10 | 조합 상한(cap) 초과 시도 | T1~T6: 다수 조합 동시 성립 | combo_cap으로 상한 고정 확인 | risk_score가 cap 이상으로 튀지 않음 |

---

*이 문서는 스텁에서 상세 스펙으로 전환하는 초안이다. §3~4의 수치·규칙은 Phase 2 구현 착수 및 `eval/datasets/multiturn/` 기반 튜닝 후 확정한다.*