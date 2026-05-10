
## 시간 연구소 준비하기

```bash
cd ~/project
mkdir git-config-lab
cd git-config-lab
git init #Initialized empty Git repository in /home/labex/project/git-config-lab/.git/
```

## 타임머신의 현재 설정 확인하기

Git 구성은 타임머신의 제어판과 같으며, 세 가지 수준의 설정이 있습니다
특정 설정을 확인하려면 키 (key) 를 지정하면 됩니다. 예를 들어, 설정된 시간 여행자의 이름을 확인하려면 다음과 같이 입력합니다:

```bash
git config --list
git config user.name
```

## 시간 여행자 신원 설정하기

가장 중요한 설정 중 하나는 시간 여행자의 신원입니다. 타임머신은 이 정보를 사용하여 타임라인의 여러 지점에 여러분의 흔적을 남깁니다. 이는 협력 시간 여행에서 매우 중요한데, 다른 여행자들이 타임라인의 특정 변경 사항을 누가 만들었는지 확인할 수 있게 해주기 때문입니다.
시간 통신 주소를 전역으로 설정하려면 
`--global` 플래그는 타임머신에게 이 시스템에서 수행하는 모든 시간 여행 실험에 이 설정을 적용하도록 지시합니다.

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

git config --global user.name 
git config --global user.email
```

## 타임머신 디스플레이 개선하기

타임머신은 출력 결과에 색상을 사용하여 서로 다른 타임라인의 정보를 빠르게 이해할 수 있도록 도와줍니다. 이는 복잡한 시간 데이터를 조사할 때 특히 유용합니다. 이 기능을 활성화해 봅시다.
auto 값은 터미널로 출력할 때는 색상을 사용하고, 다른 장치나 타임라인으로 데이터를 보낼 때는 일반 텍스트로 전환함을 의미합니다.


```bash
git config --global color.ui auto
git config --global color.ui
```

## 선호하는 시간 로그 에디터 선택하기

타임머신은 시간 저장 지점 (커밋) 을 생성할 때와 같이 메시지를 작성해야 할 때가 많습니다. 이때 텍스트 에디터를 엽니다.

```bash
git config --global core.editor nano
git config --global core.editor
```

## 차원 간 타임라인 동기화하기

차원마다 시간 로그의 끝을 처리하는 방식이 다릅니다. Windows 차원은 캐리지 리턴과 라인 피드 문자 (CRLF) 를 모두 사용하는 반면, Unix 기반 차원 (Linux 및 macOS 등) 은 라인 피드 (LF) 만 사용합니다. 이는 서로 다른 차원 평면에서 협업할 때 시간적 왜곡을 일으킬 수 있습니다.


```bash
git config --global core.autocrlf input
git config --global core.autocrlf
```


## 시간 여행 단축키 생성하기

시간 여행 에일리어스 (별칭) 를 사용하면 자주 사용하는 타임머신 명령어에 대한 단축키를 만들 수 있습니다.
status 명령어에 대한 st 에일리어스를 생성합니다. 이제 git status를 입력하는 대신 간단히 git st라고 입력할 수 있습니다
이 명령어가 하는 일은 시간 여행 기록을 다채롭고 유익하게 보여주는 lg라는 에일리어스를 만드는 것입니다.


```bash
git config --global alias.st status
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)' --abbrev-commit"
git config --global alias.st 
git config --global alias.lg
```

## 실험실 전용 설정

특정 실험에 대해 다른 설정이 필요할 수도 있습니다. 타임머신은 실험 수준에서 설정을 할 수 있게 해주며, 이는 해당 실험에 대해서만 전역 설정을 덮어씁니다.
이번에는 --global 플래그를 사용하지 않았음에 주목하세요. 이는 이 설정이 오직 이 특정 실험에만 적용됨을 의미합니다.

```bash
cd ~/project/git-config-lab
git config user.name "Lab User"
git config user.name

git config --global user.name
```
