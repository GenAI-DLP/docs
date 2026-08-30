# 하이브리드 PII 탐지

우선순위: 필수
한 줄 정의: 정규식/체크섬 + 사전(Aho-Corasick) + NER 세 레이어를 병렬 실행하고 병합해 정형·비정형 금융 PII를 동시 탐지한다.
담당 모듈: `detect/regex_rules.py`, `detect/dictionary.py`, `detect/ner.py`, `detect/merge.py`

설계 의도·전체 맥락: `../../architecture/dlp-server-architecture.md` §6
관련 스키마: `../../schemas/dlp-server/session-context.md`
색인: `../spec-index.md`

---

## 1. 입력 예시

세 레이어 모두 `pipeline.analyze()`가 전달하는 `AnalysisContext` (혹은 그 안의 `Turn` 리스트)를 입력으로 받는다. 레이어별로 필요한 필드만 꺼내 쓰되, 계약은 동일하다.

```jsonc
// AnalysisContext (요약)
{
  "session_id": "sess_01H...",
  "turns": [
    {
      "turn_id": "t1",
      "role": "user",
      "text": "제 계좌번호는 110-234-567890이고, 카드번호 4111-1111-1111-1111 로 결제해주세요.",
      "adapter": "openai"   // gateway / openai / anthropic
    }
  ]
}
```

- 텍스트는 `app/adapters/`에서 이미 정규화된 본문(파서 통과 후)이라고 가정한다.
- 레이어는 `Turn.text` 단위로 동작하며, 여러 턴이 있으면 턴별로 독립 실행 후 병합 단계에서 합친다.

## 2. 출력 예시

세 레이어 모두 `Span` 리스트를 반환한다. `Span`에 레이어 출처(`source`)와 신뢰도(`confidence`)를 실어 `merge.py`가 우선순위를 계산할 수 있게 한다.

```jsonc
// regex_rules.py 출력 예시
[
  {
    "turn_id": "t1",
    "start": 8, "end": 21,
    "text": "110-234-567890",
    "label": "BANK_ACCOUNT",
    "source": "regex",
    "confidence": 0.99,
    "meta": { "rule_id": "kr_bank_account_v1", "checksum_passed": true }
  },
  {
    "turn_id": "t1",
    "start": 30, "end": 49,
    "text": "4111-1111-1111-1111",
    "label": "CARD_NUMBER",
    "source": "regex",
    "confidence": 0.99,
    "meta": { "rule_id": "card_luhn_v1", "checksum_passed": true }
  }
]
```

```jsonc
// merge.py 최종 출력 (Decision에 실리는 형태)
{
  "spans": [
    {
      "start": 8, "end": 21, "text": "110-234-567890",
      "label": "BANK_ACCOUNT", "confidence": 0.99,
      "sources": ["regex"], "merged": false
    },
    {
      "start": 30, "end": 49, "text": "4111-1111-1111-1111",
      "label": "CARD_NUMBER", "confidence": 0.995,
      "sources": ["regex", "ner"], "merged": true
    }
  ]
}
```

- `merged: true`는 2개 이상 레이어가 겹치는 구간에서 동일/유사 라벨로 hit한 경우.
- `sources`는 어느 레이어가 기여했는지 감사(audit) 로그용으로 남긴다.

## 3. 판정 로직 (룰 / 프롬프트 / 임계값)

### 3.1 레이어별 판정

| 레이어 | 방식 | hit 조건 |
|---|---|---|
| `regex_rules.py` | 정규식 + 체크섬 | 정규식 매치 + 체크섬 통과(카드 Luhn, 주민번호 체크디짓 등) 시에만 confidence 0.95 이상. 체크섬 실패 시 별도 낮은 confidence(예 0.4)로 내려 병합 단계에서 걸러지게 함 |
| `dictionary.py` | Aho-Corasick 다중 패턴 매칭 | 사전 등재어(은행명, 금융상품명, 내부 코드명 등) 매치 시 hit. confidence는 사전 항목별 고정값(설정 파일에서 관리) |
| `ner.py` | NER 모델 | 모델 스코어 ≥ threshold(§4) 시 hit. 라벨은 모델 출력 그대로(PERSON, ADDRESS 등)를 내부 라벨셋으로 매핑 |

### 3.2 병합 로직 (`merge.py`)

