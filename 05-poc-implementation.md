### 핵심 구현 내용

이번 PoC에서 실제 코드로 구현된 핵심 기능들을 **동작 원리**와 **사용 기술** 중심으로 기술합니다.

---

#### 1.1 소스 구성

```
project/
├── lambda/
│   ├── template.yaml                        # SAM: API Gateway + Lambda A/B/C 정의
│   ├── samconfig.toml                       # 배포 환경 설정 (dev/prod)
│   └── src/
│       ├── requirements.txt                 # boto3>=1.38.0
│       ├── slack_handler/
│       │   └── handler.py                   # Lambda A: Slack 수신 및 이벤트 분기
│       ├── agentcore_invoker/
│       │   └── handler.py                   # Lambda B: AgentCore 동기 호출
│       └── scheduled_reporter/
│           └── handler.py                   # Lambda C: 정기 보고 (fire-and-forget)
│
└── agent/
    ├── agentcore/
    │   └── agentcore.json                   # AgentCore 배포 설정 (agentcore-cli 관리)
    └── app/
        └── SlackStrandAgent/
            ├── main.py                      # 진입점: BedrockAgentCoreApp + Strands Agent
            ├── slack_utils.py               # md_to_slack, split_message, post_to_slack
            ├── model/
            │   └── load.py                  # BedrockModel 초기화 (Haiku 4.5)
            └── tools/
                ├── __init__.py
                ├── cost.py                  # get_cost_summary
                ├── idle_resources.py        # list_idle_resources
                ├── cloudwatch.py            # get_cloudwatch_metrics_bulk
                ├── resource_age.py          # list_resources_by_age
                └── lambda_stats.py          # get_lambda_invocation_stats
```

---

#### 1.2 에이전트 워크플로우 (Agent Workflow)

**구현 기능:** Slack 이벤트 수신 → 비동기 분기 → AgentCore Runtime 에이전트 실행 → Slack 응답의 3단계 파이프라인

**동작 원리:**

Slack의 3초 응답 제한을 만족시키면서 장시간(수 분) 실행되는 에이전트를 운용하기 위해, Lambda를 A/B/C 세 개로 분리하고 비동기 호출 체인으로 구성했습니다.

```
[Slack DM/채널 멘션]
    └→ Lambda A (SlackReceiver)
           ├ 서명 검증 + 이벤트 분기
           ├ Lambda B를 InvocationType=Event로 비동기 호출
           └ 즉시 200 반환 (Slack 3초 제한 대응)
                   ↓
           Lambda B (AgentCoreInvoker)
                   └→ AgentCore Runtime (SlackStrandAgent)
                          ├ Strands Agent 실행 (6개 툴 활용)
                          └ Slack API 직접 호출 (Identity Vault 토큰 사용)

[EventBridge 정기 보고]
    └→ Lambda C (ScheduledReporter)
           ├ fire-and-forget으로 AgentCore 호출
           └ ReadTimeoutError 정상 처리 → 즉시 반환
```

AgentCore Runtime 내 에이전트는 `BedrockAgentCoreApp`의 `@app.entrypoint`로 진입하며, Strands `Agent.stream_async()`로 비동기 스트리밍 실행합니다.

**주요 기술:** AWS Lambda (Python 3.12), AWS SAM, Strands Agents SDK, AWS Bedrock AgentCore Runtime, `InvocationType=Event` (Lambda 비동기 호출)

---

#### 1.3 도구(Tool) 및 함수 연동

**구현 기능:** AWS 비용·리소스·운영 현황을 조회하는 커스텀 툴 5종 + fallback 1종

**동작 원리:**

에이전트가 Strands `@tool` 데코레이터로 정의된 함수를 ReAct 루프로 호출합니다. 각 툴은 boto3로 AWS API를 직접 호출하고 구조화된 dict를 반환하며, LLM이 결과를 해석해 다음 툴 호출 또는 최종 답변을 생성합니다.

| 툴 | 파일 | 용도 |
|---|---|---|
| `get_cost_summary` | `tools/cost.py` | Cost Explorer 서비스별 비용 (상위 15개) |
| `list_idle_resources` | `tools/idle_resources.py` | 미연결 EBS·EIP·ELB·NAT GW 목록 + owner 태그 |
| `get_cloudwatch_metrics_bulk` | `tools/cloudwatch.py` | 최대 500개 리소스 메트릭 단일 호출 조회 |
| `list_resources_by_age` | `tools/resource_age.py` | 생성일 기준 60일+ EC2·RDS·Lambda·CFN 스택 |
| `get_lambda_invocation_stats` | `tools/lambda_stats.py` | Lambda 호출 횟수·에러율 일괄 조회 |
| `use_aws` | strands_tools (라이브러리) | 커스텀 툴 미커버 엣지 케이스 fallback |

