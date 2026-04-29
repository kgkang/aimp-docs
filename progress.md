# 개발 진행 상태

> 작업 완료 시 [ ] → [x] 로 업데이트. 새 태스크는 해당 Phase에 추가.

## Phase 1: Lambda 기본 구조
> 참조: docs/02-lambda-spec.md

- [x] `lambda/template.yaml` — SAM API Gateway + Lambda 정의
- [x] `lambda/samconfig.toml` — dev/prod 배포 환경 설정
- [x] `lambda/src/requirements.txt` — boto3>=1.38.0 (CodeUri 루트에 위치, SAM 인식)
- [x] `lambda/src/slack_handler/handler.py` — Lambda A: Slack 수신 핸들러
  - [x] `verify_slack_signature()` — Slack 서명 검증
  - [x] `parse_event()` — 이벤트 타입 분기
  - [x] `build_response_target()` — 응답 대상 및 세션 ID 결정
  - [x] `strip_mention()` — 멘션 텍스트 제거
  - [x] `invoke_agentcore_async()` — Lambda B를 InvocationType=Event로 비동기 호출
- [x] `lambda/src/agentcore_invoker/handler.py` — Lambda B: AgentCore 호출 전담 (Slack DM/채널 멘션)
- [x] `lambda/src/scheduled_reporter/handler.py` — Lambda C: 정기 보고 전담
  - EventBridge Schedule(cron) → 직접 AgentCore Runtime 호출
  - 실행마다 고유 session_id(`scheduled-report-{uuid}`)로 히스토리 오염 방지
- [x] `lambda/template.yaml` — `ScheduledReporterFunction` + `ReportSchedule` 파라미터 추가
- [x] `sam build` 성공 확인
- [x] `url_verification` 로컬 테스트

## Phase 2: AgentCore Agent
> 참조: docs/03-agent-spec.md

- [x] `agent/app/SlackStrandAgent/main.py` — Strands SDK 기반 Agent 골격
  - [x] `invoke()` — Lambda payload(`input_text`, `session_id`, `response_target`) 수신
  - [x] `post_to_slack()` — `@requires_api_key`로 Identity Vault에서 토큰 자동 주입
- [x] `agent/agentcore/agentcore.json` — `credentials` 배열에 `slack-bot-token` 등록
- [ ] AgentCore Identity Vault에 Slack Bot Token 등록
  - [ ] `agentcore add credential --name slack-bot-token --api-key <token>`
- [ ] `agent/app/SlackAws/requirements.txt`
- [ ] `agent/app/SlackAws/tools.py` — AWS + Slack 툴 4종
  - [ ] `get_monthly_cost()`
  - [ ] `get_idle_resources()`
  - [ ] `list_running_instances()`
  - [ ] `send_slack_message()`
- [ ] `agent/app/SlackAws/agent.py` — LangGraph ReAct Agent
- [ ] `agentcore validate` 통과
- [ ] 로컬 단위 테스트: 각 툴 개별 동작 확인

## Phase 3: 연동 및 E2E 검증
> 두 컴포넌트 연결 + 실제 AWS/Slack 연동

- [ ] SSM 파라미터 생성 (dev 환경)
  - [ ] `/dev/slack/signing-secret`
  - [ ] `/dev/agentcore/endpoint`
  - [ ] `/dev/slack/report-channel`
- [ ] AgentCore Identity Vault 설정
  - [ ] `agentcore add credential --name slack-bot-token --api-key <token>`
- [ ] `sam deploy --config-env dev` 성공
- [ ] `agentcore deploy` (dev) 성공
- [ ] Lambda → AgentCore 연동 테스트
- [ ] Slack DM E2E 테스트
- [ ] Slack 채널 멘션 E2E 테스트
- [ ] EventBridge 정기 보고 수동 트리거 테스트
- [ ] prod 배포

## 설계 결정 기록

| 날짜 | 결정 | 이유 |
|------|------|------|
| 2026-04-26 | Lambda → AgentCore 비동기 호출 방식 | Slack 3초 응답 제한 대응 |
| 2026-04-26 | 세션 ID: thread_ts/user_id 사용 | thread/DM 단위 대화 컨텍스트 유지 |
| 2026-04-26 | AgentCore 직접 Slack 응답 | Lambda 폴링 불필요, 아키텍처 단순화 |
| 2026-04-27 | boto3 서비스명: `bedrock-agentcore`, 메서드: `invoke_agent_runtime` | `bedrock-agentcore-runtime`은 유효하지 않은 서비스명 |
| 2026-04-27 | requirements.txt 위치: `lambda/src/` (CodeUri 루트) | SAM PythonPipBuilder가 CodeUri 루트에서 requirements.txt를 탐색 |
| 2026-04-27 | Lambda A→B 분리 (InvocationType=Event) | threading.Thread는 Lambda freeze 시 실행 보장 없음, AWS 재시도 보장 필요 |
| 2026-04-27 | Slack Bot Token: AgentCore Identity Vault 사용 | 환경변수/SSM 대비 토큰 미노출, 감사 로그, 재배포 없는 로테이션 가능 |
| 2026-04-28 | 정기 보고: 전용 Lambda C(ScheduledReporterFunction) 분리 | AgentCoreInvoker와 관심사 분리, IAM 최소 권한 명확화, 스케줄 독립 관리 |
| 2026-04-28 | 정기 보고 session_id: 실행마다 `scheduled-report-{uuid}` | 고정 세션 시 히스토리 누적 방지, AgentCore 33자 최소 요건 충족 |
