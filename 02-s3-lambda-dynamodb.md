# 02. S3 + Lambda + DynamoDB

> 주제: S3 JSON 파일을 Lambda가 읽어서 DynamoDB에 저장

## 1) 문제 (이렇게 나올 것)

> S3 버킷에 있는 JSON 파일을 Lambda가 읽어서 DynamoDB에 INSERT 하시오.

**응용으로:**
- 특정 필드(key)만 골라서 저장
- JSON이 배열(`[ ... ]`)로 여러 개 들어있을 때 전부 저장
- 저장할 때 `timestamp`(현재시간) 추가

---

## 2) 흐름

```
S3 버킷에 JSON 파일 업로드
        ↓
Lambda가 boto3로 S3에서 그 파일을 읽음 (get_object)
        ↓
읽은 내용을 json.loads 로 파이썬 딕셔너리로 변환
        ↓
timestamp(현재시간) 추가
        ↓
DynamoDB 테이블에 put_item 으로 저장
```

---

## 3) 해결방식 (단계별 따라하기)

### (A) 준비물 3개

S3 버킷 / DynamoDB 테이블 / Lambda 함수

### (B) S3에 JSON 올리기

1. **S3 > 내 버킷** 들어가기
2. 폴더 `dynamo_data` 만들고 그 안에 `user_likes_data.json` 업로드
3. 객체 키(경로): `dynamo_data/user_likes_data.json`

**user_likes_data.json 예시:**

```json
{
  "user_id": "haeri05",
  "product": "워치",
  "price": 50000,
  "brand": "애플"
}
```

### (C) DynamoDB 테이블 확인

- 테이블명: 시험에서 주는 이름 (예: `sgu-202628-user-likes-time`)
- **파티션 키:** `user_id` (문자열)
- **정렬 키:** `timestamp` (문자열) ← 보통 이 구조

### (D) Lambda 생성

4. Lambda > 함수 생성 > 이름: `sgu-202628-lambda-s3`
5. 런타임: **Python 3.13**
6. 권한(역할): 시험에서 지정한 역할 선택
7. 코드 붙여넣고 **Deploy**

### (E) ⭐ 타임아웃 변경 (매우 중요, 시험 단골)

8. **구성(Configuration) > 일반 구성 > 편집**
9. 제한 시간: `0분 3초` → `0분 10초` 로 변경 후 저장

   이유: S3 읽기 + JSON 파싱 + DynamoDB 쓰기가 3초 안에 안 끝나면
   `Sandbox.Timedout: Task timed out after 3.00 seconds` 에러 발생.

### (F) 테스트 > DynamoDB 항목 보기에서 데이터 들어왔는지 확인

---

## 4) 실전용 코드 (그대로 복붙)

### 4-1. 기본 — JSON 전체를 그대로 저장 (+ timestamp)

```python
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('sgu-202628-user-likes-time')

    bucket_name = 'sgu-202628-3b'
    object_key = 'dynamo_data/user_likes_data.json'

    try:
        response = s3.get_object(Bucket=bucket_name, Key=object_key)
        content = response['Body'].read().decode('utf-8')
        data = json.loads(content)

        timestamp = datetime.now().isoformat()
        data['timestamp'] = timestamp

        table.put_item(Item=data)

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'DynamoDB insert successful!',
                'item': data
            }, ensure_ascii=False)
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)}, ensure_ascii=False)
        }
```

### 4-2. 응용 — 필요한 필드(user_id, product)만 저장

```python
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('sgu-202628-user-likes-time')

    bucket_name = 'sgu-202628-3b'
    object_key = 'dynamo_data/user_likes_data_2.json'

    try:
        response = s3.get_object(Bucket=bucket_name, Key=object_key)
        content = response['Body'].read().decode('utf-8')
        raw_data = json.loads(content)

        user_id = raw_data.get('user_id')
        product = raw_data.get('product')

        if not user_id or not product:
            return {'statusCode': 400, 'body': '필수 데이터 누락: user_id 또는 product가 없습니다.'}

        item = {
            'user_id': user_id,
            'timestamp': datetime.now().isoformat(),
            'product': product
        }
        table.put_item(Item=item)

        return {
            'statusCode': 200,
            'body': json.dumps({'message': '선택 필드만 Insert 완료', 'item': item}, ensure_ascii=False)
        }
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)}, ensure_ascii=False)}
```

### 4-3. 응용 — JSON이 배열(`[ ... ]`)일 때 전부 저장

