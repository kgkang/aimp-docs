# 요구사항 정의

## 프로젝트 목표
AWS 리소스 모니터링 및 비용 분석 결과를 Slack으로 보고하고, Slack에서 자연어로 AWS 운영 질의를 할 수 있는 AI Agent 시스템 구축.

## 핵심 유스케이스

### UC-1: 정기 비용/리소스 보고 (Push)
- **트리거**: EventBridge 스케줄 (매주 월요일 오전 9시 KST 등)
- **동작**: AgentCore Agent가 AWS Cost Explorer + CloudWatch 조회 후 요약 보고서 생성
- **출력**: Slack 지정 채널(SLACK_REPORT_CHANNEL)에 포스팅
- **보고 내용**:
  - 이번 달 서비스별 비용 (전월 대비 증감)
  - 유휴 EC2/RDS 인스턴스 목록 (CPU < 5% 지속 시)
  - 비용 이상 징후 (전월 대비 20% 이상 증가 서비스)

### UC-2: Slack DM 질의응답 (Pull)
- **트리거**: 사용자가 봇에게 DM 전송
- **동작**: Lambda → AgentCore Agent 호출 → 자연어로 AWS 상태 답변
- **대화 컨텍스트**: user_id 기반으로 동일 사용자의 대화 이력 유지
- **예시 질의**:
  - "지난달 EC2 비용이 얼마야?"
  - "현재 실행 중인 RDS 인스턴스 알려줘"
  - "어떤 서비스가 비용을 제일 많이 쓰고 있어?"

### UC-3: Slack 채널 멘션 질의응답 (Pull)
- **트리거**: 채널에서 봇 @mention
- **동작**: Lambda → AgentCore Agent 호출 → 멘션된 thread에 답변
- **대화 컨텍스트**: thread_ts 기반으로 thread별 대화 이력 유지
- **예시**: `@aws-agent 이번 달 비용 요약해줘`

## 비기능 요구사항

| 항목 | 요구사항 |
|------|----------|
| 응답 시간 | Slack에 3초 이내 즉시 응답 (비동기 처리) |
| 가용성 | Lambda + API Gateway SLA 따름 |
| 보안 | Slack 서명 검증 필수, 환경변수 SSM 저장 |
| 비용 | Lambda 무료 티어 범위 내 운영 목표 |
| 봇 루프 방지 | bot_id 있는 이벤트 무조건 무시 |

## 범위 밖 (Out of Scope)
- AWS 리소스 변경 작업 (읽기 전용 Agent)
- 멀티 워크스페이스 Slack 지원
- 비용 예측/알림 (현재는 조회만)