**소유자 태그 내재화 설계:** 초기 구현에서 LLM이 `use_aws` fallback으로 소유자 태그를 별도 조회하는 문제가 발견되어, 모든 커스텀 툴의 반환값에 `owner` 필드(`cz-owner` / `Creator` 태그 우선순위)를 직접 포함시켰습니다. 이를 통해 실행당 `use_aws` 호출 10회 → 0회로 제거했습니다.

**주요 기술:** Strands Agents `@tool`, boto3, AWS Cost Explorer, CloudWatch `GetMetricData` (배치), EC2/RDS/ELB/EIP/NAT GW API, Resource Groups Tagging API

---

#### 1.4 Slack 메시지 처리

**구현 기능:** Markdown → Slack mrkdwn 변환, 바이트 단위 메시지 분할, AgentCore Identity Vault 토큰 주입

**동작 원리:**

LLM 응답(GitHub Markdown)을 Slack이 렌더링하는 mrkdwn 포맷으로 변환한 뒤, 2000바이트 단위로 분할하여 순차 전송합니다. Slack Bot Token은 코드와 분리하여 AgentCore Identity Vault에 저장하고, `@requires_api_key` 데코레이터로 함수 호출 시점에 자동 주입받습니다.

```python
@requires_api_key(provider_name="slack-bot-token", into="token")
async def post_to_slack(text, response_target, log, *, token: str) -> None:
    # token이 Identity Vault에서 자동 주입됨
    ...
```

테이블은 동아시아 문자(한글) 너비를 고려한 열 정규화 후 코드 블록으로 감싸 모노스페이스로 렌더링합니다.

**주요 기술:** httpx (비동기 HTTP), AgentCore `@requires_api_key`, `unicodedata.east_asian_width`, Slack `chat.postMessage` API

---

#### 1.5 데이터 및 세션 메모리

**구현 기능:** 이벤트 타입별 대화 컨텍스트 격리 및 세션 메모리 관리

**동작 원리:**

AgentCore Runtime이 `runtimeSessionId` 기준으로 대화 히스토리를 자동 관리합니다. Strands SDK가 turn-by-turn 컨텍스트를 유지하므로 별도 Memory API를 사용하지 않습니다.

| 이벤트 | session_id 생성 방식 | 목적 |
|---|---|---|
| 채널 멘션 | `sha256("thread:{thread_ts}")` | thread 단위 대화 연속성 |
| DM | `sha256("dm:{user_id}")` | 사용자 단위 대화 연속성 |
| 정기 보고 | `f"scheduled-report-{uuid4().hex}"` | 실행마다 새 세션 (충돌 방지) |

세션은 idle 15분 초과 시 자동 종료, 최대 8시간(HEADLESS 모드).

**주요 기술:** AgentCore Runtime 세션 메모리, SHA-256 session_id 해싱, UUID4

---

### 주요 문제 해결 및 기술 리서치

구현 과정에서 마주친 기술적 문제와 이를 해결하기 위해 찾아본 자료 및 적용한 방법을 기록합니다.

