# Lambda Handler 명세

## 파일 구조
```
lambda/
├── template.yaml          # SAM: API Gateway + Lambda 정의
├── samconfig.toml         # 배포 환경 설정 (dev/prod)
└── src/
    ├── requirements.txt
    ├── slack_handler/
    │   └── handler.py     # Lambda A: Slack 수신 및 이벤트 분기
    └── agentcore_invoker/
        └── handler.py     # Lambda B: AgentCore 호출
```

## Lambda A — slack_handler/handler.py

### `lambda_handler(event, context)` — 진입점
```python
def lambda_handler(event: dict, context) -> dict:
    """
    Returns: {"statusCode": 200, "body": ""}  # 항상 즉시 반환
    """
```

**처리 순서:**
1. Slack 서명 검증 (`verify_slack_signature`) → 실패 시 403 반환
2. body 파싱 → `event_type` 추출
3. `url_verification` → 챌린지 값 즉시 반환
4. `bot_id` 존재 시 → 200 즉시 반환 (봇 루프 방지)
5. `app_mention` / `message.im` → `invoke_agentcore_async` 호출 후 200 반환

---

### `verify_slack_signature(headers, body_raw) -> bool`
```python
def verify_slack_signature(headers: dict, body_raw: bytes) -> bool:
    """
    Slack의 X-Slack-Signature 헤더로 요청 진위 검증.
    SLACK_SIGNING_SECRET 환경변수 사용.
    타임스탬프 5분 초과 시 False 반환 (replay attack 방지).
    """
```

---

### `parse_event(body: dict) -> tuple[str, dict]`
```python
def parse_event(body: dict) -> tuple[str, dict]:
    """
    Returns: (event_type, slack_event)
    event_type: "url_verification" | "app_mention" | "message.im" | "unknown"
    """
```

**분기 로직:**
```
body["type"] == "url_verification"  → ("url_verification", body)
body["event"]["type"] == "app_mention"  → ("app_mention", body["event"])
body["event"]["type"] == "message" AND body["event"]["channel_type"] == "im"
    → ("message.im", body["event"])
그 외 → ("unknown", {})
```

---

### `build_response_target(event_type: str, slack_event: dict) -> dict`
```python
def build_response_target(event_type: str, slack_event: dict) -> dict:
    """
    Returns:
      app_mention → {"type": "thread", "channel": "C...", "thread_ts": "..."}
      message.im  → {"type": "dm", "channel": "D..."}
    """
```

**세션 ID 결정:**
- `app_mention`: `slack_event["thread_ts"]` 있으면 사용, 없으면 `slack_event["ts"]`
- `message.im`: `slack_event["user"]` (Slack user_id)

---

### `strip_mention(text: str) -> str`
```python
def strip_mention(text: str) -> str:
    """
    "<@U12345> 비용 알려줘" → "비용 알려줘"
    정규식: re.sub(r"<@[A-Z0-9]+>\s*", "", text).strip()
    """
```

---

### `invoke_agentcore_async(input_text, session_id, response_target) -> None`
```python
def invoke_agentcore_async(
    input_text: str,
    session_id: str,
    response_target: dict,
) -> None:
    """
    Lambda B(AgentCoreInvokerFunction)를 InvocationType="Event"로 비동기 호출.
    AGENTCORE_INVOKER_FUNCTION_NAME 환경변수에서 함수명 읽음.
    호출 후 즉시 반환 — AgentCore 완료를 기다리지 않음.
    """
```

---

## Lambda B — agentcore_invoker/handler.py

### `lambda_handler(event, context)` — 진입점
```python
def lambda_handler(event: dict, context) -> None:
    """
    Lambda A가 전달한 payload를 받아 AgentCore를 동기 호출.
    event: {"input_text": str, "session_id": str, "response_target": dict}
    AGENTCORE_ENDPOINT 환경변수에서 AgentCore Runtime ARN 읽음.
    Timeout: 300초 (AgentCore 처리 시간 감안).
    """
```

**AgentCore 호출 payload:**
```json
{
    "input_text": "사용자 메시지",
    "session_id": "thread_ts | user_id",
    "response_target": {"type": "thread|dm", "channel": "C...", "thread_ts": "..."}
}
```

---

## Lambda C — scheduled_reporter/handler.py

