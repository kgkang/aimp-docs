# AgentCore Agent 명세

## 위치
```
agent/
├── agentcore/           # AgentCore 배포 설정 (agentcore-cli 관리, 임의 변경 금지)
├── app/
│   └── SlackAws/
│       ├── agent.py     # LangGraph Agent 진입점
│       ├── tools.py     # AWS + Slack 툴 정의
│       └── requirements.txt
└── AGENTS.md
```

## Agent 진입점: agent.py

```python
async def invoke(input_text: str, session_id: str, response_target: dict) -> str:
    """
    AgentCore Runtime이 호출하는 메인 함수.
    LangGraph ReAct 루프 실행 후 Slack에 응답 전송.
    """
```

**처리 흐름:**
1. `session_id`로 메모리 로드 (AgentCore Memory API)
2. LangGraph ReAct Agent 실행 (도구 호출 반복)
3. 최종 답변 생성
4. `response_target` 기반으로 Slack에 메시지 전송
5. 메모리 저장

## 툴 명세: tools.py

### Tool 1: `get_monthly_cost`
```python
@tool
def get_monthly_cost(year_month: str = None) -> str:
    """
    AWS Cost Explorer로 월별 서비스별 비용 조회.
    year_month: "YYYY-MM" 형식, None이면 이번 달
    Returns: 서비스명별 USD 비용 JSON 문자열
    """
```

**AWS API:**
```python
ce.get_cost_and_usage(
    TimePeriod={"Start": "YYYY-MM-01", "End": "YYYY-MM-01 (다음달)"},
    Granularity="MONTHLY",
    Metrics=["UnblendedCost"],
    GroupBy=[{"Type": "DIMENSION", "Key": "SERVICE"}]
)
```

---

### Tool 2: `get_idle_resources`
```python
@tool
def get_idle_resources(threshold_cpu_percent: float = 5.0) -> str:
    """
    CPU 평균 사용률 threshold 이하인 EC2/RDS 목록 조회.
    최근 7일 평균 기준.
    Returns: 유휴 리소스 목록 JSON (instance_id, type, avg_cpu, days)
    """
```

**AWS API:**
- EC2: `ec2.describe_instances(Filters=[{"Name": "instance-state-name", "Values": ["running"]}])`
- CloudWatch: `cloudwatch.get_metric_statistics` (CPUUtilization, 7일, Average)

---

### Tool 3: `list_running_instances`
```python
@tool
def list_running_instances(resource_type: str = "all") -> str:
    """
    실행 중인 EC2/RDS 인스턴스 목록 조회.
    resource_type: "ec2" | "rds" | "all"
    Returns: 인스턴스 목록 JSON (id, type, region, state, tags)
    """
```

---

### Tool 4: `send_slack_message`
```python
@tool
def send_slack_message(channel: str, text: str, thread_ts: str = None) -> str:
    """
    Slack API로 메시지 전송. Agent의 최종 응답 전송에 사용.
    channel: Slack channel ID (C... or D...)
    thread_ts: 지정 시 thread에 달림
    SLACK_BOT_TOKEN 환경변수 사용.
    Returns: "ok" | 에러 메시지
    """
```

**Slack API:** `POST https://slack.com/api/chat.postMessage`

## LangGraph 구조

```
START → agent_node ⟷ tool_node → END
```

- **모델**: Claude Sonnet (Bedrock)
- **루프**: ReAct (Reasoning + Acting) 최대 10회
- **System Prompt**: AWS 비용/리소스 분석 전문가 페르소나, 한국어 응답

## System Prompt 기본 구조

```
당신은 AWS 비용 및 리소스 최적화 전문가입니다.
사용자의 AWS 환경을 분석하고 한국어로 답변합니다.

답변 형식:
- Slack 메시지에 적합한 간결한 형식 사용
- 비용은 USD로 표시하되 KRW 환산값 병기 (환율 약 1,350원)
- 유휴 리소스 발견 시 구체적인 비용 절감 예상액 제시

제약:
- 읽기 전용 작업만 수행 (리소스 변경/삭제 불가)
- 답변 완료 후 반드시 send_slack_message 툴로 응답 전송
```

## 세션 메모리 설정 (agentcore/agentcore.json)

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

- 세션 최대 8시간 (HEADLESS 모드)
- SUMMARIZATION: 긴 대화를 요약하여 컨텍스트 유지
- USER_PREFERENCE: 사용자가 자주 묻는 리소스/비용 패턴 기억

## 환경변수 (AgentCore Runtime)
- `SLACK_BOT_TOKEN`: SSM에서 로드
- `AWS_DEFAULT_REGION`: ap-northeast-2

## 에러 처리
- AWS API 실패: 에러 메시지를 tool 반환값으로 전달 → Agent가 사용자에게 설명
- Slack API 실패: 재시도 1회, 실패 시 CloudWatch Logs에 기록
- 최대 루프 초과: "처리 중 오류가 발생했습니다" 메시지 전송
