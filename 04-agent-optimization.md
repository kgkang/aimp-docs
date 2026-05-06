# Agent 비용 및 성능 최적화

## 1. 모델 선택

### 현황
전 에이전트 모델: `global.anthropic.claude-haiku-4-5-20251001-v1:0`  
변경일: 2026-05-05 (Sonnet 4.5 → Haiku 4.5)

### 비용 비교 (Bedrock On-Demand, Standard 티어)

| 모델 | Input / 1M tokens | Output / 1M tokens | 비고 |
|------|:-----------------:|:------------------:|------|
| Claude Sonnet 4.5 (변경 전) | $3.00 | $15.00 | — |
| Claude Sonnet 4 (APAC) | $3.00 | $15.00 | Sonnet 4.5와 단가 동일, 절감 없음 |
| **Claude Haiku 4.5** (현재) | **$0.80** | **$4.00** | **약 73% 절감** |

> `global.` 크로스 리전 프로파일은 in-region 대비 약 18% 할증. 세 모델 모두 동일 방식이므로 상대 비율에는 영향 없음.

**실측 기반 추정** (5PM 정기 보고 로그, ~150K input / ~6K output):

| | Sonnet 4.5 | Haiku 4.5 |
|--|:----------:|:---------:|
| 1회 실행 비용 | ~$0.54 | ~$0.14 |
| 24회/일 | ~$13.0 | ~$3.5 |
| 30일 | ~$390 | ~$104 |
| 4일 실측 비용 | $74.90 | ~$20 (예상) |

---

## 2. 크로스 리전 지연 시간

### ap-northeast-2(서울) Haiku 4.5 가용 옵션

| 옵션 | 가용 여부 |
|------|:---------:|
| In-Region | ❌ |
| Geo Cross-Region (APAC) | ❌ |
| **Global Cross-Region** (`global.`) | ✅ (유일한 선택지) |

> Haiku 4.5 APAC Geo 프로파일은 AU(시드니·멜버른)만 존재. 서울에서는 Global이 유일.

### 목적지 리전별 지연 추정

`global.` 프로파일은 부하 상황에 따라 목적지를 동적으로 선택합니다.

| 목적지 리전 | 왕복 레이턴시 | TTFT 추가 오버헤드 |
|------------|:------------:|:-----------------:|
| ap-northeast-1 (도쿄) | ~12ms | ~100ms |
| ap-southeast-1 (싱가포르) | ~75ms | ~150ms |
| us-east-1 (버지니아) | ~180ms | ~300–500ms |
| eu-west-1 (아일랜드) | ~270ms | ~500ms+ |

**이 에이전트에서의 실질적 영향:**  
22회 LLM 호출 × 200–500ms = 총 4–11초 추가. 전체 실행 ~5분 대비 1–4% 수준으로 무시 가능.

### 개선 방법

| 방법 | 효과 | 비고 |
|------|------|------|
| 도쿄 In-Region 사용 (`ap-northeast-1`) | 레이턴시 최소화 | AgentCore는 서울에 있으므로 서울→도쿄 모델 호출 구조 |
| 서울 In-Region 출시 대기 | 근본 해결 | AWS가 순차 확대 중 |
| **Prompt Caching** | TTFT 단축 + 비용 절감 | 즉시 적용 가능, 아래 참조 |
| 현상 유지 | 조치 불필요 | 실행 시간 대부분이 AWS API 호출 |

---

## 3. Prompt Caching

### 동작 원리

LLM은 매 호출마다 프롬프트 전체를 처음부터 재처리합니다. Cache Checkpoint는 **"여기까지는 다음 호출에서 다시 계산하지 말고 저장해둬"** 라고 표시하는 경계선입니다.

```
[system prompt] [tools 정의] │← 체크포인트  [메시지 히스토리] │← 체크포인트  [새 메시지]
  정적, 변경 없음            │              누적 히스토리     │

첫 호출:    전체 처리 → 체크포인트 이전 토큰을 캐시 WRITE
이후 호출:  체크포인트 이전은 캐시 READ (건너뜀) → 정적 구간 처리 비용 0
```

### Haiku 4.5 캐시 스펙

| 항목 | 값 |
|------|----|
| 최소 토큰 수 (체크포인트당) | 1,024 tokens |
| 최대 체크포인트 수 | 4개 |
| 기본 TTL | 5분 |
| **확장 TTL (Haiku 4.5 지원)** | **1시간** |
| 캐시 가능 필드 | `system`, `tools`, `messages` |

### 이 에이전트의 캐시 대상 토큰

```
system prompt:  ~80 tokens  (날짜 포함 고정 텍스트)
tools 정의:
  get_cost_summary              ~200 tokens
  list_idle_resources           ~150 tokens
  get_cloudwatch_metrics_bulk   ~200 tokens
  list_resources_by_age         ~180 tokens
  get_lambda_invocation_stats   ~150 tokens
  use_aws (strands_tools)       ~500 tokens
  ────────────────────────────────────────
  합계:                       ~1,380 tokens

system + tools 합계:  ~1,460 tokens  → 1,024 최소 요건 충족 ✓
```

