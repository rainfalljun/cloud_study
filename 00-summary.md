# 00. 시험 요약 / 공통 설정

> 내 아이디: **sgu-202628** · 리전: **ap-northeast-2 (서울)**

## 이 자료 사용법

- 시험 5문제 예상 주제별로 파일을 나눠놓음
- 각 페이지 구성:
  1. **문제** (이런 식으로 나올 것)
  2. **흐름** (데이터가 어디서 어디로 가는지)
  3. **해결방식** (콘솔에서 뭘 누르는지 단계별)
  4. **실전용 코드** (그대로 복붙)
  5. **주석용 코드** (시험 변수가 다를 때 한 줄씩 보고 수정)
- 시험장에서는 **주석용 코드**로 바꿔야 할 값(버킷명/테이블명/인스턴스ID 등)을 먼저 고친 다음, 그 결과를 람다에 붙여넣기

---

## 예상 5문제 주제

1. **[Lambda Layer](01-lambda-layer.md)** — emoji / requests 같은 외부 라이브러리를 람다에서 쓰기
2. **[S3 + Lambda + DynamoDB](02-s3-lambda-dynamodb.md)** — S3의 JSON 파일을 람다가 읽어 DynamoDB에 저장
3. **[CloudWatch Log + Lambda + DynamoDB](03-cloudwatch-lambda-dynamodb.md)** — 람다가 CloudWatch 로그를 조회해서 DynamoDB에 저장
4. **[EC2 감시 (LambdaA + LambdaB)](04-ec2-lambdaA-lambdaB.md)** — EC2 상태 점검 → SNS 알림 + 다른 람다 비동기 호출로 EC2 시작
5. **위 4가지 응용/변형 문제 가능성 높음** — 특정 필드만 저장, JSON 배열 처리, 타임아웃 변경 등. 각 페이지 안에 "번외/응용"으로 같이 포함.

---

## 시험장에서 거의 항상 고쳐야 하는 값 — 체크리스트

- [ ] **람다 함수 이름** : `sgu-202628-???`
- [ ] **IAM 역할(권한)** : 시험에서 지정해주는 역할 선택
       (예: `SafeRoleForUser-sgu-202628` 또는 `SafeRole-{username}`)
- [ ] **런타임(Runtime)** : Python 3.13 (과제는 3.14 지시 — 문제 지시 따를 것)
- [ ] **제한 시간(Timeout)** : 기본 3초 → **10초로 변경** (구성 > 일반 구성 > 편집)
- [ ] **S3 버킷 이름** : 시험에서 주는 본인 버킷명
- [ ] **S3 객체 키(파일 경로)** : 예) `dynamo_data/user_likes_data.json`
- [ ] **DynamoDB 테이블 이름** : 시험에서 주는 테이블명
- [ ] **EC2 인스턴스 ID** : 예) `i-037391d324f052fa2`
- [ ] **SNS 주제 ARN** : SNS 콘솔에서 복사 (`arn:aws:sns:ap-northeast-2:계정ID:주제이름`)

---

## 꼭 기억할 핵심 개념

- `boto3.client(...)` : AWS 서비스를 **명령어 단위**로 부를 때 (s3, logs, ec2, sns, lambda)
- `boto3.resource(...)` : **객체처럼** 다룰 때 (DynamoDB의 `Table` 쓸 때 주로 사용)
- 람다 **기본 타임아웃은 3초**. S3/DynamoDB 작업은 3초 넘을 수 있어서 **10초로 늘림**.
- DynamoDB는 보통 `user_id`(파티션키) + `timestamp`(정렬키) 조합.
  → `timestamp` 값이 같으면 덮어써지니 주의.
- 외부 라이브러리(`emoji`, `requests` 등)는 그냥 `import` 안 됨 → **Layer 필요**.
- **비동기 호출**(`InvocationType='Event'`) : 호출만 던지고 응답 안 기다림.
