# 01. Lambda Layer (람다 레이어)

> 주제: emoji / requests 같은 외부 라이브러리를 람다에서 쓰기

## 1) 문제 (이렇게 나올 것)

> emoji(또는 requests) 라이브러리를 사용하는 람다 함수를 만드시오.
> 단, 외부 라이브러리는 **Layer**를 만들어서 적용할 것.

**핵심:** 람다는 `emoji`, `requests` 같은 외부 라이브러리가 기본으로 없음. 그냥 `import` 하면 아래 에러가 뜸:

```
Unable to import module 'lambda_function': No module named 'emoji'
```

그래서 라이브러리를 zip으로 묶어서 **Layer(계층)** 로 올린 뒤 람다에 붙여야 함.

---

## 2) 흐름

```
내 PC에서 라이브러리 설치 (python 폴더에)
        ↓
python 폴더를 zip으로 압축 (python.zip)
        ↓
AWS 콘솔 > Lambda > 계층(Layer) 생성 > zip 업로드
        ↓
람다 함수에 그 계층(Layer) 추가
        ↓
람다 코드에서 import 해서 사용 → 성공
```

---

## 3) 해결방식 (단계별 따라하기)

> ⚠️ **매우 중요:** 라이브러리를 설치할 폴더 이름은 반드시 **`python`** 이어야 합니다. 람다가 `python` 폴더 안을 찾도록 약속되어 있음. 다른 이름이면 import 실패.

### (A) 내 PC에서 라이브러리 준비

1. 작업 폴더 만들기. 예) `D:\aws_3c`
2. CMD(명령 프롬프트) 켜서 그 폴더로 이동: `cd D:\aws_3c`
3. 파이썬 버전 확인: `py -3.13 -V` (3.13 아니면 3.13 설치, 설치 때 **"Add to PATH" 체크!**)
4. `python` 폴더 안에 라이브러리 설치:

   ```bash
   py -3.13 -m pip install emoji -t python/
   ```

   (`-t python/` → `python` 이라는 폴더를 만들고 그 안에 설치하라는 뜻)
5. 생긴 `python` 폴더를 통째로 zip 압축 → `python.zip`
   (`python` 폴더를 **우클릭 > 압축**. zip 안에 `python` 폴더가 들어있어야 함)

### (B) AWS 콘솔에서 계층(Layer) 생성

6. **Lambda > 추가 리소스 > 계층 > 계층 생성**
7. 이름: `sgu-202628-layer`
8. 설명: `emoji install` (아무거나 OK)
9. **`.zip 파일 업로드`** 선택 > `python.zip` 올리기
10. 호환 런타임: **Python 3.13** 선택
11. **생성** 클릭 → 버전 1 생성 확인

### (C) 람다 함수 생성 + 계층 적용

12. Lambda > 함수 생성 > 이름: `sgu-202628-layertest`
13. 런타임: **Python 3.13**
14. 권한(역할): 시험에서 지정한 역할 선택
15. 함수 들어가서 아래로 스크롤 > **Layers** 클릭 > **Add a layer**
16. **사용자 지정 계층** 선택 > `sgu-202628-layer` > 버전 1 > 추가
17. 코드 붙여넣고 **Deploy** > **테스트** → 성공 확인

### (D) 라이브러리를 "추가"하고 싶을 때 (예: emoji에 더해 requests도)

- 같은 `python` 폴더 경로에 requests도 설치:

  ```bash
  py -3.13 -m pip install requests -t python/
  ```
- 다시 zip 만들고 → 만들었던 레이어에서 **"버전 생성"** 으로 새 버전 업로드
- 람다에서 계층을 새 버전(예: 버전 2)으로 바꿔서 추가

---

## 4) 실전용 코드 (그대로 복붙)

### 4-1. emoji 테스트용 람다 코드

```python
import emoji

def lambda_handler(event, context):
    text = "AWS Lambda is awesome! :rocket:"
    result = emoji.emojize(text, language='alias')
    print(result)
    return {
        'statusCode': 200,
        'body': result
    }
```

### 4-2. requests GET 코드

```python
import requests

def lambda_handler(event, context):
    url = "https://httpbin.org/get"
    response = requests.get(url)
    data = response.json()
    print("API 응답:", data)
    return {
        'statusCode': 200,
        'body': str(data)
    }
```

### 4-3. requests POST 코드

```python
import requests
import json

def lambda_handler(event, context):
    url = "https://httpbin.org/post"
    payload = {"name": "test"}
    response = requests.post(url, json=payload)
    data = response.json()
    print("API 응답:", data)
    return {
        'statusCode': 200,
        'body': str(data)
    }
```

---

## 5) 주석용 코드 (시험 변수 다를 때 한 줄씩 보고 수정)

### emoji — 주석판

```python
import emoji                                          # Layer로 올린 emoji 라이브러리 불러오기

def lambda_handler(event, context):                   # 람다 진입점 (이름 바꾸지 말 것)
    text = "AWS Lambda is awesome! :rocket:"          # <- 변환할 문장. :rocket: 처럼 :별칭: 형태가 이모지로 바뀜
    result = emoji.emojize(text, language='alias')    # alias = :rocket: 같은 영어 별칭을 이모지로 변환하는 모드
    print(result)                                     # CloudWatch 로그에 결과 출력 (테스트 스크린샷용)
    return {                                          # 람다 응답 반환
        'statusCode': 200,                            # 성공 코드
        'body': result                                # 변환된 문자열을 body에 담음
    }
```

### requests GET — 주석판

```python
import requests                                       # Layer로 올린 requests 라이브러리

def lambda_handler(event, context):
    url = "https://httpbin.org/get"                   # <- 요청 보낼 주소. 다른 URL이면 여기 수정
    response = requests.get(url)                      # GET 방식으로 요청 전송
    data = response.json()                            # 응답(JSON)을 파이썬 딕셔너리로 변환
    print("API 응답:", data)                          # 로그 출력
    return {
        'statusCode': 200,
        'body': str(data)                             # 딕셔너리를 문자열로 바꿔서 반환 (body는 문자열이어야 안전)
    }
```

### requests POST — 주석판

```python
import requests
import json                                           # (사실상 안 써도 되지만 보통 같이 import)

def lambda_handler(event, context):
    url = "https://httpbin.org/post"                  # <- POST 보낼 주소
    payload = {"name": "test"}                        # <- 보낼 데이터(딕셔너리). 키/값 시험 지시대로 수정
    response = requests.post(url, json=payload)       # json=payload -> 딕셔너리를 JSON으로 자동 변환 + 헤더 자동 설정
    data = response.json()                            # 응답 JSON -> 파이썬 딕셔너리
    print("API 응답:", data)
    return {
        'statusCode': 200,
        'body': str(data)
    }
```

---

## 🚨 자주 나는 에러 / 체크포인트

- **`No module named 'emoji'`** → 계층(Layer)이 람다에 안 붙어있음. 또는 zip 안 폴더명이 `python`이 아님.
- zip 만들 때 **`python` 폴더 자체를 압축**해야 함 (`python` 폴더 "안의 내용물"만 압축하면 ❌).
- **호환 런타임을 안 맞추면** 계층이 목록에 안 보일 수 있음 (Python 3.13 맞추기).