1. **구간 겹침 판정**: 두 `Span`의 `[start, end)`가 겹치면 병합 후보.
2. **라벨 일치 시**: confidence는 `max(a, b)`가 아니라 레이어 수에 따른 보정(`base + 0.02 * (n_sources - 1)`, 상한 0.999)으로 소폭 상향 — 다중 레이어 합의는 신뢰도를 높인다.
3. **라벨 불일치 시**: `regex` > `dictionary` > `ner` 우선순위로 라벨 채택하되, `meta`에 충돌 기록을 남긴다 (추후 튜닝용).
4. **겹치지 않는 단일 레이어 hit**: 레이어별 최소 confidence 미만이면 드롭, 이상이면 그대로 통과.
5. 최종적으로 `AnalysisContext` 전체에서 하나라도 hit이 남으면 `Decision`에 반영 — 실제 block/transform 여부는 `policy/` 단계(구현 예정)에서 결정하며, 이 모듈은 **탐지까지만** 책임진다.

> `guardrail/injection.py`처럼 이 모듈 자체가 block을 내리지 않는다. `Span` 탐지 결과를 넘기는 것까지가 책임 범위.

## 4. 파라미터 · 설정값

기존 설정 네이밍 컨벤션(`DLP_GUARDRAIL__INJECTION_THRESHOLD` 등)을 따른다.

| 키 | 의미 | 기본값 |
|---|---|---|
| `DLP_DETECT__NER_THRESHOLD` | NER 레이어 hit 판정 최소 스코어 (0~1) | 0.7 |
| `DLP_DETECT__REGEX_MIN_CONFIDENCE` | 체크섬 실패 등으로 낮아진 regex 결과의 통과 최소 confidence | 0.5 |
| `DLP_DETECT__DICTIONARY_PATH` | 사전 파일 경로 (Aho-Corasick 빌드 대상) | `app/detect/dictionaries/financial_terms.txt` |
| `DLP_DETECT__MERGE_OVERLAP_BONUS` | 다중 레이어 합의 시 confidence 가산치 | 0.02 |
| `DLP_DETECT__ENABLED_LAYERS` | 활성화할 레이어 목록 (테스트/부분 배포용) | `regex,dictionary,ner` |

- 모두 `app/config.py` / `config.yaml`의 `detect` 섹션 아래 중첩 필드로 들어간다.
- NER 모델 자체(모델명, 로딩 경로 등)는 별도 §4.1로 추후 채움 (모델 선정 미정).

## 5. 엣지 케이스 / 테스트 케이스표

| # | 케이스 | 기대 동작 |
|---|---|---|
| 1 | 카드번호 형식이지만 Luhn 체크섬 실패 | regex confidence 하향(0.4) → 다른 레이어 hit 없으면 드롭 |
| 2 | 계좌번호가 문장 중간에 다른 숫자와 붙어 있음 (예: "110-234-567890원 입금") | 정규식 boundary 처리로 숫자 뒤 단위(원) 오탐 방지 |
| 3 | NER이 사람 이름을 오탐 (일반 명사 대문자 등) | threshold 미달 시 드롭, 사전에 없는 단어는 dictionary가 보완하지 않음 → 로그로만 남기고 오탐 리뷰 대상 |
| 4 | 동일 구간에서 regex(BANK_ACCOUNT) vs ner(PERSON) 라벨 충돌 | regex 우선 채택 + meta에 충돌 기록 |
| 5 | 텍스트가 여러 턴에 걸쳐 PII가 쪼개짐 (예: 앞턴 "계좌번호는", 뒷턴 "110-234-567890") | 턴 단위 독립 처리가 기본 — 턴 간 결합 탐지는 범위 밖(향후 검토 항목으로 별도 기재) |
| 6 | 빈 텍스트 / 공백만 있는 턴 | 모든 레이어 즉시 빈 리스트 반환, 에러 없이 스킵 |
| 7 | 사전에 없는 신조어성 금융 상품명 | dictionary 미탐 → NER/정규식도 대상 아님 → 탐지 실패를 알려진 한계로 명시 (사전 업데이트 프로세스 필요) |
| 8 | `DLP_DETECT__ENABLED_LAYERS`에서 ner 제외 설정 | ner.py 호출 자체를 스킵, merge는 regex+dictionary 결과만으로 동작 |

---

### 미정 / 추후 결정 필요

- NER 모델 선정 (온프레미스 vs API, 한국어 금융 도메인 파인튜닝 여부)
- 턴 간(멀티턴) PII 결합 탐지 범위 포함 여부
- 사전 업데이트 운영 프로세스 (누가, 얼마나 자주)