### 22회 도구 호출 세션의 캐시 적용 흐름

```
Call  1: [system+tools 1,460t] 처리 + 캐시 WRITE
Call  2: [⚡캐시 READ 1,460t] + 누적 히스토리 처리
Call  3: [⚡캐시 READ 1,460t] + 누적 히스토리 처리
  ...
Call 22: [⚡캐시 READ 1,460t] + 누적 히스토리 처리
         └─ 21회 × 1,460t = 30,660 tokens 절감
```

### 비용 절감 계산

**Haiku 4.5 토큰 단가:**

| 유형 | 단가 / 1M tokens |
|------|:---------------:|
| 일반 Input | $0.80 |
| Cache Write | $1.00 (+25%) |
| **Cache Read** | **$0.08 (−90%)** |
| Output | $4.00 |

**1회 실행 기준, system+tools 구간(~1,460 tokens × 22 calls):**

| | 처리 방식 | 비용 |
|--|-----------|:----:|
| 캐시 없음 | 22 × 1,460t × $0.80/MTok | $0.0257 |
| 캐시 적용 | 1 Write + 21 Read | $0.0040 |
| **절감** | | **$0.0217 (−85%)** |

**1시간 TTL 적용 시, 정기 보고 하루(24회 실행) 기준:**

같은 날에는 system prompt의 날짜가 동일 → 실행 간 캐시 유지.

| | 비용 |
|--|:----:|
| 하루 합계 (캐시 없음) | 24 × $0.026 ≈ **$0.62** |
| 하루 합계 (1h TTL 캐시) | Write 1회 + Read 23회 ≈ **$0.010** |
| **일간 절감** | **~$0.61 (−98%)** ← system+tools 구간 |

> 전체 비용에서 system+tools 비중이 크지 않으므로 실제 총 절감액은 이보다 작음.  
> 입력 토큰 중 누적 히스토리(tool results 등)가 대부분을 차지하며, 이 부분은 매 호출마다 변하므로 캐싱 불가.

### 주의 사항

**자정 캐시 미스:**  
system prompt에 `Today's date is {today} (KST)`가 포함되어 있어 날짜가 바뀌면 cache miss → 당일 첫 실행은 반드시 cache WRITE.

**Cross-Region 캐시 미스:**  
`global.` 프로파일은 목적지 리전이 동적으로 결정됩니다. 연속 실행 사이에 라우팅 목적지가 바뀌면 cache miss. 문서에도 "cross-region inference may lead to increased cache writes"라고 명시.

---

## 4. 구현 방법

### Prompt Caching 활성화

`cache_prompt="default"` 한 줄 추가로 Strands SDK가 system + tools 뒤에 자동으로 cache checkpoint를 삽입합니다.

**`agent/app/SlackStrandAgent/model/load.py`:**
```python
from strands.models.bedrock import BedrockModel


def load_model() -> BedrockModel:
    return BedrockModel(
        model_id="global.anthropic.claude-haiku-4-5-20251001-v1:0",
        cache_prompt="default",  # system + tools 자동 캐시 체크포인트
    )
```

1시간 TTL을 명시적으로 설정하려면 (정기 보고처럼 1시간 간격 실행에 유리):
```python
return BedrockModel(
    model_id="global.anthropic.claude-haiku-4-5-20251001-v1:0",
    cache_prompt="default",
    additional_model_request_fields={"cache_control": {"type": "ephemeral", "ttl": "1h"}},
)
```

> `SlackFormatAgent`, `SlackAws`도 동일하게 적용 가능.

### 캐시 히트 확인 방법

CloudWatch Logs에서 Bedrock 응답의 `usage` 필드를 확인합니다:
```json
{
  "usage": {
    "inputTokens": 150,
    "outputTokens": 45,
    "cacheReadInputTokens": 1460,   // 캐시에서 읽은 토큰 수
    "cacheWriteInputTokens": 0      // 캐시에 쓴 토큰 수 (첫 호출만 > 0)
  }
}
```

---

## 5. 툴 반환값에 소유자 태그 포함 (use_aws 호출 제거)

### 문제 배경

`scheduled_reporter` 프롬프트는 각 리소스의 소유자(`cz-owner` / `Creator` 태그)를 보고서에 포함하도록 요구합니다.  
기존 커스텀 툴은 리소스 목록만 반환하고 태그를 포함하지 않았기 때문에, LLM이 매 실행마다 `use_aws` fallback 툴로 태그를 별도 조회했습니다.

### 문제 분석 — 2026-05-05 23:00 실측 (개선 전)

CloudWatch bedrock-logging 로그에서 추출한 `use_aws` 호출 패턴:

