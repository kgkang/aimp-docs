# AgentCore Agent 명세 (SlackStrandAgent 기준)

## 폴더 구조
```
agent/
├── agentcore/                  # AgentCore 배포 설정 (agentcore-cli 관리, 임의 변경 금지)
│   └── agentcore.json
└── app/
    └── SlackStrandAgent/
        ├── main.py             # 진입점: BedrockAgentCoreApp + Strands Agent
        ├── slack_utils.py      # Slack 전송 유틸 (md_to_slack, split_message, post_to_slack)
        ├── model/
        │   └── load.py         # BedrockModel 초기화
        └── tools/
            ├── __init__.py
            ├── cost.py         # get_cost_summary
            ├── idle_resources.py  # list_idle_resources
            ├── cloudwatch.py   # get_cloudwatch_metrics_bulk
            ├── resource_age.py # list_resources_by_age
            └── lambda_stats.py # get_lambda_invocation_stats
```

---

## 프레임워크

| 항목 | 값 |
|------|----|
| Agent SDK | [Strands Agents](https://strandsagents.com) (`strands`) |
| 모델 클라이언트 | `strands.models.bedrock.BedrockModel` |
| AgentCore 런타임 | `bedrock_agentcore.runtime.BedrockAgentCoreApp` |
| 실행 방식 | `agent.stream_async(input_text)` — 비동기 스트리밍 |

---

## 모델 설정

| 에이전트 | 모델 ID | 변경 이력 |
|---------|---------|-----------|
| SlackStrandAgent | `global.anthropic.claude-haiku-4-5-20251001-v1:0` | 2026-05-05 Sonnet 4.5 → Haiku 4.5 (비용 73% 절감) |
| SlackFormatAgent | `global.anthropic.claude-haiku-4-5-20251001-v1:0` | 동일 |
| SlackAws | `global.anthropic.claude-haiku-4-5-20251001-v1:0` | 동일 (LangChain `ChatBedrock` 사용) |

**비용 비교 (Bedrock On-Demand, Standard 티어)**

| 모델 | Input / 1M tokens | Output / 1M tokens |
|------|:-----------------:|:------------------:|
| Claude Sonnet 4.5 (변경 전) | $3.00 | $15.00 |
| Claude Sonnet 4 (APAC) | $3.00 | $15.00 (Sonnet 4.5와 동일, 절감 없음) |
| **Claude Haiku 4.5** (현재) | **$0.80** | **$4.00** |

> `global.` 크로스 리전 프로파일은 in-region 대비 약 18% 할증. 모든 모델이 동일한 방식이므로 상대 비율에는 영향 없음.

실측 기반 추정 (1회 실행, ~150K input / ~6K output):
Sonnet 4.5 ~$0.54/회 → Haiku 4.5 ~$0.14/회 (**74% 절감**)

---

## 진입점: main.py

```python
@app.entrypoint
async def invoke(payload, context):
    """
    payload: {"input_text": str, "session_id": str, "response_target": dict}
    """
```

**처리 흐름:**
```
payload 수신
  └→ get_or_create_agent()   # 싱글턴 — 컨테이너 재시작 전까지 재사용
       └→ agent.stream_async(input_text)
            └→ event["data"] 청크 수집
  └→ md_to_slack("".join(chunks))   # Markdown → Slack mrkdwn 변환
  └→ split_message(full_text, max_bytes=2000)   # 2000바이트 단위 분할
  └→ post_to_slack(part, response_target, log)  # 분할된 파트별 Slack 전송
```

**싱글턴 Agent 생성:**
```python
_agent = None

def get_or_create_agent():
    global _agent
    if _agent is None:
        today = datetime.now(tz=KST).strftime("%Y-%m-%d")
        system_prompt = (
            "You are an AWS infrastructure assistant that reports to a Slack channel. "
            f"Today's date is {today} (KST, Asia/Seoul). "
            "Always use this date when querying AWS Cost Explorer or any time-based APIs."
        )
        _agent = Agent(model=load_model(), system_prompt=system_prompt, tools=tools)
    return _agent
```

> **날짜 주입**: 컨테이너가 기동될 때 system_prompt에 당일 날짜를 고정. Cost Explorer 등 날짜 기반 API 쿼리의 기준일로 사용됨.

---

## 툴 목록

에이전트가 사용하는 툴은 총 **6개** (5개 커스텀 + 1개 fallback).

```python
tools = [
    get_cost_summary,
    list_idle_resources,
    get_cloudwatch_metrics_bulk,
    list_resources_by_age,
    get_lambda_invocation_stats,
    use_aws,   # strands_tools — 전용 툴이 커버하지 못하는 엣지 케이스 fallback
]
```

---

### Tool 1: `get_cost_summary` — cost.py

```python
@tool
def get_cost_summary(months: int = 1, region_filter: str = "") -> dict:
```

| 파라미터 | 기본값 | 설명 |
|---------|-------|------|
| `months` | `1` | 조회 개월 수 (1 = 이번 달만, 2 = 지난달 포함) |
| `region_filter` | `""` | 특정 리전만 필터 (예: `"ap-northeast-2"`), 빈 문자열이면 전체 |

**반환값:**
```json
{
  "period": "2026-05-01 ~ 2026-05-05",
  "region_filter": "ap-northeast-2",
  "total_usd": 303.31,
  "by_service": [
    {"service": "Amazon Virtual Private Cloud", "cost_usd": 58.51},
    ...
  ]
}
```
> `by_service`는 비용 내림차순 상위 **15개** 서비스.  
> Cost Explorer는 글로벌 API — `us-east-1` 엔드포인트 고정 사용.

---

### Tool 2: `list_idle_resources` — idle_resources.py

```python
@tool
def list_idle_resources(region: str = "ap-northeast-2") -> dict:
```

미연결 EBS 볼륨, 미할당 EIP, ALB/NLB/Classic ELB, NAT Gateway를 **단일 호출**로 반환.  
각 리소스에 `cz-owner` / `Creator` 태그 기반 **`owner` 필드를 포함**하여 LLM이 별도 태그 조회를 하지 않도록 함.

**태그 조회 방식:**

| 리소스 | 방법 |
|--------|------|
| EBS, EIP, NAT GW | `describe_*` 응답에 Tags 이미 포함 — 추가 API 호출 없음 |
| ALB/NLB | `elbv2.describe_tags(ResourceArns=[...])` — 20개씩 배치 호출 |
| Classic ELB | `elb.describe_tags(LoadBalancerNames=[...])` — 20개씩 배치 호출 |

**반환값 구조:**
```json
{
  "idle_ebs": [{"id": "vol-xxx", "name": "...", "owner": "kgkang", "size_gb": 20, "type": "gp3", "created": "..."}],
  "unassociated_eip": [{"allocation_id": "eipalloc-xxx", "public_ip": "3.34.x.x", "owner": "확인 불가"}],
  "load_balancers": {
    "alb_nlb": [{"name": "...", "type": "application|network", "arn": "...", "state": "active", "created": "...", "owner": "kgkang"}],
    "classic": [{"name": "...", "dns": "...", "created": "...", "owner": "확인 불가"}]
  },
  "nat_gateways": [{"id": "nat-xxx", "subnet_id": "...", "vpc_id": "...", "state": "available", "created": "...", "owner": "kgkang"}]
}
```
> ELB 트래픽 판단(저사용 여부)은 `get_cloudwatch_metrics_bulk`로 별도 조회.

---

### Tool 3: `get_cloudwatch_metrics_bulk` — cloudwatch.py

```python
@tool
def get_cloudwatch_metrics_bulk(
    resource_ids: list[str],
    namespace: str,
    metric_name: str,
    stat: str = "Average",
    period_days: int = 7,
    region: str = "ap-northeast-2",
) -> dict:
```

`GetMetricData` API로 최대 **500개** 리소스를 단일 호출로 일괄 조회.

| 파라미터 | 설명 |
|---------|------|
| `resource_ids` | 조회할 리소스 ID 목록 |
| `namespace` | `"AWS/EC2"`, `"AWS/RDS"`, `"AWS/ApplicationELB"`, `"AWS/NatGateway"` 등 |
| `metric_name` | `"CPUUtilization"`, `"RequestCount"`, `"BytesProcessed"` 등 |
| `stat` | `"Average"` \| `"Sum"` \| `"Maximum"` \| `"Minimum"` |
| `period_days` | 조회 기간 (기본 7일, 1일 단위 집계) |

**namespace별 자동 dimension 매핑:**
| namespace | dimension key |
|-----------|--------------|
| `AWS/EC2` | `InstanceId` |
| `AWS/RDS` | `DBInstanceIdentifier` |
| `AWS/ApplicationELB` / `AWS/NetworkELB` | `LoadBalancer` |
| `AWS/NatGateway` | `NatGatewayId` |
| `AWS/Lambda` | `FunctionName` |

**반환값:**
```json
{
  "i-0abc123": {"max": 1.23, "avg": 0.16, "sum": 1.12, "has_data": true, "data_points": 7},
  "i-0def456": {"max": null, "avg": null, "sum": null, "has_data": false, "data_points": 0}
}
```

---

### Tool 4: `list_resources_by_age` — resource_age.py

```python
@tool
def list_resources_by_age(
    min_age_days: int = 60,
    region: str = "ap-northeast-2",
) -> dict:
```

EC2, RDS, Lambda 함수, CloudFormation 스택을 **생성일 기준**으로 일괄 조회.  
각 리소스에 `cz-owner` / `Creator` 태그 기반 **`owner` 필드를 포함**.

| 리소스 | age 필터 적용 | 생성일 기준 필드 | owner 조회 방법 |
|--------|:------------:|----------------|----------------|
| EC2 (running) | ✅ | `LaunchTime` | `describe_instances` 응답의 `Tags` |
| RDS | ✅ | `InstanceCreateTime` | `describe_db_instances` 응답의 `TagList` |
| Lambda | ❌ (전체 반환) | `LastModified` | `resourcegroupstaggingapi.get_resources` (전체 일괄) |
| CloudFormation (활성 스택) | ✅ | `CreationTime` | — (owner 미포함) |

> Lambda는 `list_functions` 응답에 Tags 미포함. Resource Groups Tagging API로 리전 내 Lambda 함수 태그를 한 번에 조회 후 딕셔너리로 매핑.  
> Lambda age 필터 없이 전체 목록 반환 → 호출 통계는 `get_lambda_invocation_stats`로 별도 확인.

**CloudFormation 활성 상태 기준:**
`CREATE_COMPLETE`, `UPDATE_COMPLETE`, `UPDATE_ROLLBACK_COMPLETE`

---

### Tool 5: `get_lambda_invocation_stats` — lambda_stats.py

```python
@tool
def get_lambda_invocation_stats(
    function_names: list[str],
    days: int = 30,
    region: str = "ap-northeast-2",
) -> dict:
```

`GetMetricData`로 Lambda 함수들의 **호출 횟수 + 에러 횟수**를 단일 호출로 일괄 조회.

**반환값:**
```json
{
  "slack-aws-agent-dev-ScheduledReporterFunction": {
    "invocations": 162,
    "errors": 0,
    "error_rate_pct": 0.0
  },
  "athenafederatedcatalog_dynamo_ds": {
    "invocations": 420,
    "errors": 300,
    "error_rate_pct": 71.4
  }
}
```

---

### Tool 6: `use_aws` — fallback

`strands_tools`에서 임포트. 커스텀 툴이 커버하지 못하는 엣지 케이스(예: CloudTrail 조회 등) 처리용.

> **주의**: ELB 태그 조회는 `list_idle_resources`에서 이미 반환하므로 `use_aws`로 중복 조회하지 않아야 함.  
> `use_aws` 서비스명 참고: ALB/NLB → `"elbv2"`, Classic ELB → `"elb"`

---

## Slack 유틸리티: slack_utils.py

### `md_to_slack(text: str) -> str`

GitHub Markdown → Slack mrkdwn 변환.

| 변환 규칙 | 변환 전 | 변환 후 |
|---------|--------|--------|
| Bold | `**텍스트**` | `*텍스트*` |
| 제목 | `## 제목` | `*제목*` |
| 구분선 | `---` | (빈 줄) |
| 마크다운 테이블 | `\| col \| col \|` | ` ``` ` 블록 + 열 너비 정규화 |

> 테이블은 `_wrap_table_blocks()`로 열 너비를 동아시아 문자(한글 등) 기준으로 정규화 후 코드 블록으로 감쌈.

### `split_message(text: str, max_bytes: int = 3000) -> list[str]`

Slack API 메시지 크기 제한 대응. ` ``` ` 코드 블록 내부는 분할하지 않음.

### `post_to_slack(text, response_target, log, *, token: str) -> None`

```python
@requires_api_key(provider_name="slack-bot-token", into="token")
async def post_to_slack(...):
```

AgentCore Identity Vault에서 `slack-bot-token`을 자동 주입받아 `chat.postMessage` 호출.

| `response_target.type` | 동작 |
|------------------------|------|
| `"thread"` | `thread_ts` 지정 — 채널 thread에 달림 |
| `"dm"` | channel(D...) 에 직접 포스팅 |
| `"channel"` | channel(C...) 에 직접 포스팅 |

---

## 세션 메모리 (agentcore.json)

```json
{
  "memories": [{
    "name": "SlackAwsMemory",
    "strategies": [
      {"type": "SUMMARIZATION"},
      {"type": "USER_PREFERENCE"}
    ],
    "expiry": {"duration": 480}
  }]
}
```

| 항목 | 값 |
|------|----|
| 최대 세션 | 8시간 (HEADLESS 모드) |
| SUMMARIZATION | 긴 대화 요약으로 컨텍스트 유지 |
| USER_PREFERENCE | 자주 묻는 리소스/비용 패턴 기억 |

---

## IAM 권한 (AgentCore 실행 역할)

| AWS API | 용도 |
|---------|------|
| `ce:GetCostAndUsage` | Cost Explorer 비용 조회 |
| `cloudwatch:GetMetricData` | EC2/RDS/ELB/NAT GW/Lambda 메트릭 |
| `ec2:DescribeInstances`, `ec2:DescribeVolumes`, `ec2:DescribeAddresses`, `ec2:DescribeNatGateways` | EC2/EBS/EIP/NAT GW 목록 및 태그 |
| `rds:DescribeDBInstances` | RDS 목록 및 태그 |
| `elasticloadbalancing:DescribeLoadBalancers`, `elasticloadbalancing:DescribeTags` | ALB/NLB/Classic ELB 목록 및 태그 |
| `lambda:ListFunctions` | Lambda 함수 목록 |
| `tag:GetResources` | Resource Groups Tagging API — Lambda 함수 태그 일괄 조회 |
| `cloudformation:ListStacks` | CloudFormation 스택 목록 |
| `cloudtrail:LookupEvents` | 생성자 조회 (태그 없는 경우, `use_aws` fallback) |
| `bedrock-agentcore-identity:GetCredential` | Identity Vault에서 slack-bot-token 조회 |

---

## 자격증명 관리 (AgentCore Identity Vault)

Slack Bot Token은 SSM이 아닌 **AgentCore Identity Vault**에 저장.

| 항목 | 값 |
|------|-----|
| Provider 이름 | `slack-bot-token` |
| Provider 타입 | `ApiKeyCredentialProvider` |
| 코드 접근 | `@requires_api_key(provider_name="slack-bot-token", into="token")` |
| 등록 명령 | `agentcore add credential --name slack-bot-token --api-key <token>` |

---

## 에러 처리

| 상황 | 처리 |
|------|------|
| AWS API 실패 | 에러 메시지를 툴 반환값으로 전달 → Agent가 사용자에게 설명 |
| Slack API 실패 | CloudWatch Logs에 `error` 레벨 기록 |
| AgentCore ConcurrencyException | Strands SDK가 자동으로 raise — 기존 세션 처리 완료 후 재시도 필요 |
| 빈 응답 | `full_text` 가 비어 있으면 Slack 전송 스킵 |
