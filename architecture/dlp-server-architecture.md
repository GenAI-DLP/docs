# DLP 서버 아키텍처

생성형 AI Dynamic DLP Gateway의 **의미 기반 검사 백엔드**(`dlp-server`, FastAPI)에 대한 아키텍처 문서다.
전체 시스템에서의 위치(§1), 프록시와의 계약(§2), 요청 파이프라인(§3), 모듈 구조(§4), 계약 타입(§5),
기능별 설계(§6), 데이터 저장소(§7), 구현 로드맵(§9)을 정의한다.

문서 내 상태 표기: **[확정]** 합의 완료 · **[논의]** 방향은 있으나 확정 전 · **[미정]** 착수 전.

---

## 1. 시스템 컨텍스트

### 1.1 문제 정의

금융 직원이 외부 LLM(ChatGPT, Claude 등)을 업무에 활용하는 과정에서 개인정보·신용정보·내부정보가
외부로 유출될 위험이 있다. 단일 요청 안의 PII 패턴만 탐지·차단하는 기존 DLP는 다음 우회에 취약하다.

- 개인정보를 여러 턴에 나누어 입력
- 우회적 표현으로 민감정보 전달
- AI에게 민감정보 조합·재구성을 요청
- 목적과 무관한 개인정보까지 AI에 제공

따라서 "개인정보가 포함되었는가"를 넘어 **"사용자가 AI로 무엇을 하려는가", "대화에서 어떤 정보가
누적되고 있는가"** 까지 판단하고, 발견 시 무조건 차단하는 대신 **업무 목적에 필요한 최소한의 정보만
필요한 수준으로 제공**하도록 데이터를 동적 변환하는 것이 이 시스템의 목표다.

### 1.2 컴포넌트 분리

시스템은 **네트워크 레벨**과 **애플리케이션 레벨** 두 컴포넌트로 나뉜다.

```
[직원 PC] ──TLS──▶ [Go 프록시]  ── SNI allowlist 필터 → TLS 종단(MITM) → HTTP 파싱
                       │
                       │  gRPC :50051   InspectRequest (평문 본문, 호출 중 트래픽 Hold)
                       ▼
                 [dlp-server]  ── 본 문서의 범위. 의미 기반 판단만 수행
                       │
                       │  Verdict (allow / block / transformed_body)
                       ▼
              [Go 프록시가 판정대로 실제 외부 LLM에 중계]
                       │
              [외부 LLM 응답] ──▶ Go 프록시가 다시 Hold
                       │  gRPC   InspectRequest (direction = "output")
                       ▼
                 [dlp-server]  ── Output Guard
                       │  Verdict
                       ▼
                  [직원 PC에 응답 전달]     [관리자 대시보드: 감사 로그]
```

| Go 프록시 (별도 컴포넌트) | dlp-server (본 문서) |
|---|---|
| SNI 기반 검사 대상 도메인 allowlist 필터 | 요청 파이프라인 (Input Guard, 멀티턴 분석, PII 탐지, 목적·정책, 동적 변환) |
| 회사 루트 CA 재서명 MITM, TLS 종단 | 응답 파이프라인 (Output Guard, detokenize) |
| HTTP 요청/응답 파싱, 세션 식별자·role 헤더 추출 | gRPC 서버로 판정 제공 |
| gRPC로 판정 요청 후 Hold, 판정대로 **외부 LLM에 직접 중계** | **외부 LLM을 직접 호출하지 않음** |
| DLP 서버 장애 시 fail-closed 집행 | 내부 예외 시에도 유효한 판정 반환 |

> dlp-server는 외부 LLM을 호출하지 않으므로 업스트림 LLM 클라이언트 모듈이 필요 없다.
> 사내 LLM을 dlp-server가 직접 호출하는 별도 배포 모드를 도입할 때만 추가한다.

---

## 2. 프록시 인터페이스 계약

### 2.1 프로토콜 — gRPC unary  **[확정]**

Go 프록시와 dlp-server는 하나의 protobuf 계약을 공유한다. 이 계약은 **두 컴포넌트의 공유 소스**로
관리하며(예: 공용 서브모듈), dlp-server 측에서 임의로 필드를 추가·변경하지 않는다. 필드 변경은 양
팀 합의와 프록시 재조율을 전제로 한다.

- `Inspect(InspectRequest) returns (Verdict)` **unary** RPC. 스트리밍 아님.
- `InspectRequest` = `session_id`, `direction`(input|output), `method`, `path`, `headers`, `body`.
- `Verdict` = `action`(allow|block|transform), `transformed_body`, `reason`(JSON 문자열).

전체 메시지 정의·필드 의미·에러/타임아웃 규약은 **[dlp-proto.md](dlp-proto.md)** 를 단일 기준으로 한다.

### 2.2 계약 규칙

- **unary 호출.** 응답을 청크 단위로 검사하는 양방향 스트리밍은 프로토콜과 프록시를 모두
  바꿔야 하므로 현재 범위 밖이다. 스트리밍 응답(SSE 등)은 프록시가 전체 버퍼링한 뒤 한 번에 넘긴다.
- **`action`은 세 값(`allow` / `block` / `transform`)만 전송한다.** 동적 변환의 세부 전략
  (`mask`, `generalize`, `aggregate`, `tokenize`, `synthetic`, `redact`)은 모두 `transform`으로
  직렬화하고, 세부 종류와 근거는 `reason`(JSON)에 담는다. `action` enum을 확장하지 않는다.
- **`reason`** 은 dlp-server가 만든 판정 근거 객체를 그대로 JSON 직렬화한 것이다. 관리자
  대시보드와 성능 평가 스크립트가 이 필드를 파싱한다.

