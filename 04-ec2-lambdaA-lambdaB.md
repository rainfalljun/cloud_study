# 04. EC2 감시 (LambdaA + LambdaB) — 과제 내용

> 주제: EC2 상태 점검 → SNS 알림 + 다른 람다 비동기 호출로 EC2 시작

## 1) 문제 (과제 내용 그대로)

**Lambda 함수 2개 생성:**
- `sgu-202628-LambdaA`
- `sgu-202628-LambdaB`

**공통 설정:**
- 두 람다 모두 IAM 역할 `SafeRole-{username}` 사용
- 런타임: Python 3.14

**LambdaA 역할:**
- 지정된 EC2 인스턴스 상태 점검
- `running`이 아닐 경우:
  1. SNS 주제로 이메일 알림 발송
     - 메시지: `경고: EC2 인스턴스 i-037391d324f052fa2 상태가 'stopped'입니다.`
  2. LambdaB를 **비동기** 방식으로 호출

**LambdaB 역할:**
- 전달받은 EC2 인스턴스 ID로 `start_instances()` 호출 → EC2 시작

**필요 권한** (`SafeRole-{username}`에 포함):
- `ec2:DescribeInstanceStatus`
- `sns:Publish`
- `lambda:InvokeFunction`
- `ec2:StartInstances`

**SNS:** 실습시간에 생성한 SNS 사용해서 이메일 보냄

**주의사항:** 코드에 주석 처리 / 설계 의도(왜 나눴는지, 왜 비동기인지, 실무 장점) + 느낀점 작성

---

## 2) 흐름

### LambdaA

```
EC2 상태 점검 (describe_instance_status)
        ↓
running 이면 -> 그냥 종료 (정상)
        ↓
running 이 아니면:
   (1) SNS로 이메일 경고 발송 (sns.publish)
   (2) LambdaB를 비동기 호출 (invoke, InvocationType='Event')
       이때 인스턴스 ID를 payload로 같이 넘김
```

### LambdaB

```
전달받은 instance_id 로 EC2 시작 (start_instances)
```

---

## 3) 해결방식 (단계별 따라하기)

### (A) SNS 준비 (이메일 알림용)

1. **SNS > 주제(Topic) 생성** (또는 실습 때 만든 것 사용). 예: `sgu-202628-topic`
2. **구독(Subscription) 만들기** > 프로토콜: **이메일** > 내 이메일 입력
3. 메일함에서 **"Confirm subscription"** 클릭해서 구독 확정 (⭐ 안 하면 메일 안 옴)
4. **주제 ARN 복사**해두기 (`arn:aws:sns:ap-northeast-2:계정ID:sgu-202628-topic`)

### (B) LambdaB 먼저 생성 (A가 B를 부르므로 B가 있어야 함)

5. Lambda > 함수 생성 > 이름: `sgu-202628-LambdaB`
6. 런타임: **Python 3.14** (문제 지시 따름. 없으면 3.13)
7. 권한(역할): `SafeRole-{username}` (시험 지정 역할)
8. 아래 LambdaB 코드 붙여넣고 **Deploy**
9. 타임아웃 **10초**로 변경 권장

### (C) LambdaA 생성

10. Lambda > 함수 생성 > 이름: `sgu-202628-LambdaA`
11. 런타임/역할 동일하게
12. 아래 LambdaA 코드 붙여넣기
13. 코드 안의 `instance_id`, `sns_topic_arn`, LambdaB 함수이름을 **내 값으로 수정**
14. **Deploy** > 타임아웃 10초

### (D) 테스트 (스크린샷용)

15. EC2를 **"중지(stop)"** 상태로 만들어 둠
16. LambdaA 테스트 실행
17. **확인할 것 (테스트 스크린샷 3종):**
    - SNS 알림 수신 이메일
    - 람다 호출 로그 (CloudWatch Log)
    - EC2 중지화면 → 시작화면 (LambdaB가 EC2를 켜는지)

---

## 4) 실전용 코드 (그대로 복붙)

### 4-1. LambdaA — `sgu-202628-LambdaA`

