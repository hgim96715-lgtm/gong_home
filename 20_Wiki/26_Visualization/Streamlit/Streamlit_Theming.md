---
aliases:
  - 색상 정의
  - theme
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_QuickStart]]"
  - "[[Streamlit_Config]]"
---
##  테마의 뼈대 세우기 (`base` 옵션)

모든 색상을 하나하나 지정하기는 너무 귀찮습니다. 그래서 Streamlit은 **'기본 도화지'** 를 먼저 고르게 해줍니다.

**`[theme]` 섹션의 가장 첫 줄에는 반드시 `base`를 적어주는 것이 국룰입니다.**
이걸 적지 않으면 사용자의 컴퓨터 설정(OS 다크모드/라이트모드)에 따라 내 의도와 다르게 화면이 제멋대로 바뀔 수 있습니다.

- **`base="light"`** : 하얀 도화지 (깔끔하고 정갈한 사내 대시보드 느낌)
- **`base="dark"`** : 까만 도화지 (개발자/엔지니어의 전문성이 돋보이는 힙한 느낌)

---
### 내 입맛대로 색상 덧칠하기 (Core Variables)

기본 도화지를 깔았다면, 이제 내 브랜드 컬러(포인트 색상)를 입힐 차례입니다. 
`.streamlit/config.toml` 파일에 아래 변수들을 추가합니다.

|**변수명**|**어디에 칠해질까? (역할)**|**실무 디자인 꿀팁 💡**|
|---|---|---|
|**`primaryColor`**|버튼, 체크박스, 슬라이더, 강조 텍스트|내 포트폴리오의 **메인 포인트 컬러**(예: 형광 초록, 핫핑크)를 씁니다.|
|**`backgroundColor`**|메인 콘텐츠 영역의 전체 배경색|`base`와 톤을 맞춰 눈이 편안한 색(완전 흰색/검은색 보다는 살짝 탁한 색)을 고릅니다.|
|**`secondaryBackgroundColor`**|사이드바(Sidebar) 및 위젯 배경색|메인 배경색과 **살짝 대비되는 색**을 써서 영역을 확실히 분리해 줍니다.|
|**`textColor`**|기본 글자색|배경이 어두우면 밝게, 배경이 밝으면 어둡게 (가독성이 생명!)|
|**`font`**|앱 전체 글꼴 스타일|`sans serif`(깔끔), `serif`(진지함), `monospace`(코딩/터미널 느낌) 중 택 1|


----
### 노가다 금지! "UI 에디터"로 1분 만에 테마 만들기

RGB 코드를 일일이 찾아가며 코드를 수정할 필요가 없습니다. Streamlit은 **실시간 테마 만들기 화면**을 제공합니다.

- 내 Streamlit 앱 화면을 띄운다. (`streamlit run app.py`)
- 우측 상단 햄버거 메뉴( **⋮** ) 클릭 ➡️ **Settings** ➡️ **Theme** 메뉴로 이동.
- **Edit active theme**를 눌러 팔레트에서 색상을 이리저리 바꿔본다. (실시간으로 내 앱 색깔이 바뀜!)
- 맘에 드는 조합을 찾았다면? 아래쪽의 **Copy color to clipboard**를 클릭!
- 내 코드의 `config.toml` 파일에 그대로 `Ctrl+V` 하면 끝!

---
## 실전 꾸미기

```TOML
[theme]
base='dark'
primaryColor='#00C2FF'
backgroundColor='#0E1117'
secondaryBackgroundColor='#161B22'
textColor='#E6EDF3'
font='monospace'
```