### 2.3 세션 상관관계와 role

- 프록시가 추출한 `session_id`를 멀티턴 분석의 키로 그대로 사용한다. 프록시의 세션 식별
  우선순위는 `X-Corp-User-Id` 헤더 → 서비스 세션 쿠키 → 원격 주소 순이다.
- **요청자 role은 dlp-server가 헤더에서 해석한다.** 사내 AI Gateway가 주입하는 사용자
  식별 헤더를 role로 매핑하며, 이 값은 목적 기반 접근 제어(§6-f)의 정책 입력이 된다.

### 2.4 장애 정책 — fail-closed  **[확정]**

- **프록시 측:** dlp-server가 타임아웃/연결 실패로 응답하지 않으면 요청을 **차단(fail-closed)** 한다.
  이때 로그에 "DLP 판정이 아닌 장애 대응"임을 구분하는 플래그를 남긴다.
- **dlp-server 측:** 파이프라인 전체를 예외 처리로 감싸 **내부 오류 시에도 유효한 판정을 반환**한다
  (gRPC 오류를 던지지 않는다). 기본값은 `block`이며, 시연 안정용 `allow` 스위치를 설정에 둔다.
- **지연 예산:** 프록시 호출 deadline 3초. dlp-server soft budget 2.5초. 외부 모델 호출이 없는
  로컬 파이프라인은 600ms 이내를 목표로 한다.

---

## 3. 요청 파이프라인

모든 기능은 파이프라인의 미들웨어 스테이지로 구현한다. 각 스테이지는 요청 컨텍스트 객체를
넘겨받아 갱신하고, 마지막에 하나의 판정 객체를 만든다. 후순위 기능을 나중에 끼워 넣기 쉬운 구조다.

```
                     ┌──────────── 공용 엔진 (PII 탐지 · 정책) ────────────┐
                     │                                                    │
[수신] → [Input Guard] → [하이브리드 PII 탐지] → [멀티턴 분석] ────────────┤
                     │                                                    │
                     │                  [목적 분석 + 정책 엔진]  ◀─────────┘
                     │                          │
                     │                  [동적 변환 + 토큰화] → 본문 재조립 → (프록시가 업스트림 중계)
                     │
   (응답)  [Output Guard] : PII 재검사 → detokenize → 정책 위반 필터 → 응답 본문 재조립
                     │
                  [구조화 감사 로그]
```

### 3.1 입력 경로 스테이지 순서

1. **어댑터 선택** — 본문 형식(사내 Gateway / OpenAI / Anthropic)을 판별해 대화 턴을 추출
2. **Input Guard** — 프롬프트 인젝션·탈옥·명시적 반출 요청 탐지. 적중 시 즉시 `block`
3. **하이브리드 PII 탐지** — 정규식/체크섬 + 사전 + NER 실행 후 병합
4. **세션 컨텍스트 누적** — 세션 스토어 로드 → 이번 턴 엔티티 누적 → 누적 위험도 갱신 → 저장
5. **목적 분석** — 요청 목적 분류 (규칙 기반, 필요 시 모델 기반)
6. **정책 엔진** — `(목적 × role × 엔티티 타입 × 누적 위험도)` → 엔티티별 조치 결정
   - 분류된 목적은 세션 스토어에도 기록한다(`remember_purpose_stage`). 출력 경로 detokenize 가
     이 값을 읽어 토큰 복원 인가를 판단한다(§3.2).
7. **동적 변환** — 결정된 조치 실행(마스킹/일반화/집계/토큰화/합성/삭제/차단), 본문 재조립
8. **감사 로그 기록**

### 3.2 출력 경로 스테이지 순서

1. **응답 파싱** — 어댑터가 응답 본문(SSE 조각 포함)을 조립
2. **Output Guard** — 모델이 새로 생성한 타인 PII 재스캔(재마스킹), 인젝션 순응 여부. 필요 시 `block`
3. **detokenize** — 요청자 role·목적이 토큰의 접근 범위를 통과하면 원본 복원.
   목적은 입력 경로가 세션에 기록해둔 값을 조회한다(§3.1)
4. **응답 본문 재조립**, 감사 로그 기록

재스캔이 detokenize 앞에 오는 이유: 토큰 라벨 `<PII:…>` 은 정규식에 안 걸리므로 이 시점에 잡히는
건 모델이 새로 만든 PII 뿐이다. detokenize 뒤에 재스캔하면 복원된 인가 PII 까지 다시 가려버린다.

입력 [3]과 출력 [2]는 **동일한 탐지 엔진을 재사용**하며, 임계값만 방향별로 다르게 준다.

---

## 4. 모듈 구조

