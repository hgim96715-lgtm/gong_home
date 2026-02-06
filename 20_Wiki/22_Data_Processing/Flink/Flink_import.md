---
aliases:
  - import
---
```python
# 1. 타입 정보 (데이터가 문자열인지, 숫자인지 Flink에게 알려줌)
from pyflink.common import Types

# 2. 실행 환경 (무대 설정 - 스트림 처리의 시작과 끝)
from pyflink.datastream import StreamExecutionEnvironment

# 3. (옵션) 운영체제/로깅 관련 (경로 찾고 로그 찍을 때 필요)
import os
import sys
```