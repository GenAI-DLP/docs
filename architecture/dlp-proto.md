# dlp-proto — Proxy ↔ dlp-server 인터페이스 계약

Go 프록시(`dlp-proxy-server`)와 검사 백엔드(`dlp-server`) 사이의 gRPC 계약을 정의한다.
**이 문서와 실제 `.proto` 파일이 두 컴포넌트의 단일 기준이다.** 어느 한쪽이 임의로 필드를 추가·변경하지
않으며, 변경은 양 팀 합의 + 프록시 재조율을 전제로 한다. `.proto` 원본은 공유 소스(공용 서브모듈
`dlp-proto` 등)로 관리한다.

상태: `Inspect` RPC·`InspectRequest`·`Verdict` 형태와 `action` 3값 규칙은 **[확정]**.
확장(스트리밍 등)은 현재 범위 밖.

---

## 1. 서비스 정의

```protobuf
syntax = "proto3";

service DLPInspector {
  // 요청/응답 각각 1회씩 호출되는 unary RPC. 스트리밍 아님.
  rpc Inspect (InspectRequest) returns (Verdict);
}

message InspectRequest {
  string session_id = 1;
  string direction  = 2;   // "input" | "output"
  string method     = 3;   // HTTP 메서드 (예: "POST")
  string path       = 4;   // HTTP 경로 (예: "/v1/chat/completions")
  map<string, string> headers = 5;
  bytes  body       = 6;   // 평문 본문 (프록시가 TLS 종단 후 추출)
}

message Verdict {
  string action           = 1;   // "allow" | "block" | "transform"
  bytes  transformed_body = 2;    // action == "transform" 일 때만 유효
  string reason           = 3;    // 구조화 사유 (JSON 문자열)
}
```

---

## 2. 필드 의미

### 2.1 `InspectRequest`

| 필드 | 의미 | 비고 |
|---|---|---|
| `session_id` | 멀티턴 분석의 상관관계 키 | 프록시가 `X-Corp-User-Id` 헤더 → 서비스 세션 쿠키 → 원격 주소 순으로 결정. dlp-server는 이 값을 그대로 세션 스토어 키로 사용 |
| `direction` | `"input"` = 직원 PC → 외부 LLM 요청, `"output"` = 외부 LLM → 직원 PC 응답 | 같은 대화 턴에 대해 프록시가 input 1회, output 1회 호출 |
| `method`, `path` | 원 HTTP 요청/응답의 메서드·경로 | 어댑터 선택과 로깅에 사용 |
| `headers` | 원 HTTP 헤더 맵 | dlp-server가 여기서 요청자 role을 해석(`role_resolver`). 인증 토큰 등 민감 헤더는 로그에 남기지 않는다 |
| `body` | 평문 본문 바이트 | 프록시가 MITM으로 복호화한 결과. dlp-server의 어댑터가 여기서 대화 메시지 배열을 파싱 |

### 2.2 `Verdict`

| 필드 | 의미 |
|---|---|
| `action` | `"allow"` 원본 그대로 통과 · `"block"` 요청/응답 거부 · `"transform"` `transformed_body`로 치환해 통과 |
| `transformed_body` | `action == "transform"` 일 때 프록시가 실제로 전달할 본문. 그 외에는 빈 값 |
| `reason` | dlp-server의 판정 근거 객체를 JSON 직렬화한 문자열. 관리자 대시보드·성능 평가 스크립트가 파싱 |

---

## 3. 계약 규칙

### 3.1 `action`은 3값 고정

`action` enum을 **확장하지 않는다.** dlp-server 내부의 세부 변환 전략
(`mask` / `generalize` / `aggregate` / `tokenize` / `synthetic` / `redact`)은 전송 시 모두 `transform`
하나로 직렬화하고, 세부 종류와 근거는 `reason`(JSON)에 담는다. `block`은 그대로 `block`.

- 프록시는 `allow` / `block` / `transform` 세 갈래만 구현하면 된다.
- 새 전략이 생겨도 wire 계약과 프록시 코드는 그대로다.

### 3.2 `reason` JSON 형태 (권장 필드)

`reason`은 자유 형식 JSON이지만 대시보드/eval 이 아래 키를 기대한다.

```json
{
  "verdict": "transform",
  "transforms": [
    {"entity": "RRN", "action": "tokenize", "token_label": "<PII:RRN:1>"},
    {"entity": "PHONE", "action": "mask"}
  ],
  "entities_summary": [
    {"type": "RRN", "masked_preview": "8801**-*******", "confidence": 0.99}
  ],
  "purpose": "doc_summarize",
  "risk_score": 0.42,
  "guardrail_hits": [],
  "fail_policy_applied": false
}
```

원문은 절대 담지 않는다(마스킹된 미리보기만). 로그 이벤트 스키마는
[`../schemas/dlp-server/log-event.md`](../schemas/dlp-server/log-event.md) 참조.

### 3.3 호출 모델

- **unary.** 요청·응답 각각에 대해 프록시가 `Inspect`를 1회 호출하고 판정을 받을 때까지 트래픽을
  Hold 한다.
- 스트리밍 응답(SSE 등)은 프록시가 **전체 버퍼링** 후 한 번에 `body`로 넘긴다. 청크 단위 검사(양방향
  스트리밍)는 프로토콜·프록시를 모두 바꿔야 하므로 현재 범위 밖.

### 3.4 에러 · 타임아웃

- 프록시 호출 deadline **3초**. dlp-server soft budget **2.5초**.
- dlp-server는 내부 오류 시에도 **유효한 `Verdict`를 반환**한다 (gRPC 에러를 던지지 않는다).
  기본값 `block`, 시연 안정용 `allow` 스위치.
- dlp-server가 타임아웃/연결 실패로 응답하지 않으면 **프록시가 fail-closed(차단)** 하고, 로그에
  "DLP 판정이 아닌 장애 대응"임을 구분하는 플래그(`fail_policy_applied`)를 남긴다.

---

## 4. 관련 문서

- 파이프라인·모듈 구조: [`dlp-server-architecture.md`](dlp-server-architecture.md) §2~§5
- 프록시 측 구현: [`dlp-proxy-server-architecture.md`](dlp-proxy-server-architecture.md)
- 로그 이벤트 스키마: [`../schemas/dlp-server/log-event.md`](../schemas/dlp-server/log-event.md)