```
dlp-server/
├── proto/
│   └── dlp.proto                 # Go 프록시와 공유하는 gRPC 계약 (공유 소스, 직접 수정 금지)
│
├── app/
│   ├── main.py                   # 앱 부트스트랩 (설정 로드, gRPC 서버 + HTTP API 동시 기동)
│   │
│   ├── proto/                    # protoc 생성물 (빌드 시 생성, VCS 제외)
│   │
│   ├── grpc_server.py            # 얇은 어댑터: proto ↔ 내부 타입 변환 + pipeline.analyze() 호출 + 예외 처리
│   ├── api.py                    # 얇은 어댑터: /health, /events(대시보드), 평가 스크립트 진입점
│   │
│   ├── pipeline.py               # [핵심] analyze() 단일 진입점 — gRPC · HTTP · 평가 스크립트가 모두 재사용
│   ├── models.py                 # [핵심] 계약 타입 (Turn / Span / AnalysisContext / Decision) + WIRE/TRANSFORM_ACTIONS 상수
│   ├── ids.py                    # session_id → 결정론적 UUID (감사·볼트 저장 계층 공용)
│   ├── config.py / config.yaml   # 런타임 설정 (분류기 backend, 임계값, 장애 정책 등)
│   │
│   ├── adapters/                 # 본문 형식 파서 (인프라). 입력·출력 양방향 재사용
│   │   ├── base.py               # Adapter 프로토콜 (matches / extract_turns / rebuild / parse_response / rebuild_response)
│   │   ├── gateway.py            # 사내 AI Gateway 포맷 {messages:[{role, content}]} — 1차 필수
│   │   ├── openai.py             # OpenAI 공식 API 포맷
│   │   └── anthropic.py          # Anthropic 공식 API 포맷
│   │
│   ├── detect/                   # 하이브리드 PII 탐지. 입력·출력 양방향 재사용
│   │   ├── regex_rules.py        # 정규식 / 체크섬 (주민번호, 카드 Luhn, 계좌, 전화, 이메일, 여권, 운전면허, 사업자번호)
│   │   ├── dictionary.py         # 사전 매칭 — Aho-Corasick (직원명단 / VIP / 내부 프로젝트명)
│   │   ├── ner.py                # GLiNER 제로샷 NER (gliner_multi-v2.1, CPU, Apache-2.0) — main.py 에서 preload
│   │   └── merge.py              # span 중복 제거 / 신뢰도 융합 / 우선순위 (Output Guard도 재사용)
│   │
│   ├── guardrail/
│   │   ├── injection.py          # Input Guard: 인젝션 / 탈옥 / 반출 요청 패턴 (교체 가능한 인터페이스, 규칙 기본)
│   │   └── output_check.py       # Output Guard: 타인 PII 재생성 / 인젝션 순응 / 정책 위반
│   │
│   ├── context/                  # 멀티턴 분석. 향후 행동 시퀀스 분석도 이 인프라 재사용
│   │   ├── store.py              # SessionStore 인터페이스 + InMemory(TTL 스윕). 데모 기본값
│   │   ├── accumulator.py        # 턴별 엔티티 누적 — 엔티티 그래프 + 슬라이딩 윈도우, 누적 위험도 가중
│   │   └── stage.py              # Stage 계약 어댑터 — multiturn_stage / remember_purpose_stage / get_last_purpose
│   │
│   ├── purpose/
│   │   ├── classifier.py         # 목적 분류 — 규칙 기반 기본 / 모델 기반 드롭인 (교체 가능한 인터페이스)
│   │   └── role_resolver.py      # 헤더에서 요청자 role 추출 → 정책 엔진 입력
│   │
│   ├── policy/
│   │   ├── engine.py             # (목적 × role × 엔티티) → 조치 평가. 동적 변환 계층이 결과 소비
│   │   └── policy.yaml           # 정책 매트릭스 + 위험도 오버라이드
│   │
│   ├── transform/                # 토큰화 + 동적 변환. Output Guard의 detokenize와 공유
│   │   ├── vault.py              # 결정론적 토큰화 / 복원 + 접근 범위(access scope) + TTL 파기
│   │   └── apply.py              # keep / mask / generalize / aggregate / tokenize / synthetic / redact / block
│   │
│   ├── agent_control/            # 후순위 — context/store 의 세션 이벤트 로그 재사용 예정
│   │   ├── mcp_proxy.py          # MCP tool 호출 가로채기 (스켈레톤)
│   │   └── sequence_analyzer.py  # tool call 시퀀스 이상행동 탐지 (스켈레톤)
│   │
│   └── logging/
│       └── events.py             # 구조화 감사 로그 — 모든 스테이지 판정이 수렴. PostgreSQL log_events sink
│
├── eval/
│   ├── run_eval.py               # baseline(정규식 즉시 차단) vs full(전체 파이프라인) 비교
│   └── datasets/{benign,attack,multiturn}/
│
└── tests/
    ├── test_detect.py           # 정탐 / 오탐 케이스표
    ├── test_vault.py            # 토큰화 왕복 + 접근 범위 게이팅
    ├── test_policy.py          # 목적 × role × 엔티티 전 조합
    ├── test_adapters.py       # 실제 페이로드 샘플
    ├── test_accumulator.py   # 누적 / 위험도 / 엔티티 그래프
    └── test_grpc.py         # 인메모리 gRPC 서버 + client (프록시 측 테스트와 대칭)
```

**모듈 설계 규칙**

- `pipeline.py`와 `models.py`는 여러 기능이 직접 import하는 공용 계약이다. 시그니처 변경은
  영향 범위가 넓으므로 먼저 합의한다.
- `context/store.py`는 멀티턴 분석용이지만, 향후 행동 시퀀스 분석이 tool call 이벤트를 같은
  타임라인에 기록해 재사용할 수 있도록 **세션 이벤트 로그의 공통 기반**으로 설계한다.
- 후순위 기능(MCP tool 통제, 행동 시퀀스 분석)은 현재 스켈레톤만 둔다.
- 교체 가능성이 있는 지점(목적 분류기, Input Guard)은 프로토콜/인터페이스 뒤에 두고 규칙 기반
  구현을 기본값으로 한다. 모델 기반 구현은 무중단 교체가 가능하도록 한다.

---

## 5. 계약 타입 (`app/models.py`)

구현보다 먼저 고정하는 타입이다.

