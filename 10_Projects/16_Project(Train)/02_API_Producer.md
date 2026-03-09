---
aliases:
  - API Producer
  - 열차 API
  - 공공데이터 API
  - 한국철도공사
  - requests
  - API 키발급
tags:
  - Project
related:
  - "[[00_Seoul Station Real-Time Train Project|train-project]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
  - "[[Python_URL_Parsing]]"
  - "[[Python_Requests_Methods]]"
  - "[[Python_DateTime]]"
  - "[[Python_Classes_Objects]]"
---
# 02_API_Producer — Python 으로 서울역 열차 데이터 가져오기

## 이 단계의 목표

```
공공데이터포털에서 API 키를 발급받고
Python 으로 서울역 열차 데이터를 호출해서
응답(JSON) 을 확인하는 것까지
```

---

---

# ① API 키 발급 — data.go.kr

## 발급 순서

```
1. https://www.data.go.kr 접속

2. 회원가입 / 로그인

3. 검색창에 검색
   "한국철도공사_열차운행정보 조회서비스"

4. 검색 결과에서 해당 API 클릭
   → [활용신청] 버튼 클릭
   → 활용목적 입력 (간단하게 "개인 학습용" 으로 작성)
   → 신청 완료

5. 마이페이지 → 개발계정 → 인증키(일반인증키) 확인
   (즉시 발급, 늦으면 최대 1~2시간)
```

## 발급받은 키 .env 에 저장

```bash
# .env 에 추가
TRAIN_API_KEY=발급받은_인증키_여기에_붙여넣기
```

> `.env` 는 `.gitignore` 에 등록되어 있어서 GitHub 에 올라가지 않는다. API 키를 코드에 직접 넣으면 절대 안 됨.

---

---

# ② API 명세 확인