```python
import boto3
import json

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    sns = boto3.client('sns')
    lambda_client = boto3.client('lambda')

    instance_id = 'i-037391d324f052fa2'
    sns_topic_arn = 'arn:aws:sns:ap-northeast-2:443370697536:sgu-202628-topic'

    # EC2 상태 점검 (중지 상태도 보려면 IncludeAllInstances=True)
    response = ec2.describe_instance_status(
        InstanceIds=[instance_id],
        IncludeAllInstances=True
    )

    statuses = response['InstanceStatuses']
    if not statuses:
        return {'statusCode': 404, 'body': f'인스턴스 {instance_id} 상태를 찾을 수 없음'}

    state = statuses[0]['InstanceState']['Name']

    if state != 'running':
        # 1) SNS 이메일 경고 발송
        message = f"경고: EC2 인스턴스 {instance_id} 상태가 '{state}'입니다."
        sns.publish(
            TopicArn=sns_topic_arn,
            Subject='EC2 상태 경고',
            Message=message
        )

        # 2) LambdaB 비동기 호출 (인스턴스 ID 전달)
        lambda_client.invoke(
            FunctionName='sgu-202628-LambdaB',
            InvocationType='Event',
            Payload=json.dumps({'instance_id': instance_id})
        )

        return {'statusCode': 200, 'body': f'알림 발송 + LambdaB 호출 완료 (현재 상태: {state})'}

    return {'statusCode': 200, 'body': f'정상 작동 중 (상태: {state})'}
```

### 4-2. LambdaB — `sgu-202628-LambdaB`

```python
import boto3
import json

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    instance_id = event['instance_id']

    ec2.start_instances(InstanceIds=[instance_id])

    return {
        'statusCode': 200,
        'body': json.dumps(f'{instance_id} 시작 요청 완료', ensure_ascii=False)
    }
```

---

## 5) 주석용 코드 (시험 변수 다를 때 한 줄씩 보고 수정)

### LambdaA — 주석판

```python
import boto3                                           # AWS 제어 모듈
import json                                            # payload를 JSON 문자열로 만들 때 사용

def lambda_handler(event, context):                    # 람다 진입점
    ec2 = boto3.client('ec2')                          # EC2 제어 클라이언트
    sns = boto3.client('sns')                          # SNS(알림) 클라이언트
    lambda_client = boto3.client('lambda')             # 다른 람다를 호출하기 위한 클라이언트

    instance_id = 'i-037391d324f052fa2'                # <- 점검할 EC2 인스턴스 ID 수정
    sns_topic_arn = 'arn:aws:sns:ap-northeast-2:443370697536:sgu-202628-topic'
                                                       # <- SNS 주제 ARN 수정 (SNS 콘솔에서 복사)

    response = ec2.describe_instance_status(           # EC2 상태 조회
        InstanceIds=[instance_id],                     # 점검할 인스턴스
        IncludeAllInstances=True                       # ★ 중지 상태도 보이게 (이거 없으면 stopped는 결과에 안 나옴)
    )

    statuses = response['InstanceStatuses']            # 상태 목록 추출
    if not statuses:                                   # 결과가 비면(없는 ID 등)
        return {'statusCode': 404, 'body': f'인스턴스 {instance_id} 상태를 찾을 수 없음'}

    state = statuses[0]['InstanceState']['Name']       # 실제 상태 문자열 (running/stopped/...)

    if state != 'running':                             # running이 아니면 (이상 상황)
        message = f"경고: EC2 인스턴스 {instance_id} 상태가 '{state}'입니다."  # <- 알림 문구(과제 형식 그대로)
        sns.publish(                                   # 이메일 알림 발송
            TopicArn=sns_topic_arn,                    # 어느 주제로 보낼지
            Subject='EC2 상태 경고',                    # 메일 제목
            Message=message                            # 메일 본문
        )

        lambda_client.invoke(                          # LambdaB 호출
            FunctionName='sgu-202628-LambdaB',         # <- 호출할 람다 이름 수정
            InvocationType='Event',                    # ★ 'Event' = 비동기(응답 안 기다림). 동기는 'RequestResponse'
            Payload=json.dumps({'instance_id': instance_id})  # B에게 인스턴스ID 전달 (JSON 문자열로)
        )

        return {'statusCode': 200, 'body': f'알림 발송 + LambdaB 호출 완료 (현재 상태: {state})'}

    return {'statusCode': 200, 'body': f'정상 작동 중 (상태: {state})'}  # running이면 여기서 끝
```

### LambdaB — 주석판

