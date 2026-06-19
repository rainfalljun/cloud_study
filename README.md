# AWS 실기 기말시험 정리 (오픈북)

> 내 아이디: **sgu-202628** · 리전: **ap-northeast-2 (서울)**

## 🔍 빠른 사용법

- 화면 **왼쪽 사이드바**에서 주제 클릭
- **상단 검색창** (또는 `Ctrl+K`)에 키워드 입력 — 예: `timeout`, `boto3`, `put_item`, `InvocationType`
- 코드 블록 우측 상단 **복사** 버튼으로 한 번에 복붙

## 📚 주제 목록

| # | 주제 | 핵심 |
|---|------|------|
| [00](00-summary.md) | 시험 요약 / 공통 설정 | 체크리스트, 자주 고치는 값 |
| [01](01-lambda-layer.md) | Lambda Layer | emoji / requests 외부 라이브러리 |
| [02](02-s3-lambda-dynamodb.md) | S3 + Lambda + DynamoDB | JSON 읽어 DynamoDB 저장 |
| [03](03-cloudwatch-lambda-dynamodb.md) | CloudWatch + Lambda + DynamoDB | 로그 조회해서 저장 |
| [04](04-ec2-lambdaA-lambdaB.md) | EC2 감시 (LambdaA + LambdaB) | SNS 알림 + 비동기 호출 |

## ⚠️ 시험장에서 거의 항상 고치는 값

- 람다 함수 이름 (`sgu-202628-???`)
- IAM 역할 (예: `SafeRoleForUser-sgu-202628` 또는 `SafeRole-{username}`)
- 런타임: **Python 3.13** (문제에서 3.14 지시 시 따름)
- **타임아웃: 기본 3초 → 10초로 변경** (구성 > 일반 구성 > 편집)
- S3 버킷 이름, 객체 키 (예: `dynamo_data/user_likes_data.json`)
- DynamoDB 테이블 이름
- EC2 인스턴스 ID, SNS 주제 ARN

> 자세한 체크리스트 → [00. 시험 요약 / 공통 설정](00-summary.md)