```python
from dataclasses import dataclass

@dataclass
class Turn:
    role: str            # user | assistant | system
    text: str

@dataclass
class Span:
    type: str            # RRN | FOREIGN_RRN | CARD | ACCOUNT | PHONE | EMAIL
                         #  | PASSPORT | DRIVER | BIZNO | NAME | CREDIT_INFO | AMOUNT
    value: str
    start: int
    end: int
    confidence: float
    source: str          # regex | dict | ner

@dataclass
class InjectionVerdict:
    hit: bool
    score: float
    pattern: str | None

@dataclass
class AnalysisContext:                   # 스테이지 간 전달 객체 (탐지·판정 산출물 누적)
    session_id: str
    direction: str                      # input | output
    provider: str                       # gateway | openai | anthropic
    role: str | None                    # role_resolver 결과 (접근 제어 축). 파이프라인이 ctx 생성 시 채운다
    turns: list[Turn]
    new_turn_spans: list[Span]          # [3] 탐지
    accumulated: dict[str, list[Span]]  # [4] 멀티턴 — "이번 턴 span 을 타입별로 묶은 것" (구현 상세는 e 스펙 §3.6)
    risk_score: float                   # [4] 멀티턴. 0.0 ~ 1.0
    injection: InjectionVerdict         # [2] Input Guard
    blocked: bool                       # 스테이지가 차단 요청. _run_stages 가 이후 스테이지 스킵
    block_reason: dict | None           # blocked 근거. guardrail_hits 한 조각 형태 {"type": ...}
    purpose: str | None                 # [5] 목적 분류 결과 (purpose_ref 코드)
    purpose_confidence: float | None
    span_actions: list[tuple[Span, str]]  # [5] 결정 (span, action). action 은 TRANSFORM_ACTIONS 중 하나. g 가 실행

@dataclass
class Decision:                          # 판정 단계 산출물 → gRPC Verdict
    action: str                         # allow | block | transform
    transformed_body: bytes | None
    reason_obj: dict                    # 세부 변환 종류 · 근거 · 매칭 엔티티 요약 → Verdict.reason
```

`models.py` 는 상수 두 개도 노출한다: `WIRE_ACTIONS = ("allow", "block", "transform")`
(gRPC `Verdict.action`), `TRANSFORM_ACTIONS = ("keep", "mask", "generalize", "aggregate",
"tokenize", "synthetic", "redact", "block")` (`span_actions` 의 action 값, `action_type` ENUM 과 일치).

**어댑터 프로토콜** (`adapters/base.py`): `matches`, `extract_turns`, `rebuild`,
`parse_response`, `rebuild_response`. 본문 바이트에서 대화 메시지 배열을 꺼내고 다시 조립하는
계층이며, 없으면 파이프라인이 원본 JSON을 평문으로 오탐한다. 1차 구현은 사내 Gateway 어댑터
하나만 필수다.

---

## 6. 기능별 설계

### a. 가역적 토큰화 — `transform/vault.py`

- **결정론적 토큰:** `tokenize(session_id, type, value)`가 값의 해시를 기반으로 토큰을 만든다.
  같은 세션의 같은 값은 항상 같은 토큰이 된다. 토큰 라벨은 `<PII:RRN:1>`처럼 타입을 포함하는
  자체 서술 포맷으로 만들어, 모델이 토큰을 일반 텍스트로 변형하지 않도록 시스템 프롬프트 규칙과 결합한다.
- **복원 시 인가 검사:** vault 레코드의 접근 범위(허용 role / 목적 목록)를 요청자의 role·목적이
  통과해야 복원한다. 실패 시 토큰을 그대로 두고 근거에 기록한다. 복원 시도는 모두 감사 로그로 남긴다.
- **TTL·파기:** 세션 종료 또는 시간 경과 시 vault 레코드를 파기한다(개인정보 최소 보관). 만료 시
  즉시 복원을 막는 soft revoke를 먼저 적용하고, 유예 기간 후 암호문을 완전 삭제한다.
- 형식 보존 암호화(FPE)는 카드번호·전화번호처럼 형식이 고정된 데이터에 한해 이후 확장으로 둔다.

### b. 하이브리드 PII 탐지 — `detect/`

세 레이어를 병렬 실행하고 `merge.py`로 병합한다.

1. **정규식 / 체크섬:** 주민등록번호(생년월일 유효성 + 검증숫자 mod 11), 외국인등록번호,
   카드번호(Luhn), 계좌번호(`\d{10,14}` + "계좌/예금주" 문맥 가중), 전화
   (`01[016-9][-\s]?\d{3,4}[-\s]?\d{4}`), 이메일, 여권(`[A-Z]\d{8}`),
   운전면허(`\d{2}-\d{2}-\d{6}-\d{2}`), 사업자등록번호(`\d{3}-\d{2}-\d{5}`). 빠르고 정밀하다.
2. **사전 / 화이트리스트:** 정규식으로 못 잡는 사내 특화 용어(직원 명단, VIP 고객, 내부
   프로젝트명). Aho-Corasick 다중 패턴 매칭.
3. **NER (모델):** 사람 이름, 주소, 비정형 조직명, 비정형 신용정보(연체·대출잔액·신용등급 등).
   **GLiNER 제로샷 NER**(`urchade/gliner_multi-v2.1`, 다국어, Apache-2.0)를 CPU로 서빙한다.
   `app/main.py` 부트스트랩에서 모델을 preload 하고, 실패해도 부팅은 막지 않으며 첫 요청에서
   lazy-load 를 재시도한다. 모델명·threshold 는 `app/config.py` 의 `DetectConfig` 로 주입한다
   (`ner_model_name`, `ner_threshold` 기본 0.55). **라이선스 제약:** Apache-2.0 유지가 조건이라
   상용 전환 시에도 `gliner_ko`(CC-BY-NC-4.0)로 교체하지 않는다 — 대안은 §12.

