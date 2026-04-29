# 배포 가이드

## 개요

배포 순서:
1. SSM 파라미터 등록
2. SAM 빌드 & 배포 (Lambda + API Gateway)
3. Slack 앱 Events API URL 등록

---

## 1. SSM 파라미터 등록

Lambda 배포 전에 SSM Parameter Store에 값을 먼저 넣어야 한다.  
SAM이 `AWS::SSM::Parameter::Value<String>` 타입으로 배포 시점에 SSM에서 직접 값을 읽기 때문에, 파라미터가 없으면 배포가 실패한다.

### dev 환경

```bash
REGION=ap-northeast-2

# Slack 앱 서명 시크릿 (Slack App → Basic Information → Signing Secret)
aws ssm put-parameter \
  --name "/dev/slack/signing-secret" \
  --value "<signing_secret>" \
  --type String \
  --region $REGION

# Slack Bot Token (Slack App → OAuth & Permissions → Bot User OAuth Token, xoxb-...)
# Lambda 환경변수용 — AgentCore Identity Vault에도 별도로 등록 필요 (아래 참조)
aws ssm put-parameter \
  --name "/dev/slack/bot-token" \
  --value "<xoxb-...>" \
  --type String \
  --region $REGION

# AgentCore Runtime ARN (AgentCore 배포 완료 후 획득)
aws ssm put-parameter \
  --name "/dev/slack/agentcore/endpoint" \
  --value "<agentcore_runtime_arn>" \
  --type String \
  --region $REGION

# Slack 정기 보고를 수신할 채널 ID (예: C0123ABCDEF)
aws ssm put-parameter \
  --name "/dev/slack/report-channel" \
  --value "<channel_id>" \
  --type String \
  --region $REGION
```

### prod 환경

```bash
REGION=ap-northeast-2

aws ssm put-parameter \
  --name "/prod/slack/signing-secret" \
  --value "<signing_secret>" \
  --type String \
  --region $REGION

aws ssm put-parameter \
  --name "/prod/slack/bot-token" \
  --value "<xoxb-...>" \
  --type String \
  --region $REGION

aws ssm put-parameter \
  --name "/prod/slack/agentcore/endpoint" \
  --value "<agentcore_runtime_arn>" \
  --type String \
  --region $REGION

aws ssm put-parameter \
  --name "/prod/slack/report-channel" \
  --value "<channel_id>" \
  --type String \
  --region $REGION
```

### 파라미터 확인

```bash
# dev 전체 확인
aws ssm get-parameters-by-path \
  --path "/dev/" \
  --with-decryption \
  --region ap-northeast-2 \
  --query "Parameters[*].{Name:Name,Value:Value}"

# 개별 확인
aws ssm get-parameter \
  --name "/dev/slack/signing-secret" \
  --with-decryption \
  --region ap-northeast-2
```

### 파라미터 목록 정리

| 파라미터 경로 | 타입 | 값 출처 |
|---|---|---|
| `/dev/slack/signing-secret` | String | Slack App → Basic Information → Signing Secret |
| `/dev/slack/bot-token` | String | Slack App → OAuth & Permissions → Bot User OAuth Token |
| `/dev/slack/agentcore/endpoint` | String | AgentCore 배포 후 획득한 Runtime ARN |
| `/dev/slack/report-channel` | String | Slack 채널 ID (채널명 아님) |

> **주의:** `bot-token`은 AgentCore Agent가 Slack을 직접 호출할 때는 **AgentCore Identity Vault**에서 별도 관리한다 (아래 참조). SSM의 bot-token은 Lambda 환경변수용이다.

---

## 2. AgentCore Identity Vault — Slack Bot Token 등록

AgentCore Agent가 Slack API를 호출할 때 `@requires_api_key` 데코레이터로 토큰을 주입받는다.  
이 토큰은 SSM이 아닌 AgentCore Identity Vault에 별도로 등록해야 한다.

```bash
# agent/ 폴더에서 실행
cd agent

agentcore add credential \
  --name slack-bot-token \
  --api-key <xoxb-...>
```

토큰 로테이션 시 동일 명령을 재실행하면 재배포 없이 즉시 적용된다.

---

## 3. SAM 빌드 & 배포

### 사전 요구사항

```bash
# SAM CLI 설치 확인
sam --version   # 1.100+ 권장

# AWS 자격증명 확인
aws sts get-caller-identity
```

### dev 배포

```bash
cd lambda

# 빌드 (의존성 설치 + 패키징)
sam build

# 배포 (samconfig.toml의 [dev.deploy.parameters] 사용)
sam deploy --config-env dev
```

`confirm_changeset = false`이므로 변경사항 확인 없이 바로 배포된다.

### prod 배포

```bash
cd lambda

sam build

# confirm_changeset = true — changeset 내용을 확인 후 y 입력
sam deploy --config-env prod
```

### 배포 출력값 확인

배포 완료 후 CloudFormation Outputs에서 API 엔드포인트 URL을 확인한다:

```bash
aws cloudformation describe-stacks \
  --stack-name slack-aws-agent-dev \
  --region ap-northeast-2 \
  --query "Stacks[0].Outputs"
```

출력 예시:
```
SlackEventsEndpoint: https://<api-id>.execute-api.ap-northeast-2.amazonaws.com/dev/slack/events
```

---

## 4. Slack 앱 Events API URL 등록

SAM 배포 완료 후 출력된 `SlackEventsEndpoint` URL을 Slack 앱에 등록한다.

1. [Slack API Apps](https://api.slack.com/apps) → 해당 앱 선택
2. **Event Subscriptions** → Enable Events: On
3. **Request URL**에 `SlackEventsEndpoint` URL 입력
4. Slack이 `url_verification` 챌린지를 보내고 Lambda가 응답하면 ✓ Verified 표시
5. **Subscribe to bot events** → `app_mention`, `message.im` 추가
6. **Save Changes**

> URL 검증이 실패하면 Lambda 함수 로그(CloudWatch)에서 원인을 확인한다.

---

## 5. 재배포 및 파라미터 업데이트

### SSM 값만 변경할 때

SSM 파라미터 값을 수정한 뒤 Lambda를 재배포하면 새 값이 적용된다.  
(SAM은 배포 시점에 SSM 값을 읽어 Lambda 환경변수로 주입하기 때문)

```bash
# 값 덮어쓰기 (--overwrite 플래그 필요)
aws ssm put-parameter \
  --name "/dev/slack/agentcore/endpoint" \
  --value "<new_arn>" \
  --type String \
  --overwrite \
  --region ap-northeast-2

# 재배포
cd lambda && sam build && sam deploy --config-env dev
```

### 스택 삭제

```bash
aws cloudformation delete-stack \
  --stack-name slack-aws-agent-dev \
  --region ap-northeast-2
```