### `lambda_handler(event, context)` — 진입점
```python
def lambda_handler(_event, _context) -> None:
    """
    EventBridge Schedule에서 호출. AgentCore에 정기 보고를 요청하고 즉시 반환.
    AgentCore는 연결 종료 후에도 독립적으로 실행을 계속하여 Slack에 직접 보고.
    """
```

**처리 순서:**
1. `session_id = f"scheduled-report-{uuid.uuid4().hex}"` — 실행마다 고유 세션 생성
2. `response_target = {"type": "channel", "channel": SLACK_REPORT_CHANNEL}`
3. `invoke_agent_runtime()` 호출 (read_timeout=15s)
4. `ReadTimeoutError` → 정상 경로 (fire-and-forget 완료 신호)
5. 그 외 예외 → 로그 기록 후 raise

**fire-and-forget 구현 방식:**

```
Lambda                    AgentCore
   │──── HTTP POST ─────────→ │  요청 수신, 세션 시작
   │                          │  (독립적으로 처리 계속)
   │  [read_timeout=15s]      │
   │  ReadTimeoutError 발생   │
   │  Lambda 즉시 반환        │
                              │  도구 실행 / 보고서 생성 중...
                              │──── Slack POST ─→ 완료
```

**boto3 클라이언트 설정:**
```python
Config(
    connect_timeout=10,
    read_timeout=15,          # 요청 전송 확인 후 즉시 반환
    retries={"max_attempts": 1},  # 자동 재시도 비활성화
)
```

> **ReadTimeoutError를 정상 경로로 처리하는 근거**: AgentCore Runtime은 HTTP 연결이 종료되어도 컨테이너 내 에이전트 실행을 계속한다. ConcurrencyException 로그 분석(2026-05-04)에서 연결 끊김 후에도 에이전트가 완료되고 Slack 보고가 이루어짐을 확인.

**보고 항목 (INPUT_TEXT):**
- 이번 달 AWS 서비스별 비용 현황 (상위 10개)
- 유휴 리소스 목록 및 예상 절감액 (EC2/RDS/ELB/ElastiCache/NAT GW/EBS/EIP/Lambda)
- 생성 60일 초과 자원 목록 (유형·이름·생성일·경과일수·생성자)

---

## template.yaml 핵심 구조

```yaml
Globals:
  Function:
    Runtime: python3.12
    Timeout: 10
    Environment:
      Variables:
        SLACK_BOT_TOKEN:       # SSM resolve
        SLACK_SIGNING_SECRET:  # SSM resolve
        SLACK_REPORT_CHANNEL:  # SSM resolve

Resources:
  SlackReceiverFunction:       # Lambda A
    Timeout: 10 (Globals 상속)
    Environment:
      AGENTCORE_INVOKER_FUNCTION_NAME: !Ref AgentCoreInvokerFunction
    Policies:
      - lambda:InvokeFunction → AgentCoreInvokerFunction
    Events: POST /slack/events

  AgentCoreInvokerFunction:    # Lambda B
    Timeout: 300
    Environment:
      AGENTCORE_ENDPOINT:      # SSM resolve
    Policies:
      - bedrock-agentcore:InvokeAgentRuntime

  ScheduledReporterFunction:   # Lambda C
    Timeout: 30                # read_timeout(15) + connect_timeout(10) + 여유
    Environment:
      AGENTCORE_ENDPOINT:      # SSM resolve
    Policies:
      - bedrock-agentcore:InvokeAgentRuntime
    Events:
      DailyReport:
        Schedule: "cron(0 * * * ? *)"  # 매시 정각
        RetryPolicy:
          MaximumRetryAttempts: 0      # Lambda 실패 시 EventBridge 재시도 없음
```

## samconfig.toml 구조

```toml
[dev.deploy.parameters]
stack_name = "slack-aws-agent-dev"
region = "ap-northeast-2"
profile = "dev"
s3_bucket = "..."
capabilities = "CAPABILITY_IAM"

[prod.deploy.parameters]
stack_name = "slack-aws-agent-prod"
region = "ap-northeast-2"
profile = "prod"
```

## 에러 처리 원칙
- Slack 서명 검증 실패: 403 반환
- AgentCore 호출 실패: 로그만 남기고 200 반환 (Slack 재시도 방지)
- 알 수 없는 이벤트 타입: 200 반환 (무시)
- 예외 발생 시 CloudWatch Logs에 full traceback 기록