**병합:** 동일 구간이 겹치면 우선순위(정규식 > 사전 > NER)로 신뢰도를 조정하고,
`(start, end, type, confidence, source)` 리스트를 컨텍스트에 저장한다.

### c. 양방향 가드레일 — `guardrail/`

- **Input Guard:** "이전 지시 무시" 류 프롬프트 인젝션, 탈옥 시도, 명시적 데이터 반출 요청을
  탐지한다. 규칙 기반이 기본이며 소형 분류 모델을 선택적으로 결합한다. 적중 시 파이프라인 초입에서
  즉시 `block`.
- **Output Guard:** 응답에 대해 ① 모델이 새로 생성한 타인 PII 재스캔(환각 포함), ② 인젝션
  순응 여부, ③ 정책 위반·기밀 유출 패턴을 검사한다. 탐지 엔진은 `b`를 재사용하되 임계값은 방향별로 둔다.

### d. AI Agent / MCP Tool 통제 (후순위) — `agent_control/`

dlp-server가 MCP 클라이언트와 MCP 서버 사이의 프록시를 겸한다. tool call의 입력·출력에
`b`/`c`/`f` 파이프라인을 동일하게 적용하고, tool별 최소 권한(어떤 목적·role이 어떤 tool을 호출
가능한지) 정책을 별도 관리한다. 현재는 스켈레톤만 두고 Agent 워크플로우 도입 후 착수한다.

### e. Multi-turn Context 분석 — `context/`

- **`store.py`:** `SessionStore` 인터페이스와 인메모리(TTL 스윕) 구현. 세션 단위로 지금까지
  노출된 엔티티(타입, 값 해시, 턴 번호)를 누적 기록한다. 세션 이벤트 로그 전반의 기반이다.
- **`accumulator.py`:** 턴별 고유 엔티티를 누적하고, **엔티티 그래프**(턴 간 언급된 엔티티 연결)와
  **슬라이딩 윈도우**(최근 N턴)를 함께 사용한다. 조합 위험도를 가중한다(예: 이름 + 주민번호가
  함께 누적되면 위험도 상승, 여기에 계좌번호가 더해지면 추가 상승). 턴 간격·속도 신호도 반영한다.
- **`stage.py`:** `pipeline.py` 의 `Stage` 계약에 맞춘 어댑터. 매 턴 `load → accumulate → save`
  후 `ctx.risk_score`·`ctx.accumulated` 를 갱신한다. `multiturn_stage` 자체는 `ctx.blocked` 를
  세팅하지 않고, 파이프라인이 `risk_score >= risk.hard_block`(기본 0.6)일 때 `block` 한다.
  분류된 목적을 세션에 기록하는 `remember_purpose_stage`(출력 경로 detokenize 가 조회)도 여기 있다.
- 세션 스토어는 데모에서 **인메모리(TTL 스윕)** 로 확정했다(§7.3, §8). 세션 만료 시
  `InMemorySessionStore` 의 `on_expire` 훅이 해당 세션 vault 레코드를 soft revoke 한다.

### f. 목적 기반 동적 데이터 접근 제어 — `purpose/`, `policy/`

- **목적 확보:** 규칙 기반 분류가 기본이다(예: "요약/번역/정리" → 문서 요약, "코드/함수/에러" →
  코딩 지원, "몇 명/평균/통계" → 데이터 분석, "고객/문의" → 고객 응대, 미매칭 → unknown으로 보수적
  처리). 모델 기반 분류기는 타임아웃 + 규칙 fallback + 세션 캐시를 갖춘 드롭인으로 두고 설정으로 전환한다.
- **role 축:** 헤더에서 해석한 요청자 role(예: 1선 상담원, 2선 조사역)을 정책 입력에 더한다.
  속성 기반 접근 제어(ABAC)로, `decide(purpose, role, entity_type, context_risk) -> action`을
  엔티티 단위로 개별 결정한다.
- **정책 엔진:** `(목적 × role × 엔티티 타입) → 조치` 매트릭스와 위험도 오버라이드를 평가하는
  경량 룰 엔진. 정책은 버전 관리한다. 외부 정책 엔진(OPA 등)은 이후 확장으로 둔다.

```yaml
# policy.yaml (초안)
purposes: [customer_support, doc_summarize, code_help, data_analysis, fraud_investigation, unknown]
rules:
  - {purpose: doc_summarize,       role: "*",      entity: "*",    action: tokenize}
  - {purpose: doc_summarize,       role: "*",      entity: CARD,   action: block}
  - {purpose: data_analysis,       role: "*",      entity: RRN,    action: generalize}   # 예: <AGE:30대><SEX:여>
  - {purpose: data_analysis,       role: "*",      entity: AMOUNT, action: aggregate}
  - {purpose: code_help,           role: "*",      entity: "*",    action: block}
  - {purpose: customer_support,    role: agent_l1, entity: PHONE,  action: mask}          # 뒤 4자리만
  - {purpose: fraud_investigation, role: agent_l2, entity: PHONE,  action: keep}
  - {purpose: unknown,             role: "*",      entity: "*",    action: tokenize}
defaults: {action: tokenize}
risk_overrides:
  - {when: "injection.hit",     action: block}
  - {when: "risk_score >= 0.8", action: block}
```

### g. 동적 데이터 변환 — `transform/apply.py`

정책 엔진의 결정을 실제로 실행하는 전략 계층이다. **같은 PII 타입이라도 목적·role·위험도 조합에
따라 다른 전략이 런타임에 선택**되는 것이 핵심이며, 정적 마스킹 규칙표가 아니다.