| LLM 호출 | use_aws 횟수 | 실제 동작 | 결과 |
|---------|------------|---------|------|
| #3 | 3회 | `ec2.describe_tags` (EC2/EBS), `elasticloadbalancing.describe_tags` (실패), `ec2.describe_nat_gateways` (태그용) | 서비스명 오류로 재시도 발생 |
| #3 | 1회 | `lambda.list_tags` | 성공 |
| #4 | 3회 | `elasticloadbalancing.describe_load_balancers` (재시도, 실패), `ec2.describe_nat_gateways` (배치 2), `cloudwatch.get_metric_statistics` | 2회 실패 |
| #5 | 2회 | `elbv2.describe_tags` (성공), `ec2.describe_nat_gateways` (배치 3) | 성공 |
| #6 | 1회 | `elb.describe_tags` (Classic ELB) | 성공 |

**핵심 원인 3가지:**
1. **소유자 태그 미포함** — 툴 결과에 `owner` 없음 → 모든 리소스 유형별 별도 조회 (8회)
2. **ELB 서비스명 시행착오** — `elasticloadbalancing` → 실패 → `elbv2` → 성공 (2회 낭비, LLM 호출 1회 추가)
3. **NAT GW 태그를 8개씩 3배치** — `describe_nat_gateways`를 3번 나누어 호출 (3회)

### 적용된 변경 (2026-05-06)

**`tools/idle_resources.py`:**

```python
def _owner(tags: list[dict]) -> str:
    for key in ("cz-owner", "Creator"):
        val = next((t["Value"] for t in tags if t["Key"] == key), None)
        if val:
            return val
    return "확인 불가"

def _batch_elbv2_tags(elbv2_client, arns):
    """elbv2.describe_tags — 20개씩 배치."""
    ...

def _batch_elb_tags(elb_client, names):
    """elb.describe_tags — 20개씩 배치."""
    ...
```

| 리소스 | 변경 내용 |
|--------|---------|
| EBS, EIP, NAT GW | `describe_*` 응답에 Tags 포함 → `owner` 직접 추출 (추가 API 호출 없음) |
| ALB/NLB | `_batch_elbv2_tags()` 추가 — 전체 ARN을 배치 1회로 조회 |
| Classic ELB | `_batch_elb_tags()` 추가 — 전체 이름을 배치 1회로 조회 |

**`tools/resource_age.py`:**

```python
def _fetch_lambda_owners(region: str) -> dict[str, str]:
    """Resource Groups Tagging API로 Lambda 전체 태그 일괄 조회."""
    tagging = boto3.client("resourcegroupstaggingapi", region_name=region)
    ...
```

| 리소스 | 변경 내용 |
|--------|---------|
| EC2 | `describe_instances` 응답의 `Tags` → `owner` 직접 추출 |
| RDS | `describe_db_instances` 응답의 `TagList` → `owner` 직접 추출 |
| Lambda | `_fetch_lambda_owners()` 추가 — Tagging API로 리전 전체 Lambda 태그 1회 일괄 조회 |

### 실측 결과 비교 — 2026-05-05 23:00 vs 2026-05-06 00:22

| 지표 | 개선 전 (23:00) | 개선 후 (00:22) | 변화 |
|------|:--------------:|:--------------:|:----:|
| LLM 호출 횟수 | 7회 | **4회** | −43% |
| 총 툴 호출 | 17회 | **6회** | −65% |
| `use_aws` 호출 | 10회 | **0회** | −100% |
| Input tokens | 167,853 | **59,538** | −65% |
| Output tokens | 8,557 | 7,540 | −12% |
| 총 tokens | 176,410 | **67,078** | −62% |
| 실행 시간 | 81.9초 | **71.3초** | −13% |
| 실행 비용 | $0.169 | **$0.078** | −54% |

### 추가된 IAM 권한

| 권한 | 용도 |
|------|------|
| `elasticloadbalancing:DescribeTags` | ALB/NLB/Classic ELB 소유자 태그 배치 조회 |
| `tag:GetResources` | Resource Groups Tagging API — Lambda 함수 태그 일괄 조회 |

---

## 6. 최적화 우선순위 요약

| 우선순위 | 최적화 항목 | 효과 | 구현 난이도 |
|:-------:|------------|------|:-----------:|
| ✅ 완료 | Sonnet 4.5 → Haiku 4.5 | 비용 ~73% 절감 | 낮음 (1줄) |
| ✅ 완료 | 툴 반환값에 owner 태그 포함 | `use_aws` 제거, input token −65%, 비용 −54% | 중간 (툴 수정) |
| 🔲 권장 | Prompt Caching (5min TTL) | 세션 내 system+tools 85% 절감 | 낮음 (1줄) |
| 🔲 선택 | Prompt Caching (1h TTL) | 실행 간 캐시 유지 | 낮음 (설정 추가) |
| 🔲 대기 | 서울 In-Region 출시 | 레이턴시 근본 해결 | 없음 (출시 대기) |
| 🔲 선택 | 도쿄 In-Region 사용 | 레이턴시 소폭 개선 | 중간 (리전 변경) |
