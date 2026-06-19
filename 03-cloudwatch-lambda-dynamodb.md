# 03. CloudWatch Log + Lambda + DynamoDB

> 주제: Lambda가 CloudWatch 로그를 조회해서 DynamoDB에 저장

## 1) 문제 (이렇게 나올 것)

> Lambda가 CloudWatch 로그를 조회해서 DynamoDB에 저장하시오.

**저장 규칙(과제 기준):**
- `user_id` = `"cloudwatch"`
- `timestamp` = 로그 발생 시각 (또는 현재시간)
- `product` = CloudWatch 로그 `message` 내용

---

## 2) 흐름

```
Lambda 실행
        ↓
boto3 logs 클라이언트로 로그 그룹(log group)의 최신 로그 스트림 조회
        ↓
그 스트림의 로그 이벤트(events)들을 가져옴
        ↓
이벤트마다 timestamp(밀리초)를 사람이 읽는 시간으로 변환
        ↓
각 로그를 DynamoDB에 put_item 으로 저장
```

---

## 3) 해결방식 (단계별 따라하기)

### (A) 조회할 로그 그룹 정하기

- 보통 **다른 람다(예: S3 람다)의 로그**를 조회함.
- 로그 그룹 형식: `/aws/lambda/<함수이름>`
  - 예) `/aws/lambda/sgu-202628-lambda-s3`
- **확인법:** 그 람다 > 모니터링 > **CloudWatch Logs 보기** 클릭하면 그룹 경로가 보임.

### (B) Lambda 생성

1. Lambda > 함수 생성 > 이름: `sgu-202628-cloudwatch`
2. 런타임: **Python 3.13**
3. 권한(역할): 시험에서 지정한 역할 선택 (**CloudWatch Logs 읽기 권한** 필요)
4. 코드 붙여넣고 **Deploy**

### (C) ⭐ 타임아웃 10초로 변경

5. 구성 > 일반 구성 > 편집 > 제한시간 **10초** > 저장
   (로그 조회도 시간이 걸려서 3초로는 부족할 수 있음)

### (D) 테스트 > DynamoDB 에서 `user_id='cloudwatch'` 항목들 확인

### 참고: CloudWatch 로그 이벤트 1개 구조 (필드 3개)

```json
{
  "timestamp": 1716728459247,    // 로그 발생 시각 (Epoch 밀리초)
  "message": "로그 본문 내용",      // 로그 메시지 (1줄 문자열)
  "ingestionTime": 1716728460000  // CloudWatch가 수집한 시간
}
```

→ `timestamp`는 **밀리초**라서 `1000으로 나눠 초로 바꾼 뒤` 시간 변환함.

---

## 4) 실전용 코드 (그대로 복붙)

```python
import boto3
import json
from datetime import datetime

def lambda_handler(event, context):
    logs = boto3.client('logs')
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('sgu-202628-user-likes-time')

    log_group = '/aws/lambda/sgu-202628-lambda-s3'

    try:
        # 1. 최근 로그 스트림 가져오기
        streams = logs.describe_log_streams(
            logGroupName=log_group,
            orderBy='LastEventTime',
            descending=True,
            limit=1
        )['logStreams']

        if not streams:
            return {'statusCode': 404, 'body': 'No log streams found'}

        stream_name = streams[0]['logStreamName']

        # 2. 로그 이벤트 가져오기
        events = logs.get_log_events(
            logGroupName=log_group,
            logStreamName=stream_name,
            limit=20,
            startFromHead=True
        )['events']

        # 3. 로그 이벤트를 하나씩 DynamoDB에 저장
        for e in events:
            event_time = datetime.fromtimestamp(e['timestamp'] / 1000).isoformat()
            message = e['message'].strip()
            item = {
                'user_id': 'cloudwatch',
                'timestamp': event_time,
                'product': message
            }
            table.put_item(Item=item)
            print(f"Inserted: {item}")

        return {
            'statusCode': 200,
            'body': json.dumps('CloudWatch 로그 → DynamoDB 저장 완료', ensure_ascii=False)
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)}, ensure_ascii=False)
        }
```

---

## 5) 주석용 코드 (시험 변수 다를 때 한 줄씩 보고 수정)

```python
import boto3                                          # AWS 제어 모듈
import json                                           # JSON 처리
from datetime import datetime                         # 시간 변환용

def lambda_handler(event, context):                   # 람다 진입점
    logs = boto3.client('logs')                       # CloudWatch Logs 클라이언트 ('logs' 맞음)
    dynamodb = boto3.resource('dynamodb')             # DynamoDB 리소스
    table = dynamodb.Table('sgu-202628-user-likes-time')  # <- 저장할 테이블명 수정

    log_group = '/aws/lambda/sgu-202628-lambda-s3'    # <- 조회할 로그그룹 수정
                                                      #    형식: /aws/lambda/<조회할 함수이름>
    try:
        # 1) 가장 최근 로그 스트림 1개 찾기
        streams = logs.describe_log_streams(
            logGroupName=log_group,                   # 위에서 정한 로그 그룹
            orderBy='LastEventTime',                  # 마지막 이벤트 시간 기준 정렬
            descending=True,                          # 내림차순(=최신이 맨 위)
            limit=1                                   # 1개만 (최신 스트림)
        )['logStreams']                               # 결과에서 스트림 목록만 추출

        if not streams:                               # 스트림이 없으면
            return {'statusCode': 404, 'body': 'No log streams found'}  # 종료

        stream_name = streams[0]['logStreamName']     # 최신 스트림 이름 꺼내기

        # 2) 그 스트림의 로그 이벤트들 가져오기
        events = logs.get_log_events(
            logGroupName=log_group,                   # 같은 로그 그룹
            logStreamName=stream_name,                # 위에서 찾은 스트림
            limit=20,                                 # <- 가져올 로그 개수. 더/덜 필요하면 수정
            startFromHead=True                        # 앞(오래된 것)부터 읽기
        )['events']                                   # 이벤트 목록만 추출

        # 3) 이벤트마다 DynamoDB에 저장
        for e in events:                              # 로그 이벤트 하나씩
            event_time = datetime.fromtimestamp(e['timestamp'] / 1000).isoformat()
                                                      # timestamp는 밀리초 -> 1000으로 나눠 초 -> 시간 문자열
            message = e['message'].strip()            # 로그 본문(양 끝 공백 제거)
            item = {
                'user_id': 'cloudwatch',              # <- 문제에서 user_id 지정값 다르면 수정
                'timestamp': event_time,              # 정렬키(로그시각). 현재시간 원하면 datetime.now().isoformat()
                'product': message                    # <- 저장할 본문. 컬럼명 다르면 수정
            }
            table.put_item(Item=item)                 # 저장
            print(f"Inserted: {item}")                # 로그 출력

        return {'statusCode': 200,
                'body': json.dumps('CloudWatch 로그 → DynamoDB 저장 완료', ensure_ascii=False)}
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)}, ensure_ascii=False)}
```

---

## 🚨 자주 나는 에러 / 체크포인트

- **`boto3.client('logs')`** → `'cloudwatch'`가 아니라 **`'logs'`** 입니다. (자주 헷갈림!)
- **`ResourceNotFoundException`** → 로그 그룹 경로 오타. `/aws/lambda/` 까지 포함했는지 확인.
- **`AccessDenied`** → 역할에 CloudWatch Logs 읽기 권한 부족.
- 로그 안에 **같은 `timestamp`가 여럿이면** 일부가 덮어써질 수 있음 (정렬키 충돌).
- **타임아웃 10초** 변경 잊지 말기.
