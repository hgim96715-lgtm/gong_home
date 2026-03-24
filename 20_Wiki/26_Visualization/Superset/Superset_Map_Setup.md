---
aliases:
  - Superset 지도
  - Mapbox
  - Deck.gl
  - superset_config.py
tags:
  - Superset
related:
  - "[[00_Superset_HomePage]]"
  - "[[Superset_Chart_Types]]"
---

# Superset_Map_Setup — 지도 차트 설정

## 한 줄 요약

```
Superset 지도 차트 (Deck.gl Scatter 등) 는
Mapbox API 키 + superset_config.py 설정이 필요
일반 Bar/Pie 차트와 달리 추가 설정 없이는 지도 안 나옴
```

---

---

# ① 필요한 것

```
1. Mapbox API 키 발급
   https://account.mapbox.com → Access tokens → Create token

2. superset_config.py 파일 생성

3. docker-compose.yml 환경변수 + 볼륨 설정

4. .env 파일에 키 추가
```

---

---

# ② superset_config.py 생성

```
프로젝트 루트에 superset_config.py 파일 생성
Docker 컨테이너 안 /app/superset/pythonpath/ 에 마운트
```

```python
# superset_config.py
import os

# Mapbox 설정 (지도 표시용)
MAPBOX_API_KEY = os.getenv("MAPBOX_API_KEY")

# 기본 설정
ROW_LIMIT = 5000
SUPERSET_WEBSERVER_TIMEOUT = 300

# 자바스크립트 컨트롤 기능 활성화
FEATURE_FLAGS = {
    "ENABLE_JAVASCRIPT_CONTROLS": True
}

# 로컬 개발용 — 엄격한 웹 보안 정책(CSP) 임시 비활성화
TALISMAN_ENABLED = False
```

```
MAPBOX_API_KEY:
  지도 타일(배경 지도 이미지) 를 Mapbox 서버에서 받아옴
  키 없으면 지도 배경이 안 나오거나 에러 발생

FEATURE_FLAGS ENABLE_JAVASCRIPT_CONTROLS:
  Deck.gl 지도 차트 등 고급 기능 활성화

TALISMAN_ENABLED = False:
  Content Security Policy 비활성화
  로컬 개발 환경에서 지도 로드 차단 방지
  운영 환경에서는 True 권장
```

---

---

# ③ docker-compose.yml 설정

```yaml
superset:
  image: apache/superset:3.1.0
  container_name: hospital-superset
  env_file:
    - .env
  ports:
    - "8089:8088"
  environment:
    - SUPERSET_SECRET_KEY=${SUPERSET_SECRET_KEY}
    - MAPBOX_API_KEY=${MAPBOX_API_KEY}         # ← Mapbox 키 주입
    - SUPERSET_FEATURE_ENABLE_JAVASCRIPT_CONTROLS=true
  volumes:
    - superset_data:/app/superset_home
    - ./superset/superset_config.py:/app/superset/pythonpath/superset_config.py # ← superset 폴더 안
  networks:
    - hospital-network
  depends_on:
    - postgres
```

```text
폴더 구조:
  hospital-project/
  ├── superset/
  │   └── superset_config.py    ← 여기
  ├── docker-compose.yml
  └── .env

볼륨 마운트 경로:
  ./superset_config.py           → 프로젝트 루트에 있을 때
  ./superset/superset_config.py  → superset 폴더 안에 있을 때 ✅
```

```
⚠️ volumes 에 config 파일 마운트 방식:
  ./superset/superset_config.py:/app/superset/pythonpath/superset_config.py # ← superset 폴더 안
  → 로컬 파일을 컨테이너 안 pythonpath 에 연결
  → Superset 시작 시 자동으로 읽어들임
```

---

---

# ④ .env 파일 설정

```bash
# .env
SUPERSET_SECRET_KEY=your_secret_key_here
MAPBOX_API_KEY=pk.eyJ1IjoiX...   # Mapbox 에서 발급받은 키
```

```bash
# .gitignore 에 반드시 추가
.env
```

---

---

# ⑤ 적용 순서

```bash
# 1. superset_config.py 생성 (프로젝트 루트)
# 2. .env 에 MAPBOX_API_KEY 추가
# 3. 컨테이너 재시작
docker compose down
docker compose up -d

# 4. Superset UI 접속
http://localhost:8089
```

---

---

# ⑥ 지도 차트 만들기

```
Charts → + Chart
Dataset: 
Chart type: 

설정:
  Longitude:  위도 
  Latitude:   경도 
  Color by:   
  Point size: 50~100
  Viewport:   서울 중심 (위도 37.5 / 경도 127.0 / Zoom 10)
```

```
Deck.gl Scatterplot 외 지도 차트 종류:
  Deck.gl Arc          두 지점 사이 호(Arc) 연결
  Deck.gl Polygon      지역 경계선 채우기
  Deck.gl Geojson      GeoJSON 데이터로 지도
  Country Map          국가/지역 단위 색상 지도
```


---
---

# ⑦ 툴팁 커스터마이징 — extraProps ⭐️

```
Deck.gl 차트에서 포인트 호버 시 나타나는 툴팁 커스터마이징
Charts → Deck.gl Scatterplot → Customize 탭 → Tooltip
FEATURE_FLAGS ENABLE_JAVASCRIPT_CONTROLS = True 필요
```

## extraProps 구조

```
d.object                        → 현재 호버한 포인트 객체
d.object.extraProps             → 포인트에 붙어있는 실제 데이터
d.object.extraProps.컬럼명      → Dataset 컬럼 값에 접근

Dataset 에 있는 컬럼 전부 접근 가능:
  d.object.extraProps.hpname    → 병원명
  d.object.extraProps.hvec      → 병상 수
  d.object.extraProps.region    → 지역
  d.object.extraProps.duty_tel  → 전화번호
  d.object.extraProps.status    → 커스텀 상태값
```

## 기본 툴팁 코드 예제 

```javascript
d => `
  <div style="padding: 5px;">
    <strong style="font-size: 14px;">
      ${d.object.extraProps.hpname}
    </strong><br/>
    <hr style="margin: 5px 0; border-color: #555;"/>
    <span style="color: ${d.object.extraProps.status === '포화' ? '#FF4B4B' : '#4ADE80'};
                 font-weight: bold;">
      상태: ${d.object.extraProps.status}
    </span>
  </div>
`
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|지도 배경 안 나옴|Mapbox API 키 없음|.env MAPBOX_API_KEY 확인|
|config 적용 안 됨|볼륨 마운트 경로 오류|docker-compose.yml volumes 경로 확인|
|CSP 에러로 지도 차단|TALISMAN_ENABLED = True|superset_config.py TALISMAN_ENABLED = False|
|위도/경도 컬럼 안 보임|Dataset 컬럼 타입 미설정|Dataset → 컬럼 타입 Float 으로 변경|
|컨테이너 재시작 후도 안 됨|config 파일 수정 후 재시작 필요|docker compose restart superset|