| 조치 | 설명 | 예 |
|---|---|---|
| `keep` | 원본 유지 | `홍길동` |
| `mask` | 비가역 부분 마스킹 | `홍*동` · `880101-*******` · `****-****-****-1234` |
| `generalize` | 범주로 일반화 | `35세` → `30대` · 주민번호 → `<AGE:30대><SEX:여>` |
| `aggregate` | 목록을 통계로 | 금액 목록 → 합계 / 평균 / 구간 |
| `tokenize` | 가역적 토큰 치환 (기능 a) | `<PII:RRN:1>` + vault 기록 |
| `synthetic` | 형식이 같은 가짜 값 | 실명 → 무작위 그럴듯한 이름 |
| `redact` | 완전 삭제 | `[삭제됨]` |
| `block` | 요청/필드 자체 거부 | 응답 없이 에러 |

`keep`을 제외한 조치는 전송 시 `transform`(또는 `block`)으로 직렬화되고, 세부 종류는 판정 근거에 담긴다.

### h. Agent 행동 시퀀스 분석 (후순위) — `agent_control/sequence_analyzer.py`

개별 tool call은 정상이어도 연속 호출 패턴 자체가 이상행동일 수 있다(예: "고객 검색 → 전체 export
→ 외부 URL 전송"). 상태 기계 또는 n-gram 기반 시퀀스 빈도 분석으로 모니터링한다. 기능 e의 세션
이벤트 로그 인프라에 tool call 이벤트를 같은 타임라인으로 기록해 구현 부담을 줄인다.

---

## 7. 데이터 저장소

### 7.1 3계층 데이터 모델

| 계층 | 테이블 | 수명 관리 | 다른 계층으로의 FK |
|---|---|---|---|
| **운영** | `sessions`, `session_turns`, `session_entities` | `expires_at` TTL 도달 시 하드 삭제 (CASCADE) | — |
| **볼트** | `token_vault` | 자체 `expires_at` (세션과 수명 분리) | 없음 |
| **감사** | `log_events`, `token_vault_access_log` | 장기 보존, append-only 지향 | 없음 (세션 삭제 후에도 보존) |

### 7.2 설계 결정

1. **감사·볼트 테이블은 운영 테이블로의 외래키를 두지 않는다.** `log_events`, `token_vault`,
   `token_vault_access_log`는 `session_id` / `token_id`를 값 컬럼으로만 갖는다. 세션 TTL 삭제가
   CASCADE로 감사 로그까지 지우는 문제를 원천 차단한다.
2. **볼트는 세션과 수명을 분리한다.** `token_vault`는 `sessions` 참조 없이 자체 `expires_at`으로만
   관리한다. 만료 시 `revoked_at`을 세팅해 즉시 복원을 막고(soft revoke), 유예 기간(예: 1일) 후
   암호문을 하드 삭제한다.
3. **엔티티 타입과 목적은 ENUM이 아니라 조회 테이블**(`entity_type_ref`, `purpose_ref`)로 둔다.
   목록이 확정 전이라 `ALTER TYPE` 마찰을 피하기 위함이다. 엔티티 민감도 등급(low/medium/high/
   critical)은 `entity_type_ref`에 함께 둔다.
4. **ENUM은 값 집합이 안정적인 것만 사용한다:** `direction_type`(input/output),
   `span_source`(regex/dict/ner), `verdict_action`(allow/block/transform — 프록시 계약과 일치),
   `action_type`(keep/mask/generalize/aggregate/tokenize/synthetic/redact/block).
5. **볼트 인덱스 두 종:** 입력 경로는 `UNIQUE (session_id, value_hash) WHERE revoked_at IS NULL`로
   결정론적 토큰 재사용을 강제하고, 출력 경로는 `(session_id, token_label)`로 응답에서 찾은 토큰
   라벨의 복원 조회를 지원한다.
6. **정책은 버전 테이블로 관리한다:** `policy_versions`(부분 유니크 인덱스로 활성 버전 1개 보장),
   `policy_rules`(`purpose`/`role`/`entity_type`이 NULL이면 와일드카드, `priority` 높을수록 우선),
   `policy_risk_overrides`. 오버라이드 조건식은 앱의 **화이트리스트 파서**로만 평가한다
   (`injection.hit`, `risk_score >= N` 두 종류). `eval()`을 절대 쓰지 않는다.
7. **감사 로그는 append-only를 지향한다.** 원문을 저장하지 않고 마스킹된 미리보기만 남긴다. 변환
   종류·엔티티 요약·가드레일 적중은 JSONB로 둔다. 운영 환경에서는 `UPDATE`/`DELETE` 권한을 회수하고
   `created_at` 월별 파티셔닝을 적용한다.
8. **TTL 정리:** `purge_expired()` 함수를 스케줄러(예: 5분 주기) 또는 앱 배치로 실행한다. 감사
   테이블은 건드리지 않는다.

### 7.3 구현 단계별 저장소

1차 구현부터 **볼트(`token_vault`) · 정책(`policy_*`) · 감사(`log_events`,
`token_vault_access_log`)는 PostgreSQL** 로 둔다(`postgres-schema.sql` 그대로).
볼트의 `cipher_value` 는 앱레벨 AES-GCM으로 암호화하며, 키는 dlp-server 설정(env)에 두고
저장소에는 두지 않는다. KMS 연동은 이후 확장이다.

**세션 스토어(운영 3테이블)는 데모에서 인메모리(TTL 스윕)로 확정** — `context/store.py` 의
`InMemorySessionStore`. `SessionState`(세션별 엔티티 그래프·슬라이딩 윈도우·누적 위험도·마지막
목적)는 프로세스 메모리에만 있고 재시작 시 사라진다. `sessions` / `session_turns` /
`session_entities` PostgreSQL 테이블은 스키마에는 있으나 현재 코드가 쓰지 않는다(다중 프로세스·
영속 배포로 갈 때의 확장 경로). `SessionStore` 프로토콜 뒤라 교체해도 상위 파이프라인은 영향
없다. 어느 쪽이든 볼트·감사는 세션과 FK가 없어 영향받지 않는다.

네 개 데이터 계약의 단일 기준은 [`../schemas/dlp-server/postgres-schema.sql`](../schemas/dlp-server/postgres-schema.sql)
이며(계약별 설명은 [`../schemas/`](../schemas/) 하위 문서).

---

## 8. 데모 범위와 프로덕션 확장

| 영역 | 데모 범위 (현재 구현) | 프로덕션 확장 방향 |
|---|---|---|
| 세션 스토어 | 인메모리(TTL 스윕) — **확정**. PG `sessions*` 테이블은 미사용 | 외부 인메모리 스토어(Redis 등) 또는 PG `sessions*` 배선 |
| 토큰 볼트 | PostgreSQL(`token_vault`) + 앱레벨 AES-GCM + 접근 범위 게이팅 | KMS 키 연동, 형식 보존 암호화(FPE) 병행 |
| 토큰화 방식 | 결정론적 플레이스홀더 `<PII:TYPE:n>` | 형식 보존 암호화(FPE) 병행 |
| 정책 엔진 | PostgreSQL(`policy_*`) + 경량 룰 평가기 (시드 `policy.yaml`) | 외부 정책 엔진(OPA/Rego), 관리자 API CRUD |
| 감사 로그 | PostgreSQL(`log_events`, `token_vault_access_log`) | append-only 파티셔닝 / 검색엔진 이관 |
| 비동기 보강 | 프로세스 내 asyncio | 백그라운드 큐(Celery/RQ) |
| NER 서빙 | CPU 추론 | 추론 런타임 최적화(ONNX 등) |
| 프록시 연동 | gRPC unary + 전체 버퍼링 | 양방향 스트리밍 청크 단위 검사 |
| 전송 계층 | HTTPS(TCP) | QUIC/HTTP3 대응 |

---

## 9. 구현 로드맵

파이프라인 코어(`analyze()`)부터 만들고, 전송 계층 껍데기는 마지막에 붙인다. 각 단계는 검증
마일스톤으로 끝난다.

> **진행 현황(2026-09 기준):** Phase 0~3 완료, Phase 4 대체로 완료, Phase 5 진행 중
> (`/events`·대시보드는 데모 0), Phase 6 미착수. 아래 각 Phase 머리에 상태를 표기.

### Phase 0 — 스캐폴딩
**상태: ✅ 완료** (#2)
레포 생성, `dlp.proto` 연결 및 코드 생성, `models.py` 계약 타입 확정,
`pipeline.analyze()` 스켈레톤(항상 `allow` 반환), gRPC 서버와 `/health`.
**검증:** 프록시가 붙어 `allow` 판정을 수신 → 관통 확인.

### Phase 1 — 탐지 + 토큰화 (입력 경로 E2E)
**상태: ✅ 완료** (#9 DB 인프라, #12 볼트, #15/#16 탐지 레이어, #20 변환)
PostgreSQL 스키마 적용(`postgres-schema.sql`), 사내 Gateway 어댑터, 정규식 탐지, 사전 탐지, 병합,
PostgreSQL 볼트(`token_vault`, 앱레벨 AES-GCM), 기본 변환(keep/mask/redact/tokenize).
파이프라인은 입력에서 `탐지 → 전부 토큰화 → 재조립 → transform 반환`.
**검증:** Gateway 요청의 주민번호가 토큰으로 치환되어 업스트림에 전달되고, 프록시가 치환 본문을 사용.

### Phase 2 — 세션 컨텍스트 + 멀티턴
**상태: ✅ 완료** — `multiturn_stage` 배선 + E2E 검증(e 스펙 §6). 세션 스토어는 인메모리 확정(§7.3)
`SessionStore`(인메모리 TTL)와 누적기(엔티티 누적, 위험도 가중). 매 턴 로드/누적/저장,
누적 위험도가 임계값을 넘으면 `block`.
**검증:** 이름·주민번호·계좌를 3턴에 나누어 입력 → 3턴째 누적 위험도로 차단.

### Phase 3 — 출력 경로 + 가드레일
**상태: ✅ 완료** (#13 Input Guard) — 출력 경로 `_OUTPUT_STAGES = [output_guard, detokenize_stage]`
Input Guard(패턴 룰)와 Output Guard(타인 PII 재생성 / 인젝션 순응 / 정책 위반). 출력 파이프라인은
`응답 파싱 → Output Guard(재스캔) → detokenize → 재조립`. 입력 파이프라인 초입에 인젝션 체크 추가.
**검증:** 인젝션 요청 차단, 응답의 토큰이 원본으로 복원되어 사용자에게 표시.

### Phase 4 — 목적 분석 + 정책 엔진
**상태: ✅ 대체로 완료** (#17/#18 목적·정책, #20 변환) — mask/redact/tokenize/generalize(RRN)/
aggregate/synthetic 동작. 남음: `access_scope` 정책화(`policy_rules` +restore 컬럼), span↔turn 매핑
규칙 기반 목적 분류기, role 해석기, PostgreSQL `policy_*` 테이블(시드 `policy.yaml`) + 정책 엔진,
변환에 generalize/aggregate 추가.
파이프라인은 `목적 → 정책 평가 → 엔티티별 조치 → 변환 실행`.
**검증:** 같은 PII라도 목적이 "요약"이면 토큰화, "코딩 지원"이면 차단.

### Phase 5 — 로깅 + 대시보드 + NER
**상태: 🚧 진행 중** — PostgreSQL `log_events` sink · GLiNER NER 통합·병합 편입 완료.
`/events` 엔드포인트와 관리자 대시보드는 데모 0 에서 착수.
구조화 감사 로그를 PostgreSQL `log_events` 로 정착(스테이지 판정 수렴), `/events` 엔드포인트
(대시보드 tail), 한국어 NER 통합 및 병합 편입, 임계값 튜닝.
**검증:** 대시보드에 세션별 탐지·목적·조치·지연이 실시간 표시되고 원문이 노출되지 않음.

### Phase 6 — 성능 평가 + 레드팀
**상태: ⬜ 미착수** — `eval/run_eval.py` 골격만 존재
평가 데이터셋 구성(정상 / 공격 / 멀티턴), `run_eval.py`가 두 모드를 실행 —
**baseline**(정규식 탐지 시 무조건 차단, 기존 DLP 모사)과 **full**(전체 파이프라인). 지표는
탐지율(Detection Rate), 오탐률(False Positive Rate), 평균·95퍼센타일 지연. baseline과 full을
비교하는 그래프를 산출.

---

## 10. 에러 · 성능 정책

- `pipeline.analyze` 전체를 예외 처리로 감싸 예외 시에도 유효한 판정을 반환한다(gRPC 오류를 던지지
  않는다). 기본값 `block`, 시연 안정용 `allow` 스위치.
- 개별 스테이지 타임아웃(예: 모델 기반 목적 분류 1.5초) 시 fallback 후 계속 진행한다.
- soft budget 2.5초(프록시 deadline 3초). 외부 모델 호출이 없는 로컬 파이프라인은 600ms 이내를 목표.
- 스트리밍 응답은 프록시가 전체 버퍼링하는 현재 구조를 전제한다. 청크 단위 검사는 범위 밖이다.

---

## 11. 테스트 전략

- **단위 테스트:** 탐지(정탐/오탐 케이스표), 볼트(토큰화 왕복 + 접근 범위 게이팅), 정책(목적 ×
  role × 엔티티 전 조합), 어댑터(실제 페이로드 샘플), 누적기(누적/위험도/엔티티 그래프).
- **통합 테스트:** 인메모리 gRPC 서버 + Python 클라이언트로 allow/block/transform/멀티턴/타임아웃
  시나리오. 프록시 측 검사 테스트와 대칭 구조로 맞춘다.
- **관통 테스트:** Phase 1과 4에서 실제 프록시 + dlp-server + mock 업스트림을 연결한 end-to-end 확인.
- **평가:** Phase 6 데이터셋으로 baseline과 full을 자동 측정.

---

## 12. 미결정 사항

| 항목 | 현재 방향 | 상태 |
|---|---|---|
| 전송 계층 최종안 (gRPC / REST) | 코어 `analyze()`부터. gRPC를 얇게 유지하는 방향, Phase 0 말에 확정 | [논의] |
| 모델 사용 범위 (목적 분류 / Input Guard / Output Guard) | 셋 다 **규칙 기반으로 확정** (모델 교체 seam 유지). NER 은 GLiNER(`gliner_multi-v2.1`) 사용 중 | [일부 확정] |
| NER 모델 라이선스 | `gliner_multi-v2.1` = Apache-2.0. 한국어 특화 `gliner_ko` 는 CC-BY-NC-4.0 (상용 불가) → 교체하지 않고, 필요 시 자체 파인튜닝 | [확정] |
| provider 페이로드 변동 대응 | 실서비스 웹 UI 대신 공식 API 포맷과 사내 Gateway 포맷을 우선 타깃 | [논의] |
| 목적 확보 경로 | 명시 파라미터가 MITM 경로에 맞지 않음 → Gateway가 목적·role 힌트를 헤더로 주입하는 전제 필요 | [논의] |
| 엔티티 타입 / 목적 최종 목록 | 조회 테이블 seed로 시작, 확정 시 행 추가. 코드와 DB 중 하나를 단일 기준으로 | [미정] |
| 외부 세션 스토어(Redis 등) 도입 여부 | 데모는 **인메모리(TTL 스윕)로 확정**(§7.3). Redis/PG `sessions*` 는 다중 프로세스·영속 배포 시 | [데모 확정] |
| 토큰 접근 범위(access scope)의 정책 소스 | vault 레코드에 고정할지, 복원 시점에 정책 엔진이 재평가할지 | [미정] |
| 스트리밍 홀드 범위 / QUIC 대응 | 현재 범위 밖, 확장 방향으로만 기록 | [미정] |

---

## 부록 A. 데이터 저장소 스키마

네 개 데이터 계약(session-context / token-vault / policy / log-event)의 표준 정의는 이 문서가 아니라
스키마 폴더에 있다.

- 실행 가능한 DDL 원본(SSOT): [`../schemas/dlp-server/postgres-schema.sql`](../schemas/dlp-server/postgres-schema.sql)
- 계약별 설명·불변식·엣지케이스:
  - [`../schemas/dlp-server/session-context.md`](../schemas/dlp-server/session-context.md)
  - [`../schemas/dlp-server/token-vault.md`](../schemas/dlp-server/token-vault.md)
  - [`../schemas/dlp-server/policy.md`](../schemas/dlp-server/policy.md)
  - [`../schemas/dlp-server/log-event.md`](../schemas/dlp-server/log-event.md)
- 폴더 개요: [`../schemas/schema-index.md`](../schemas/schema-index.md)

§7의 3계층 모델(운영 / 볼트 / 감사)과 8개 설계 결정이 그 스키마의 근거다. 스키마를 바꿀 때는 §7과
위 문서를 함께 갱신한다.