```python
import boto3
import json

def lambda_handler(event, context):                    # 람다 진입점 (event 안에 A가 보낸 데이터가 들어옴)
    ec2 = boto3.client('ec2')                          # EC2 제어 클라이언트
    instance_id = event['instance_id']                 # <- A가 Payload로 넘긴 인스턴스 ID 꺼내기
                                                       #    (A의 Payload 키 이름과 똑같아야 함!)
    ec2.start_instances(InstanceIds=[instance_id])     # 해당 EC2 시작(켜기)
    return {
        'statusCode': 200,
        'body': json.dumps(f'{instance_id} 시작 요청 완료', ensure_ascii=False)
    }
```

---

## 6) 교수님 요구사항: 설계 의도 (보고서/답안에 쓸 내용)

### ▶ LambdaA와 LambdaB를 왜 나누었는가?

- **역할 분리(책임 분리)** 때문입니다.
  - LambdaA는 "감시/판단(상태 점검 후 알림 여부 결정)"을 맡고,
  - LambdaB는 "실행(EC2 시작)"이라는 별개의 동작을 맡습니다.
- 이렇게 나누면 각 함수가 한 가지 일만 하므로 **코드가 단순**해지고, 한쪽을 고쳐도 다른 쪽에 **영향이 적어 유지보수가 쉽**습니다.
- LambdaB(EC2 시작)는 **다른 곳에서도 재사용**할 수 있어 확장성이 좋습니다.

### ▶ 비동기 호출을 왜 사용했는가?

- LambdaA의 핵심 임무는 **"이상 감지 + 알림"** 을 빠르게 끝내는 것입니다.
- EC2 시작은 시간이 걸리는 작업인데, 동기로 호출하면 A가 B의 완료를 끝까지 기다려야 하고, 그동안 A의 실행시간(=비용)도 같이 늘어납니다.
- **비동기(`InvocationType='Event'`)** 로 호출하면 A는 "B야 시작해" 하고 던진 뒤 바로 자기 할 일을 끝냅니다. B는 별도로 알아서 EC2를 켭니다.
- 즉, **알림은 즉시 나가고 / 복구 작업은 백그라운드로 진행**되어 응답이 빠릅니다.

### ▶ 이런 구조가 실무에서 어떤 장점이 있는가?

- **느슨한 결합(loose coupling):** A와 B가 직접 묶여있지 않아 한쪽 장애가 다른 쪽으로 잘 번지지 않습니다.
- **확장성:** 나중에 "EC2 시작" 외에 "관리자 슬랙 알림", "로그 기록" 같은 후속 작업을 또 다른 람다로 추가하기 쉽습니다.
- **비용/성능:** 감시 함수는 짧게 끝나 비용이 적게 들고, 무거운 작업은 분리되어 전체 응답이 빠릅니다.
- **이벤트 기반 아키텍처**(이상 발생 → 이벤트 → 자동 대응)의 기본 패턴입니다.

### ▶ 느낀점 (예시 — 본인 말로 바꿔 쓰기)

- 처음엔 함수 하나로 다 해도 되지 않나 싶었는데, 역할을 나누니 코드가 훨씬 깔끔하고 "왜 비동기인지"가 직접 와닿았다.
- 알림과 복구를 분리해두니 실제 서비스에서 장애 대응이 이렇게 자동화되겠구나 하는 그림이 그려졌다.
- SNS 구독 확인을 안 해서 메일이 안 와 헤맸는데, 작은 설정 하나가 전체 동작을 좌우한다는 걸 배웠다.

---

## 🚨 자주 나는 에러 / 체크포인트

- **SNS 메일이 안 옴** → 구독(Subscription) **"Confirm"** 안 함. 메일함 확인.
- **`describe_instance_status` 결과가 비어있음** → `IncludeAllInstances=True` 빠짐 (중지된 인스턴스는 기본 조회에서 안 나옴).
- **LambdaB가 `instance_id`를 못 받음** → A의 `Payload` 키와 B의 `event[...]` 키 이름 불일치.
- **`AccessDenied`** → 역할에 `ec2:StartInstances` / `sns:Publish` / `lambda:InvokeFunction` 권한 확인.
- **비동기는 `InvocationType='Event'`** (동기는 `'RequestResponse'`). 문제에서 "비동기" 명시됨.
