# 아키텍처 설계

## 시스템 흐름

### 흐름 1: 정기 보고
```
EventBridge (cron(0/30 * * * ? *), 매시 정각·30분)
    └→ Lambda C: ScheduledReporter
           └→ AgentCore Runtime
                  └→ AWS Tools (Cost Explorer, CloudWatch, EC2/RDS/ELB/ElastiCache/NAT GW/EBS/EIP/Lambda/CloudFormation, CloudTrail)
                  └→ Slack API → SLACK_REPORT_CHANNEL
```

### 흐름 2: Slack DM
```
사용자 DM
    └→ Slack Events API
           └→ API Gateway (POST /slack/events)
                  └→ Lambda A: SlackReceiver (즉시 200 반환)
                         └→ Lambda B: AgentCoreInvoker (InvocationType=Event, 비동기)
                                └→ AgentCore Runtime
                                       └→ AWS Tools
                                       └→ Slack API → DM channel
```

### 흐름 3: 채널 멘션
```
@mention in channel
    └→ Slack Events API
           └→ API Gateway (POST /slack/events)
                  └→ Lambda A: SlackReceiver (즉시 200 반환)
                         └→ Lambda B: AgentCoreInvoker (InvocationType=Event, 비동기)
                                └→ AgentCore Runtime
                                       └→ AWS Tools
                                       └→ Slack API → thread (thread_ts 지정)
```

## 컴포넌트 경계

```
┌──────────────────────────────────────────────────┐
│  AWS Cloud (ap-northeast-2)                       │
│                                                   │
│  ┌──────────────┐    ┌───────────────────────┐   │
│  │  API Gateway │    │      EventBridge       │   │
│  │  /slack/events│    │  cron(0/30 * * * ? *) │   │
│  └──────┬───────┘    └──────────┬────────────┘   │
│         │                       │                 │
│  ┌──────▼───────┐    ┌──────────▼────────────┐   │
│  │  Lambda A    │    │       Lambda C         │   │
│  │SlackReceiver │    │  ScheduledReporter     │   │
│  │  - 서명 검증  │    │  - 비용/유휴/60일 보고  │   │
│  │  - 이벤트 분기│    │  - UUID session_id     │   │
│  │  - 즉시 200  │    │  - fire-and-forget     │   │
│  └──────┬───────┘    └──────────┬────────────┘   │
│         │ Invoke(Event)         │                 │
│  ┌──────▼───────┐               │                 │
│  │  Lambda B    │               │                 │
│  │AgentCoreInvkr│               │                 │
│  │  - AgentCore │               │                 │
│  │    호출(동기) │               │                 │
│  └──────┬───────┘               │                 │
│         │                       │                 │
│  ┌──────▼───────────────────────▼─────────────┐  │
│  │              AgentCore Runtime              │  │
│  │  (SlackStrandAgent, Strands SDK)           │  │
│  │  - AWS Tools (Cost/CW/EC2/RDS/ELB/         │  │
│  │    ElastiCache/NAT GW/EBS/EIP/Lambda/      │  │
│  │    CloudFormation, CloudTrail)             │  │
│  │  - Slack 응답 (@requires_api_key)          │  │
│  │  - 세션 메모리 (idle 15분 / 최대 8시간)      │  │
│  └──────────────────┬──────────────────────────┘  │
│                     │ api_key 자동 주입            │
│  ┌──────────────────▼──────────────────────────┐  │
│  │  AgentCore Identity Vault                   │  │
│  │  slack-bot-token (ApiKeyProvider)           │  │
│  └─────────────────────────────────────────────┘  │
│                                                   │
│  ┌───────────────────────────────────────────┐   │
│  │  SSM Parameter Store                       │   │
│  │  /prod/slack/signing-secret                │   │
│  │  /prod/agentcore/endpoint                  │   │
│  │  /prod/slack/report-channel                │   │
│  └───────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
              │ (외부)
         ┌────▼────┐
         │  Slack  │
         └─────────┘
```

