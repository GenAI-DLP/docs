# pipeline 스테이지 구현 계약

`dlp-server` 요청 파이프라인의 **확정된 구현 계약**이다. 기능 a~h 담당자가 자기 로직을
파이프라인에 붙일 때 지켜야 하는 인터페이스를 정의한다.

- [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §3 / §5 = **설계 의도** (구현 전 작성)
- 이 문서 = **실제 구현된 계약** (스캐폴딩·공유 모듈 반영). 기능을 붙이는 코드는 이 문서를 따른다.

---

## 1. 진입점 — `pipeline.analyze()`

```python
def analyze(
    session_id: str,
    direction: str,          # "input" | "output"
    method: str,
    path: str,
    headers: dict[str, str],
    body: bytes,
    *,
    config: Config | None = None,
) -> Decision
```

- gRPC 서버 · HTTP API · eval 스크립트가 모두 이 함수를 호출한다 (transport-agnostic 코어).
- `direction` 에 따라 내부에서 `_analyze_input` / `_analyze_output` 로 분기한다.
- 전체가 `try/except` 로 감싸여 **예외를 전파하지 않는다** (§7).

---

## 2. `Stage` 타입

```python
Stage = Callable[[AnalysisContext], AnalysisContext]
```

- 스테이지는 `AnalysisContext`(이하 `ctx`) 하나를 받아 **필드를 갱신해 같은 종류의 객체를 반환**한다.
- 스테이지는 `Decision` 을 만들지 않는다. 최종 판정은 파이프라인이 모든 스테이지 실행 후 생성한다 (§5).
- 스테이지는 예외를 던지지 않는다 (던지면 파이프라인이 fail-closed 로 처리한다).

---

## 3. `AnalysisContext` — 스테이지가 읽고 쓰는 상태

`app/models.py` 정의 (필드 의미는 [`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §5).

| 필드 | 타입 | 누가 채우나 | 비고 |
|---|---|---|---|
| `session_id` | `str` | 파이프라인 | 프록시가 넘긴 값 |
| `direction` | `str` | 파이프라인 | `input` / `output` |
| `provider` | `str` | 파이프라인 | 어댑터 판별 결과 (`gateway` …) |
| `role` | `str \| None` | 파이프라인 (`_analyze_input`/`_analyze_output` 이 ctx 생성 시 `role_resolver.resolve(headers)`) | 접근 제어 축. 초기값 `None` |
| `turns` | `list[Turn]` | 파이프라인(어댑터) | 요청/응답 본문에서 추출한 대화 턴 |
| `new_turn_spans` | `list[Span]` | [3] PII 탐지 (b) | 이번 턴에서 탐지된 엔티티. 초기값 `[]` |
| `accumulated` | `dict[str, list[Span]]` | [4] 멀티턴 (e) | "이번 턴 span 을 타입별로 묶은 것" (구현 상세는 e 스펙 §3.6). 초기값 `{}` |
| `risk_score` | `float` | [4] 멀티턴 (e) | `0.0 ~ 1.0`. 초기값 `0.0` |
| `injection` | `InjectionVerdict` | [2] Input Guard (c) | `hit` / `score` / `pattern`. 초기값 `hit=False` |
| `blocked` | `bool` | 아무 스테이지 | injection/risk 외 사유로 차단 요청. 초기값 `False` |
| `block_reason` | `dict \| None` | 아무 스테이지 | `blocked` 시 근거. `guardrail_hits` 한 조각 형태 `{"type": "..."}`. 초기값 `None` |
| `purpose` | `str \| None` | [5] 목적+정책 (f) | 목적 분류 결과 (`purpose_ref` 코드). 초기값 `None` |
| `purpose_confidence` | `float \| None` | [5] 목적+정책 (f) | 초기값 `None` |
| `span_actions` | `list[tuple[Span, str]]` | [5] 목적+정책 (f) | `(span, action)` — action 은 `TRANSFORM_ACTIONS` 중 하나. g 가 실행. 초기값 `[]` |

**본문 변경**은 `ctx.turns[i].text` 를 수정하는 방식으로 한다. 파이프라인이 이후 어댑터로 재조립한다 (§5).

---

## 4. 스테이지 등록 & 실행 순서

`app/pipeline.py` 에 방향별 스테이지 리스트가 있다. 담당자는 자기 함수를 해당 리스트에 등록한다.

```python
_INPUT_STAGES:  list[Stage] = [ ... ]
_OUTPUT_STAGES: list[Stage] = [ ... ]
```

파이프라인은 리스트를 **순서대로** 실행한다 (`ctx = stage(ctx)` 반복).

### 4.1 input 경로 순서

[`../../architecture/dlp-server-architecture.md`](../../architecture/dlp-server-architecture.md) §3.1 번호 목록을 따른다.

실제 배선(`app/pipeline.py`):
`_INPUT_STAGES = [injection_guard, pii_detect_stage, multiturn_stage, purpose_policy_stage,
remember_purpose_stage, transform_stage]`

| # | 스테이지 (함수) | 기능 | 채우는 `ctx` 필드 |
|---|---|---|---|
| 1 | 어댑터 선택 + 턴 추출 | 인프라 | `turns` (파이프라인이 수행) |
| 2 | `injection_guard` | c | `injection` (적중 시 `blocked` + `block_reason`) |
| 3 | `pii_detect_stage` | b | `new_turn_spans` (`app.detect.detect(text) -> list[Span]`) |
| 4 | `multiturn_stage` | e | `accumulated`, `risk_score` |
| 5 | `purpose_policy_stage` | f | `purpose`, `purpose_confidence`, `span_actions` (`block` action → `blocked`) |
| 5.5 | `remember_purpose_stage` | e | (세션 스토어에 `purpose` 기록 — 출력 경로 detokenize 가 조회) |
| 6 | `transform_stage` | g, a | `turns[*].text` |
| 7 | 본문 재조립 + 감사 로그 | 인프라 | (파이프라인이 수행) |

### 4.2 output 경로 순서

실제 배선: `_OUTPUT_STAGES = [output_guard, detokenize_stage]` — **Output Guard 가 detokenize 앞.**
토큰 라벨 `<PII:…>` 은 정규식에 안 걸리므로, 재스캔이 잡는 건 모델이 새로 만든 PII 뿐이다.
detokenize 뒤에 재스캔하면 복원된 인가 PII 까지 다시 가려버린다.

| # | 스테이지 (함수) | 기능 |
|---|---|---|
| 1 | 응답 파싱 (`parse_response`) | 인프라 |
| 2 | `output_guard` (타인 PII 재스캔·재마스킹 + 인젝션 순응 → `block`) | c |
| 3 | `detokenize_stage` (인가 검사 후 복원. purpose 는 `get_last_purpose()` 조회) | a |
| 4 | 본문 재조립 + 감사 로그 | 인프라 |

---

## 5. 판정 규칙 — `Decision` 이 정해지는 방식

파이프라인은 스테이지를 전부 실행한 뒤 `ctx` 를 보고 `Decision.action` 을 정한다.

1. **block** — 아래 중 하나라도 참이면:
   - `ctx.blocked == True` (근거는 `ctx.block_reason` 를 `guardrail_hits` 에 첨부)
   - `ctx.injection.hit == True`
   - `ctx.risk_score >= config.risk.hard_block` (기본 `0.6`, input 경로만)
2. **allow** — block 이 아니고, 스테이지가 `ctx.turns[*].text` 를 **바꾸지 않았으면**.
   원본 `body` 를 그대로 통과시킨다 (재직렬화하지 않는다).
3. **transform** — block 이 아니고, `ctx.turns[*].text` 가 **바뀌었으면**.
   어댑터 `rebuild(body, ctx.turns)` 로 새 본문을 만들어 `transformed_body` 에 담는다.

`allow` / `transform` 판정은 스테이지 실행 전후의 `[t.text for t in ctx.turns]` 를 비교해 결정한다
(재직렬화가 공백·키 순서를 바꿔 생기는 오탐 방지).

> **block 신호 채널:**
> - `injection.hit` — Input Guard(c) 전용. 프롬프트 인젝션·탈옥.
> - `risk_score >= hard_block` — 멀티턴(e) 전용. 누적 위험도. 누적 스테이지가 input 에만 있어 input 경로에서만 검사.
> - `ctx.blocked` (+ `ctx.block_reason`) — **그 밖의 모든 사유.** 명시적 데이터 반출 요청, 정책상 무조건 차단,
>   Output Guard 의 타인 PII 재생성·기밀 유출 등. input·output 양쪽에서 동작.
>
> `_run_stages` 는 `ctx.blocked` 가 서면 이후 스테이지를 실행하지 않는다(조기 중단).

---

## 6. `Decision` 과 `reason_obj`

```python
@dataclass
class Decision:
    action: str                     # "allow" | "block" | "transform"  (wire 3값)
    transformed_body: bytes | None  # action == "transform" 일 때만
    reason_obj: dict                # → Verdict.reason (JSON 직렬화)
```

`reason_obj` 봉투는 **파이프라인이 생성한다** (`_reason()` 헬퍼). 스테이지는 개별 항목만 채운다.
형태는 [`../../architecture/dlp-proto.md`](../../architecture/dlp-proto.md) §3.2 를 따른다.

```json
{
  "verdict": "transform",
  "provider": "gateway",
  "transforms": [{"entity": "RRN", "action": "tokenize", "token_label": "<PII:RRN:1>"}],
  "entities_summary": [{"type": "RRN", "masked_preview": "8801**-*******", "confidence": 0.99}],
  "purpose": "doc_summarize",
  "risk_score": 0.42,
  "guardrail_hits": [],
  "fail_policy_applied": false
}
```

- **원문 금지.** `entities_summary` 는 마스킹 미리보기만. `reason_obj` 어디에도 원문을 넣지 않는다.
- 판정마다 감사 로그 1건이 기록된다 ([`../../schemas/dlp-server/log-event.md`](../../schemas/dlp-server/log-event.md)).

---

## 7. fail-closed

- `analyze()` 전체가 `try/except` 로 감싸여 있다. 내부 오류 시에도 **유효한 `Decision` 을 반환**한다
  (gRPC 에러를 던지지 않는다).
- 예외 시 기본값 `config.fail_action` (기본 `block`, 시연 안정용 `allow` 스위치).
  `reason_obj` 에 `fail_policy_applied: true` 를 남긴다.
- 스테이지에서 예외가 나도 파이프라인이 여기서 잡는다. 스테이지는 예외로 흐름을 제어하지 않는다.

---

## 8. 스테이지 작성 규칙

**해야 하는 것**

- 시그니처는 `def stage(ctx: AnalysisContext) -> AnalysisContext`.
- `ctx` 에서 필요한 값을 읽고, 자기 담당 필드를 채우고, `ctx` 를 반환한다.
- 차단을 요청하려면 `ctx.blocked = True` 로 두고 `ctx.block_reason = {"type": "...", ...}` 에 근거를 담는다.
  (`injection.hit` / `risk_score` 는 각각 Input Guard / 멀티턴 전용 신호이므로 그 외 스테이지는 쓰지 않는다.)
- 교체 가능성이 있는 지점(목적 분류기, Input Guard 등)은 인터페이스 뒤에 두고 규칙 기반을 기본값으로 한다.
- 유닛 테스트는 스테이지 함수를 직접 호출한다. 파이프라인 통합 테스트는 `_INPUT_STAGES` 를 교체(monkeypatch)한다.

**하지 말아야 하는 것**

- `Decision` 을 반환하거나 예외를 던져 판정에 직접 개입하지 않는다.
- `reason_obj` 봉투 전체를 만들지 않는다 (파이프라인 담당).
- 원문을 로그·`reason_obj` 에 넣지 않는다.
- `AnalysisContext` / `Decision` / `Span` 의 필드를 추가·변경할 때는 먼저 합의한다 (공용 계약).

---

## 9. 예시 — 스테이지 붙이기

```python
# app/detect/__init__.py — 스테이지 함수를 모듈이 직접 제공한다
def detect(text: str) -> list[Span]: ...           # 레이어 실행 + 병합
def pii_detect_stage(ctx: AnalysisContext) -> AnalysisContext:
    ctx.new_turn_spans = detect(ctx.turns[-1].text)  # 마지막 턴만
    return ctx

# app/pipeline.py
from app.detect import pii_detect_stage
_INPUT_STAGES: list[Stage] = [
    injection_guard,
    pii_detect_stage,   # [3]
    ...
]
```

파이프라인이 요청마다 `pii_detect_stage` 를 실행하고, 그 결과(`ctx.new_turn_spans`)를 이후 스테이지
([4] 멀티턴, [5] 정책)가 소비한다. block/transform/로그는 파이프라인이 처리한다.