```python
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('sgu-202628-user-likes-time')

    bucket_name = 'sgu-202628-3b'
    object_key = 'dynamo_data/user_likes_data.json'

    try:
        response = s3.get_object(Bucket=bucket_name, Key=object_key)
        content = response['Body'].read().decode('utf-8')
        data_list = json.loads(content)

        for record in data_list:
            user_id = record.get('user_id')
            product = record.get('product')
            if not user_id or not product:
                print(f"user_id 또는 product 누락: {record}")
                continue
            item = {
                'user_id': user_id,
                'timestamp': datetime.now().isoformat(),
                'product': product
            }
            table.put_item(Item=item)
            print(f"Inserted: {item}")

        return {'statusCode': 200, 'body': json.dumps('전체 레코드 Insert 완료', ensure_ascii=False)}
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)}, ensure_ascii=False)}
```

---

## 5) 주석용 코드 (시험 변수 다를 때 한 줄씩 보고 수정)

### 기본형(전체 저장) — 주석판

```python
import json                                           # JSON 다루는 기본 모듈
import boto3                                          # AWS 제어용 모듈 (S3, DynamoDB 등)
from datetime import datetime                         # 현재 시간 얻기용

def lambda_handler(event, context):                   # 람다 진입점
    s3 = boto3.client('s3')                           # S3 클라이언트 생성
    dynamodb = boto3.resource('dynamodb')             # DynamoDB 리소스 생성
    table = dynamodb.Table('sgu-202628-user-likes-time')  # <- 테이블명 수정 (시험에서 주는 이름)

    bucket_name = 'sgu-202628-3b'                     # <- 버킷명 수정 (시험에서 주는 버킷)
    object_key = 'dynamo_data/user_likes_data.json'   # <- 파일 경로(키) 수정 (폴더/파일명)

    try:                                              # 에러나면 아래 except로 빠짐
        response = s3.get_object(Bucket=bucket_name, Key=object_key)  # S3에서 파일 가져오기
        content = response['Body'].read().decode('utf-8')             # 바이트 -> 문자열(utf-8)로 변환
        data = json.loads(content)                    # JSON 문자열 -> 파이썬 딕셔너리

        timestamp = datetime.now().isoformat()        # 현재시간을 ISO형식 문자열로 (예 2025-06-19T10:00:00)
        data['timestamp'] = timestamp                 # 딕셔너리에 timestamp 키 추가
                                                      # (※ 정렬키가 timestamp면 이게 있어야 저장됨)
        table.put_item(Item=data)                     # DynamoDB에 통째로 저장(INSERT)

        return {                                      # 성공 응답
            'statusCode': 200,
            'body': json.dumps({
                'message': 'DynamoDB insert successful!',
                'item': data
            }, ensure_ascii=False)                    # ensure_ascii=False -> 한글이 깨지지 않게
        }
    except Exception as e:                            # 에러 발생 시
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)}, ensure_ascii=False)  # 에러 메시지 반환(디버깅용)
        }
```

### 특정 필드만 저장 — 핵심 줄 주석

```python
user_id = raw_data.get('user_id')                     # <- 가져올 필드명1 (.get은 없으면 None 반환=안전)
product = raw_data.get('product')                     # <- 가져올 필드명2. 필드 더 필요하면 줄 추가
if not user_id or not product:                        # 둘 중 하나라도 비면
    return {'statusCode': 400, 'body': '필수 데이터 누락...'}  # 400 에러로 중단
item = {                                              # 저장할 항목을 직접 구성
    'user_id': user_id,                               # 파티션키
    'timestamp': datetime.now().isoformat(),          # 정렬키(현재시간)
    'product': product                                # 추가 저장 필드
}
```

### 배열 처리 — 핵심 줄 주석

```python
data_list = json.loads(content)                       # 파일 내용이 [ {...}, {...} ] 형태 -> 리스트로 파싱
for record in data_list:                              # 리스트를 하나씩 꺼내서 반복
    ...                                               # (각 record를 위 방식대로 item 만들어 put_item)
    # 주의: 같은 user_id에 timestamp가 동일하면 덮어써질 수 있음.
    #       반복이 매우 빠르면 시간이 같아질 수 있으니 시험에서 데이터가 1~2건 빠지면 이걸 의심.
```

---

## 🚨 자주 나는 에러 / 체크포인트

- **`Sandbox.Timedout: Task timed out after 3.00 seconds`**
  → 타임아웃 3초가 원인. 구성 > 일반 구성 > 편집 > 제한시간 **10초로**.
- **`AccessDenied` / `NoSuchKey`**
  → 버킷명/객체키 오타 또는 역할 권한 문제. 경로 다시 확인.
- **한글 깨짐** → `json.dumps(..., ensure_ascii=False)` 빠졌는지 확인.
- `price` 같은 숫자는 정수면 그대로 저장됨. 소수점(float)이 들어오면 DynamoDB가 싫어할 수 있음.
