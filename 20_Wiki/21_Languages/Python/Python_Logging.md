---
aliases:
  - 로깅
  - 로그관리
  - print대신 logging
tags:
  - Python
related:
  - "[[Python_Builtin_Functions]]"
  - "[[00_Python_HomePage]]"
---
## Concept Summary

**"프로그램의 실행 기록(Log)을 체계적으로 남기는 도구."**
`print()`가 "야! 여기 봐!"라고 소리치는 거라면, `logging`은 "2026-02-12 15:00:00 [정보] 데이터 전송 완료"라고 **일지**를 쓰는 것입니다.

---
## 왜 필요한가? (Why: print vs logging)

**문제점 (`print`의 한계):**

1. **정보 부족:** 그냥 "Error"라고만 찍히면, _언제_ 터졌는지 알 수가 없음.
2. **제어 불가:** 코드 완성 후 `print`를 일일이 지우러 다녀야 함.
3. **저장 불가:** 터미널을 끄면 기록이 다 날아감.

**해결책 (`logging`의 장점):**

1. **자동 문맥:** 시간(`asctime`), 에러 등급(`levelname`)을 알아서 붙여줌.
2. **레벨 조절:** "중요한 에러만 보여줘!"라고 설정하면, 자잘한 정보는 자동으로 숨겨줌. (코드 수정 없이!)
3. **영구 저장:** 화면 출력뿐만 아니라 **파일**로도 동시에 저장 가능.

---
## Log Levels (로그의 계급) 

로그에는 **중요도(Level)**가 있습니다. 아래로 갈수록 심각한 상황입니다.

1. **DEBUG:** "변수 A에 10이 들어갔음" (개발할 때만 봄, 시시콜콜한 이야기)
2. **INFO:** "서버 시작함", "데이터 1건 전송함" (정상 작동 확인용) **← Producer 코드는 여기!**
3. **WARNING:** "데이터가 좀 이상한데 일단 넘어감" (경고)
4. **ERROR:** "연결 실패! 데이터 못 보냄" (에러 발생)
5. **CRITICAL:** "시스템 폭발 직전! 셧다운!" (치명적)

> **설정 원리:** 레벨을 `INFO`로 설정하면, 그보다 낮은 `DEBUG`는 무시되고, `INFO` 이상(`WARNING`, `ERROR`...)만 기록됩니다.


---
## Code Analysis

```python
logging.basicConfig(
    level=logging.INFO,   # 1. INFO 등급 이상만 기록하겠다! (DEBUG는 무시)
    stream=sys.stdout,    # 2. 파일 말고 '표준 출력(터미널)'에 쏴라! (Docker가 이걸 좋아함)
    format='%(asctime)s - %(levelname)s - %(message)s' # 3. 이 모양대로 찍어라!
)
```

- `%(asctime)s`: **언제?** (2026-02-12 15:30:01)
- `%(levelname)s`: **등급은?** (INFO)
- `%(message)s`: **할 말은?** (Sent: ...)

---
## Practical Context (실전)

**"왜 도커(Docker) 쓸 때는 logging을 써야 하나요?"**

도커 컨테이너는 백그라운드에서 돌기 때문에 우리가 화면을 계속 쳐다볼 수 없습니다. 
그래서 `logging`을 통해 **표준 출력(stdout)** 으로 기록을 남겨두면:

1. `docker logs iot-producer` 명령어로 언제든 과거 기록을 조회할 수 있습니다.
2. Kibana, Datadog 같은 모니터링 툴이 이 로그를 수집해서 예쁜 그래프를 그려줍니다.
3. `print`를 쓰면 버퍼링(Buffering) 때문에 로그가 늦게 뜨거나 씹히는 경우가 있는데, `logging`은 설정을 통해 이걸 방지할 수 있습니다.


---
## Common Beginner Misconceptions (초보자 실수)

### ① "logging.debug('안녕') 했는데 화면에 안 나와요!"

- **이유:** 기본 설정(`level`)이 보통 `WARNING` 이상으로 되어 있기 때문입니다.
- **해결:** `logging.basicConfig(level=logging.DEBUG)` 라고 설정해줘야 시시콜콜한 이야기까지 다 들려줍니다.

### ② "파일로 저장하려면요?"

- 설정에서 `stream=sys.stdout` 대신 **`filename='app.log'`** 를 쓰면 화면엔 안 나오고 파일에만 조용히 기록됩니다. (서버 개발할 때 필수!)