| 이슈 구분 | 문제 상황 및 원인 | 리서치 및 해결 과정 |
|---|---|---|
| **boto3 서비스명** | Lambda B에서 `bedrock-agentcore-runtime` 서비스명으로 클라이언트 생성 시 `UnknownServiceError` 발생 | AWS boto3 서비스 목록 확인 → 올바른 서비스명은 `bedrock-agentcore`, 메서드는 `invoke_agent_runtime`. `bedrock-agentcore-runtime`은 유효하지 않은 이름. |
| **Slack 3초 응답 제한** | Lambda에서 `threading.Thread`로 AgentCore를 비동기 호출하면 Lambda freeze 시 스레드 실행 보장 없음. Slack이 3초 내 응답 없으면 재시도 | Lambda를 A(수신·즉시반환)와 B(AgentCore 호출)로 분리하고 `InvocationType=Event`로 Lambda A→B 비동기 호출. AWS가 Lambda B의 재시도를 보장하며 Lambda A는 즉시 200 반환. |
| **EventBridge ConcurrencyException** | EventBridge Scheduler가 매시 정각 Lambda C를 트리거할 때, 이전 실행이 완료 전에 재시도가 동일 세션으로 AgentCore에 요청 → ConcurrencyException(500) | EventBridge `RetryPolicy: MaximumRetryAttempts: 0` + Lambda `EventInvokeConfig MaximumRetryAttempts: 0` 이중 설정으로 재시도 차단. 두 설정의 레이어가 다름을 확인(EventBridge 트리거 재시도 vs Lambda 비동기 재시도). |
| **boto3 legacy 재시도 모드** | `Config(retries={"max_attempts": 1})`에서 legacy 모드(기본값)로는 `max_attempts`가 재시도 횟수를 의미 → 실제 2회 요청 발생 → 동일 세션 ConcurrencyException | boto3 문서 확인 → `mode: "standard"`에서는 `max_attempts=1`이 "총 1회 시도(재시도 0회)"로 올바르게 동작. `retries={"mode": "standard", "max_attempts": 1}`으로 변경. |
| **fire-and-forget 구현** | Lambda C에서 `invoke_agent_runtime`이 HTTP blocking 호출이라 AgentCore 응답(~5분)까지 대기 → Lambda timeout. boto3 기본 read_timeout(60s) 만료 시 자동 재시도 → ConcurrencyException | AgentCore는 HTTP 연결 종료 후에도 컨테이너 내 독립 실행을 계속함을 CloudWatch 로그로 확인. `read_timeout=10s`로 의도적 short timeout 설정 + `ReadTimeoutError`를 정상 경로로 처리(fire-and-forget 완료 신호). `except Exception`에서 `raise` 제거로 Lambda 비동기 재시도 방지. |
| **고정 session_id 중복** | 정기 보고에 고정 session_id `"scheduled-report"` 사용 시 EventBridge/Lambda retry가 동일 세션에 충돌 | session_id를 `f"scheduled-report-{uuid4().hex}"` 형식으로 변경. 매 실행마다 고유 세션이므로 retry도 새 세션으로 처리됨. |
| **모델 비용 과다** | Claude Sonnet 4.5로 4일 실측 Bedrock 비용 $74.90. 툴 호출 중심 작업에 고가 모델 사용 과다 | Bedrock On-Demand 단가 비교: Sonnet 4.5 ($3.00/$15.00 per MTok) vs Haiku 4.5 ($0.80/$4.00 per MTok). 툴 호출 22회/실행 패턴은 Haiku로 충분. 전 에이전트를 `global.anthropic.claude-haiku-4-5-20251001-v1:0`으로 변경 → 실행당 $0.54 → $0.14 (~74% 절감). |
| **Haiku 4.5 크로스 리전** | ap-northeast-2(서울)에서 Haiku 4.5 In-Region 및 APAC Geo 프로파일 미가용 | APAC Geo는 AU(시드니·멜버른)만 존재. 서울에서는 `global.` 크로스 리전 프로파일이 유일한 선택지. in-region 대비 ~18% 할증이 있으나 세 모델 동일 조건이므로 상대 비율에 영향 없음. 실측: 22회 LLM 호출 × 200~500ms 추가 = 총 4~11초, 전체 실행 ~5분 대비 1~4%로 무시 가능. |
| **`use_aws` 과다 호출** | LLM이 보고서에 소유자 정보 포함을 위해 매 실행마다 `use_aws` fallback으로 태그 조회 10회 발생. ELB 서비스명 시행착오(`elasticloadbalancing` → 실패 → `elbv2`)로 LLM 호출 추가 낭비 | CloudWatch bedrock-logging으로 도구 호출 패턴 분석 → 근본 원인: 커스텀 툴 반환값에 `owner` 필드 미포함. 개선: `list_idle_resources`에 `_batch_elbv2_tags()`, `_batch_elb_tags()` 추가, `list_resources_by_age`에 `_fetch_lambda_owners()` 추가(Resource Groups Tagging API). 결과: `use_aws` 10회 → 0회, input token 65% 감소, 실행 비용 $0.169 → $0.078 (−54%). |
| **AgentCore Identity Vault** | AgentCore Agent에서 Slack Bot Token을 환경변수나 SSM으로 관리하면 토큰 노출·재배포 필요 문제 | `@requires_api_key(provider_name="slack-bot-token")` 데코레이터로 Identity Vault에서 자동 주입. `agentcore add credential --name slack-bot-token --api-key <token>` 한 번만 등록. CloudTrail 자동 기록, 로테이션 시 재배포 불필요. |
| **requirements.txt 위치** | SAM 빌드 시 `No requirements.txt found` 에러. `sam build`가 requirements.txt를 인식하지 못함 | SAM PythonPipBuilder가 `CodeUri` 루트에서 requirements.txt를 탐색함을 확인. `lambda/src/requirements.txt` 위치로 이동 후 `sam build` 성공. |
| **Markdown 테이블 Slack 렌더링** | LLM이 출력한 Markdown 테이블이 Slack에서 깨져 보임 (비례 폰트 렌더링) | Slack mrkdwn에서 테이블 지원 없음 → 코드 블록(` ``` `)으로 감싸 모노스페이스로 표시하는 방식 채택. 한글 등 동아시아 문자는 `unicodedata.east_asian_width`로 표시 너비를 2로 계산하여 열 정렬. |

---

### 핵심 동작 검증

구현된 기능이 의도대로 동작하는지 보여주는 대표적인 실행 결과를 기록합니다.

---

**[검증 시나리오 1: 정기 보고 — AWS 현황 보고서 자동 생성]**

* **트리거:** EventBridge Schedule (매시 정각)

* **에이전트 동작 (실측 2026-05-06 00:22, 개선 후):**

  1. `list_idle_resources()` 호출 → EBS 미연결 볼륨, EIP, ALB/NLB, NAT GW 목록 반환 (owner 태그 포함)
  2. `get_cloudwatch_metrics_bulk()` 호출 → ALB/NAT GW 트래픽 메트릭 조회
  3. `get_cost_summary()` 호출 → 이번 달 서비스별 비용 상위 15개 반환
  4. `list_resources_by_age(min_age_days=60)` 호출 → EC2·RDS·Lambda·CFN 스택 목록 반환
  5. `get_lambda_invocation_stats()` 호출 → Lambda 호출 횟수·에러율 반환
  6. Markdown 보고서 생성 → `md_to_slack()` 변환 → `split_message()` 분할 → Slack 전송

* **실측 지표 (개선 후):**

  | 지표 | 값 |
  |---|---|
  | LLM 호출 횟수 | 4회 |
  | 총 툴 호출 | 6회 |
  | `use_aws` 호출 | 0회 |
  | Input tokens | 59,538 |
  | Output tokens | 7,540 |
  | 실행 시간 | 71.3초 |
  | 실행 비용 | $0.078 |

---

**[검증 시나리오 2: Slack 채널 멘션 — 대화형 AWS 질의]**

* **입력:** `@aws-agent 이번달 EC2 비용 알려줘`

* **에이전트 동작:**

  1. Lambda A: Slack 서명 검증 → `app_mention` 이벤트 파싱 → `strip_mention()`으로 멘션 제거
  2. Lambda A: session_id `sha256("thread:{ts}")` 생성 → Lambda B 비동기 호출 → 200 반환
  3. Lambda B: AgentCore Runtime 동기 호출 (Timeout: 300초)
  4. AgentCore: `get_cost_summary(region_filter="ap-northeast-2")` 호출 → 비용 데이터 반환
  5. Slack thread에 응답 전송 (`thread_ts` 지정)

* **결과:** Slack thread에 EC2 비용 내역이 mrkdwn 포맷으로 전송됨. Lambda A는 Slack 3초 제한 이내 200 반환 완료.

---

**[검증 시나리오 3: fire-and-forget 동작 확인]**

* **상황:** Lambda C가 `invoke_agent_runtime` 호출 후 `read_timeout=10s` 초과

* **동작:**
  ```
  Lambda C ──── HTTP POST ────→ AgentCore Runtime
  Lambda C  [read_timeout=10s 경과]
  Lambda C  ReadTimeoutError 수신 → 정상 경로 로그 기록 → 즉시 반환
                                          AgentCore (독립 실행 계속)
                                          ... 보고서 생성 (~71초) ...
                                          Slack API → SLACK_REPORT_CHANNEL 전송
  ```

* **확인 방법:** CloudWatch Logs에서 `"Scheduled report fired (fire-and-forget)"` 로그 확인 후, 약 1~2분 뒤 Slack 채널에 보고서 수신.