## Lambda ↔ AgentCore 인터페이스

Lambda가 AgentCore를 호출할 때 전달하는 payload:

```json
{
  "input_text": "사용자 메시지 (멘션 제거 후) 또는 정기 보고 프롬프트",
  "session_id": "thread_ts | user_id | scheduled-report-{uuid32hex}",
  "response_target": {
    "type": "thread | dm | channel",
    "channel": "C... or D...",
    "thread_ts": "1234.5678"
  }
}
```

AgentCore는 응답 완료 후 response_target을 참조하여 직접 Slack API 호출.

## 세션 ID 전략

| 이벤트 | session_id | 목적 |
|--------|-----------|------|
| app_mention | thread_ts | thread 단위 대화 유지 |
| message.im | Slack user_id | 사용자 단위 대화 유지 |
| EventBridge | `scheduled-report-{uuid32hex}` (실행마다 고유) | 실행 간 히스토리 오염 방지, 동시 실행 충돌 방지 |

## IAM 권한 설계

### Lambda A (SlackReceiver) 실행 역할
- `ssm:GetParameter` — /prod/* 파라미터 읽기
- `lambda:InvokeFunction` — AgentCoreInvokerFunction 호출
- `logs:CreateLogGroup`, `logs:PutLogEvents` — CloudWatch Logs

### Lambda B (AgentCoreInvoker) 실행 역할
- `bedrock-agentcore:InvokeAgentRuntime` — AgentCore Runtime 호출
- `ssm:GetParameter` — /prod/* 파라미터 읽기
- `logs:CreateLogGroup`, `logs:PutLogEvents` — CloudWatch Logs

### Lambda C (ScheduledReporter) 실행 역할
- `bedrock-agentcore:InvokeAgentRuntime` — AgentCore Runtime 호출
- `logs:CreateLogGroup`, `logs:PutLogEvents` — CloudWatch Logs
- EventBridge가 자동 생성하는 Lambda 호출 권한 (`AWS::Lambda::Permission`)

### AgentCore 실행 역할 (agent/agentcore/에서 관리)
- `ce:GetCostAndUsage` — Cost Explorer (비용 현황)
- `cloudwatch:GetMetricStatistics` — CloudWatch 지표 (유휴 리소스 판단)
- `ec2:DescribeInstances`, `ec2:DescribeVolumes`, `ec2:DescribeAddresses`, `ec2:DescribeNatGateways` — EC2/EBS/EIP/NAT GW
- `rds:DescribeDBInstances` — RDS 목록
- `elasticloadbalancing:DescribeLoadBalancers`, `elasticloadbalancing:DescribeTargetGroups` — ELB/ALB
- `elasticache:DescribeCacheClusters` — ElastiCache
- `lambda:ListFunctions`, `lambda:GetFunctionConfiguration` — Lambda 함수
- `cloudformation:DescribeStacks` — CloudFormation 스택
- `cloudtrail:LookupEvents` — 자원 생성자 조회 (태그 없는 경우)
- `bedrock-agentcore-identity:GetCredential` — Identity Vault에서 API 키 조회

## 자격증명 관리 (AgentCore Identity)

Slack Bot Token은 SSM이 아닌 **AgentCore Identity Vault**에 저장하여 관리한다.

| 항목 | 값 |
|------|-----|
| Provider 이름 | `slack-bot-token` |
| Provider 타입 | `ApiKeyCredentialProvider` |
| 코드 접근 방식 | `@requires_api_key(provider_name="slack-bot-token")` |
| 등록 명령 | `agentcore add credential --name slack-bot-token --api-key <token>` |

**장점:**
- 토큰이 코드, 설정 파일, 환경변수 어디에도 노출되지 않음
- 모든 접근이 CloudTrail에 자동 기록
- 토큰 로테이션 시 `agentcore add credential` 재실행만으로 적용 (재배포 불필요)