> [한국철도공사_열차운행정보 조회서비스](https://www.data.go.kr/data/15125762/openapi.do#)

## 제공하는 2가지 API

```
① 여객열차 운행계획 조회   <- 예정 시각 (train_schedule 테이블)
② 여객열차 운행정보 조회   <- 실제 시각 (train_realtime 테이블)
```

## 요청 파라미터

|파라미터|설명|예시|필수|
|---|---|---|:-:|
|`serviceKey`|발급받은 API 인증키|`ABC123...`|✅|
|`pageNo`|페이지 번호|`1`||
|`numOfRows`|한 번에 가져올 행 수|`100`||
|`run_ymd`|운행 날짜 (YYYYMMDD)|`20260308`||
|`dptre_stn_cd`|출발역 코드|`0001` (서울역)||

> 별표(`*`) 가 있는 항목만 필수. 나머지는 안 넣으면 서버 기본값으로 처리. [[Python_Requests_Methods#③ API 문서 읽기 — 필수 vs 선택 파라미터|필수 vs 선택 파라미터]] 참고

## 서울역 코드

```
서울역: 3900023
```

>처음에 `0001` 로 착각했다가 데이터 0건이 나왔음. 아래 코드로 직접 API 응답에서 확인함:

```python
dptre_name = target_train.get('dptre_stn_nm', '모름')
dptre_code = target_train.get('dptre_stn_cd', '코드모름')
print(f"{dptre_name}역의 진짜 코드는 '{dptre_code}' 입니다!")
# 서울역의 진짜 코드는 '3900023' 입니다!
```

## 응답 필드 — 여객열차 운행계획 (train_schedule)

```
run_ymd             운행 날짜
trn_no              열차 번호
dptre_stn_cd        출발역 코드
dptre_stn_nm        출발역 이름
arvl_stn_cd         도착역 코드
arvl_stn_nm         도착역 이름
trn_plan_dptre_dt   계획 출발 시각
trn_plan_arvl_dt    계획 도착 시각
```

## 응답 필드 — 여객열차 운행정보 (train_realtime)

```
run_ymd             운행 날짜
trn_no              열차 번호
trn_run_sn          열차 운행 일련번호
stn_cd              역 코드
stn_nm              역 이름
mrnt_cd             주운행선 코드
mrnt_nm             주운행선 이름
uppln_dn_se_cd      상행/하행 구분 코드
stop_se_cd          정차 구분 코드
stop_se_nm          정차 구분 이름
trn_dptre_dt        열차 출발 일시
trn_arvl_dt         열차 도착 일시
```

## stop_se_nm (정차 구분) 값 정리

|값|의미|trn_dptre_dt|trn_arvl_dt|
|---|---|:-:|:-:|
|**시발**|출발역. 이 역에서 운행 시작|있음|**None**|
|**종착**|종착역. 이 역에서 운행 종료|**None**|있음|
|**여객승하차**|정차역. 승객 승하차|있음|있음|
|**통과**|무정차. 서지 않고 지나감|동일하거나 생략|동일하거나 생략|
|**운전정차**|비영업 정차. 신호대기/승무원 교대|—|—|

> 시발역은 `trn_arvl_dt` 가 `None`, 종착역은 `trn_dptre_dt` 가 `None`. 
> 슬라이싱 전에 반드시 `None` 방어 처리가 필요하다.

---

---

# ③ 환경 설정

## requirements.txt

```
# producer/requirements.txt

requests==2.31.0      # API 호출
python-dotenv==1.0.0  # .env 파일 읽기
kafka-python==2.0.2   # Kafka Producer (STEP 3 에서 사용)
```

```bash
pip install -r producer/requirements.txt
```

---

---

# ④ API 호출 코드 — main.py

## 파일 위치

```
seoul-train-realtime-project/
└── producer/
    ├── main.py
    └── requirements.txt
```

## 설계 구조

```
TrainInfo (클래스)
├── __init__()           API 키 로드 + unquote + Session 생성
├── _request_api()       URL 직접 조립해서 호출
│                        params={} 쓰면 cond[] 대괄호가 %5B 로 인코딩됨 → query_str 로 우회
├── _extract_items()     response → body → items → item 파싱
│                        item 이 dict 이면 [item] 으로 감싸서 항상 list 반환
├── _format_dt()         날짜 문자열 → HH:MM 변환 (@staticmethod)
│                        None / 짧은 값 방어 포함
├── get_train_schedule() 운행계획 전체 페이지네이션 수집
└── get_train_realtime() 특정 열차 실시간 위치 조회
```

## main.py

```python
import requests
import os
import urllib.parse
from dotenv import load_dotenv
from datetime import datetime, timedelta
from requests.exceptions import RequestException, JSONDecodeError

load_dotenv()

class TrainInfo:
    def __init__(self):
        train_api_key = os.getenv("TRAIN_API_KEY")
        if not train_api_key:
            raise ValueError("환경변수 TRAIN_API_KEY 가 설정되지 않았습니다. 확인해주세요.")

        # .env 에서 읽어온 키가 이미 인코딩된 상태일 수 있음
        # unquote 로 먼저 풀어줘야 requests 가 이중 인코딩하지 않음
        # [[Python_URL_Parsing]] unquote 참고
        self.api_key = urllib.parse.unquote(train_api_key)
        self.base_url = "https://apis.data.go.kr/B551457/run/v2"

        # 같은 서버에 반복 호출 -> Session 으로 연결 재사용
        # [[Python_Requests_Methods]] Session 참고
        self.session = requests.Session()

    def _request_api(self, endpoint: str, query_string: str) -> dict:
        # params={} 로 넘기면 cond[run_ymd::EQ] 의 대괄호가 %5B 로 인코딩됨 -> 400 Bad Request
        # URL 을 직접 문자열로 조립해서 요청
        url = f"{self.base_url}/{endpoint}?serviceKey={self.api_key}&returnType=JSON&{query_string}"

        try:
            res = self.session.get(url, timeout=10)
            res.raise_for_status()  # 4xx, 5xx -> HTTPError 발생
            return res.json()

        except JSONDecodeError:
            # API 키 오류 시 XML 반환 / 트래픽 초과 시 HTML 반환
            print(f"[오류] JSON 형식이 아닙니다. API 키 오류이거나 트래픽 초과일 수 있습니다.\n응답 내용: {res.text[:100]}")
            return {}

        except RequestException as e:
            # ConnectionError, HTTPError, Timeout 등 전부 잡음
            print(f"[네트워크 오류] API 요청 중 문제가 발생했습니다: {e}")
            return {}

    def _extract_items(self, data: dict) -> list:
        # response -> body -> items -> item 순서로 파싱
        # [[Python_Dictionaries]] .get() 체이닝 참고
        try:
            items = (
                data
                .get("response", {})
                .get("body", {})
                .get("items", {})
                .get("item", [])
            )
            # 공공데이터 API 함정:
            # 결과 1개  -> dict 하나로 반환
            # 결과 여러 개 -> list 로 반환
            # isinstance 로 확인 후 항상 list 로 통일
            # [[Python_Lists_Tuples]] isinstance 참고
            return items if isinstance(items, list) else [items]

        except AttributeError:
            return []

    @staticmethod
    def _format_dt(date_str: str, default_text: str) -> str:
        # None / 짧은 값 방어 (시발역은 trn_arvl_dt=None, 종착역은 trn_dptre_dt=None)
        if not date_str or not isinstance(date_str, str) or len(date_str) < 16:
            return default_text
        try:
            # 정석: strptime 으로 파싱 후 strftime 으로 포맷
            dt_obj = datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S.%f")
            return dt_obj.strftime("%H:%M")
        except ValueError:
            # 포맷이 다를 경우 슬라이싱으로 fallback
            return date_str[11:16]

    def get_train_schedule(self, run_ymd: str) -> list:
        all_items = []
        page_no = 1
        while True:
            print(f" 운행계획 {page_no}페이지 데이터를 가져오는 중입니다...")
            query_str = f"pageNo={page_no}&numOfRows=100&cond[run_ymd::EQ]={run_ymd}"
            data = self._request_api("travelerTrainRunPlan2", query_str)

            items = self._extract_items(data)
            if not items:
                break
            all_items.extend(items)

            body = data.get("response", {}).get("body", {})
            total_count = body.get("totalCount", 0)
            print(f"🔄 {page_no}페이지 수집 중... (모은 데이터: {len(all_items)}개 / 서버 전체: {total_count}개)")

            if len(all_items) >= total_count:
                break
            # 안전 브레이크: pagNo 오타 등 비정상 상황에서 무한루프 방지
            if page_no >= 10:
                print("🚨 [경고] 데이터가 비정상적으로 많습니다. 무한 루프를 강제 종료합니다.")
                break

            page_no += 1
        return all_items

    def get_train_realtime(self, run_ymd: str, trn_no: str) -> list:
        all_items = []
        page_no = 1
        while True:
            print(f"⚡️ 실시간 정보 {page_no}페이지 데이터를 가져오는 중입니다...")
            query_str = f"pageNo={page_no}&numOfRows=100&cond[run_ymd::EQ]={run_ymd}&cond[trn_no::EQ]={trn_no}"
            data = self._request_api("travelerTrainRunInfo2", query_str)
            items = self._extract_items(data)
            if not items:
                msg = data.get("response", {}).get("header", {}).get("resultMsg", "알수없음")
                print(f"   [서버 응답 메시지] {msg}")
                break

            all_items.extend(items)
            body = data.get("response", {}).get("body", {})
            total_count = int(body.get("totalCount", 0))

            if len(all_items) >= total_count or page_no >= 10:
                break

            page_no += 1

        return all_items


if __name__ == "__main__":
    today = datetime.now().strftime("%Y%m%d")
    print(f"[조회날짜] {today}")

    try:
        train_info = TrainInfo()

        schedule_items = train_info.get_train_schedule(today)

        print("\n=== 실시간 기차 전광판 ===")

        if schedule_items:
            current_time_str = datetime.now().strftime("%H:%M")
            target_train = None

            # 현재 시각 기준으로 아직 도착하지 않은 첫 번째 열차 탐색
            for item in schedule_items:
                raw_arr = item.get("trn_plan_arvl_dt", "")
                arr_time_str = TrainInfo._format_dt(raw_arr, "00:00")

                if current_time_str <= arr_time_str:
                    target_train = item
                    break

            if target_train:
                target_trn_no = target_train.get("trn_no", "00000")
                route = f"{target_train.get('dptre_stn_nm', '모름')}➡️{target_train.get('arvl_stn_nm', '모름')}"

                print(f" 현재 시각({current_time_str}) 기준, 운행 중인 [{route}] {target_trn_no}호 열차를 추적합니다.\n")

                realtime_items = train_info.get_train_realtime(today, target_trn_no)

                if realtime_items:
                    for item in realtime_items[:5]:
                        real_dep = TrainInfo._format_dt(item.get("trn_dptre_dt"), "출발 전")
                        real_arr = TrainInfo._format_dt(item.get("trn_arvl_dt"), "운행중")

                        print(f"[{item.get('trn_no', '000')}호 열차] "
                              f"현재: {item.get('stn_nm', '미확인역')} ({item.get('stop_se_nm', '운행중')}) | "
                              f"출발: {real_dep} | 도착: {real_arr}")
                else:
                    print("해당 열차의 실시간 위치 정보가 업데이트 되지 않았습니다.")
            else:
                print(f"현재시각({current_time_str}), 금일 여객열차 운행이 모두 종료되었습니다. 모두 안녕히 주무세요😴")

    except ValueError as e:
        print(e)
```

## 실행

```bash
cd producer
python main.py
```

## 실행 결과

```
[조회날짜] 20260309

 운행계획 1페이지 데이터를 가져오는 중입니다...
🔄 1페이지 수집 중... (모은 데이터: 100개 / 서버 전체: 350개)
 운행계획 2페이지 데이터를 가져오는 중입니다...
...

=== 실시간 기차 전광판 ===
 현재 시각(14:23) 기준, 운행 중인 [서울➡️부산] 00051호 열차를 추적합니다.

⚡️ 실시간 정보 1페이지 데이터를 가져오는 중입니다...
[00051호 열차] 현재: 서울 (시발) | 출발: 13:00 | 도착: 운행중
[00051호 열차] 현재: 광명 (여객승하차) | 출발: 13:19 | 도착: 13:16
[00051호 열차] 현재: 천안아산 (여객승하차) | 출발: 13:47 | 도착: 13:44
[00051호 열차] 현재: 오송 (여객승하차) | 출발: 14:04 | 도착: 14:01
[00051호 열차] 현재: 대전 (여객승하차) | 출발: 14:19 | 도착: 14:15
```


---

---

# ⑤ 응답 구조 (JSON)

```json
{
  "response": {
    "header": {
      "resultCode": "00",
      "resultMsg": "NORMAL SERVICE"
    },
    "body": {
      "items": {
        "item": [
          {
            "run_ymd": "20260308",
            "trn_no": "KTX001",
            "dptre_stn_cd": "0001",
            "dptre_stn_nm": "서울",
            "arvl_stn_cd": "0020",
            "arvl_stn_nm": "부산",
            "trn_plan_dptre_dt": "2026-03-08 09:00:00.0",
            "trn_plan_arvl_dt": "2026-03-08 11:50:00.0"
          }
        ]
      },
      "totalCount": 50,
      "pageNo": 1,
      "numOfRows": 100
    }
  }
}
```

---

---

# 트러블슈팅 — 실제로 겪은 오류들

## ① NoneType 슬라이싱 에러

```
TypeError: 'NoneType' object is not subscriptable
```

```
원인: 시발역은 trn_arvl_dt=None, 종착역은 trn_dptre_dt=None
      None 에 [11:16] 슬라이싱 시도 -> TypeError

해결: _format_dt() 로 isinstance(str) + len >= 16 방어
```

## ② 무한 로딩 착각 (KeyboardInterrupt)

```
원인: 페이지네이션 while True 가 돌면서 서버에서 데이터를 퍼오는 중
      진행 상황 print 가 없어서 멈춘 줄 알고 Ctrl+C 로 강제 종료

해결: 루프 안에 print(f"🔄 {page_no}페이지 수집 중...") 추가
```

## ③ 78만 개 무한루프 지옥 + pagNo 오타

```
원인: pageNo 를 pagNo 로 오타 -> 서버가 계속 1페이지만 반환
      검색 조건(cond) 도 제대로 안 들어가서 전체 DB 를 다 던져줌
      -> 78만 개, 9만 개씩 무한정 쏟아짐

해결: 오타 수정 + 안전 브레이크 (page_no >= 10) 추가
```

## ④ 400 Bad Request — `cond[]` 인코딩 문제

```
원인: params={} 딕셔너리로 넘기면
      cond[run_ymd::EQ] 의 대괄호 [] 를 requests 가 %5B%5D 로 인코딩
      -> 서버가 알아보지 못하고 400 Bad Request 반환

해결: params={} 대신 URL 을 직접 f-string 으로 조립해서 요청
      url = f"...?serviceKey=...&cond[run_ymd::EQ]={run_ymd}"
```

## ⑤ 운행계획 데이터 0건 — 역 코드의 배신

```
원인: 서울역 코드가 0001 일 것이라 착각
      stn_cd=0001 로 필터를 걸었으나
      코레일 V2 API 에서는 다른 코드 체계 사용

해결: 역 코드 필터 제거
      날짜(run_ymd) 로만 검색해서 전체 데이터 가져옴
      이후 API 응답에서 직접 코드 확인

      dptre_code = target_train.get('dptre_stn_cd', '코드모름')
      print(f"서울역의 진짜 코드는 '{dptre_code}'")
      -> 3900023
```

## ⑥ 실시간 데이터 0건 — 시간의 흐름

```
원인: 특정 열차(00001호) 가 이미 아침에 종착역 도착 후 운행 종료
      또는 심야 시간이라 서버에서 데이터 삭제

해결: 현재 시각 기준으로 아직 도착 전인 열차를 자동으로 탐색
      current_time_str <= arr_time_str 조건으로 대상 열차 선택
```

## ⑦ item 타입 통일 — 공공데이터 API 함정

```
원인: 결과 1개  -> API 가 dict 하나로 반환  {"trn_no": "KTX001"}
      결과 여러 개 -> API 가 list 로 반환    [{"trn_no": "KTX001"}, ...]
      타입이 달라서 for 문 돌리면 TypeError 발생

해결: isinstance(items, list) 로 타입 확인 후 항상 list 로 통일
      return items if isinstance(items, list) else [items]
```

## ⑧ 이중 인코딩 방지 — unquote

```
원인: .env 에서 읽어온 API 키가 이미 % 인코딩된 상태
      requests 가 한 번 더 인코딩 -> %25 로 변환되는 대참사
      -> SERVICE KEY IS NOT REGISTERED ERROR

해결: urllib.parse.unquote(train_api_key) 로 먼저 풀고 넘기기
      이미 디코딩된 키라면 그대로 통과 (안전장치)
```

## ⑨ 자잘한 포맷 오류

```
증상: %H%M (콜론 누락) -> 1423 형태로 출력
      2026/03/09 (슬래시 포함) -> API 파싱/검색 실패

해결: %H:%M 으로 수정
      슬래시 없는 YYYYMMDD 포맷으로 변경 (strftime("%Y%m%d"))
```

---
# totalCount 는 어디서 오나

```json
"body": {
    "numOfRows": 100,
    "pageNo": 1,
    "totalCount": 350,   <- 서버가 가진 전체 데이터 개수
    "items": { ... }
}
```

```python
body = data.get("response", {}).get("body", {})
total_count = body.get("totalCount", 0)

# 페이지네이션 종료 조건
if len(all_items) >= total_count:
    break
# 모은 데이터가 서버 전체 개수 이상이면 더 가져올 게 없음
```

---
---
# 쿼리스트링 파라미터 순서

```python
query_str = f"pageNo={page_no}&numOfRows=100&cond[run_ymd::EQ]={run_ymd}"

url = f"{self.base_url}/{endpoint}?serviceKey={self.api_key}&returnType=JSON&{query_string}"
```

```
파라미터 순서는 상관없다.
HTTP 스펙상 쿼리스트링 파라미터는 순서가 없음.
서버는 이름(key)으로 찾기 때문에 순서가 달라도 동일하게 처리됨.

다만 관례상:
  serviceKey, returnType 을 앞에 두는 것은
  "인증 + 응답 포맷" 이 가장 기본 설정이라서
  가독성 때문에 앞에 두는 것일 뿐.

cond[] 파라미터들도 순서 무관하게 AND 조건으로 처리됨.
```


---

✅ 완료되면 → [[03_Kafka_Producer]] 로 이동