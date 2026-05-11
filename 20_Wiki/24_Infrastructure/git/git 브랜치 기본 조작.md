## 멀티버스 허브 설정하기

먼저, 모든 평행 현실을 관리할 중앙 허브를 만들어야 합니다. 이것이 우리의 메인 Git 저장소가 될 것입니다
-m 플래그를 사용하면 커밋 메시지를 직접 입력할 수 있습니다. 항상 명확하고 설명적인 커밋 메시지를 작성하도록 노력하세요!
git commit은 스테이징된 변경 사항으로 새로운 커밋 (저장 지점) 을 생성합니다

```bash
mkdir git-branch-lab 
cd git-branch-lab
git init

echo "# Git Branch Lab" > README.md 
echo "This is my hub for multiversal Git branch experiments" >> README.md

git add README.md
git commit -m "Initial commit: Create the multiverse hub"
```

## 첫 번째 평행 현실 만들기

feature-dimension이라는 이름의 새 브랜치를 생성합니다
* `*`표시는 현재 여러분이 어느 현실에 있는지를 나타냅니다.
* 팁: 브랜치 목록에서 나가서 터미널로 돌아가려면 q를 누르세요.
* git branch를 실행하면 다음과 같이 보일 것입니다:
* git switch feature-dimension 명령어를 사용해도 동일한 결과를 얻을 수 있습니다. git switch는 Git 2.23 버전에서 도입된 최신 명령어로, 브랜치 전환을 위해 특별히 설계되어 더 명확하고 직관적입니다. 두 명령어 모두 결과는 같지만, 명확성을 위해 일반적으로 git switch 사용이 권장됩니다.
* 팁: 최신 버전의 Git 에서는 git checkout -b feature-dimension 또는 git switch -c feature-dimension 명령어를 사용하여 브랜치 생성과 이동을 한 번에 할 수 있습니다. 한 번의 동작으로 포털을 만들고 바로 뛰어드는 것과 같죠! git checkout의 -b 옵션이나 git switch의 -c 옵션은 브랜치 생성과 전환을 하나의 단계로 결합합니다.

```bash
git branch feature-dimension
git branch
# feature-dimension * master
git checkout feature-dimension
# * feature-dimension master
```

## 새로운 현실 형성하기

현재 브랜치와 마스터 브랜치의 차이점을 보고 싶다면 git diff master 명령어를 사용할 수 있습니다
이 변화들은 현재 feature-dimension 브랜치에만 존재합니다. 만약 master 브랜치로 다시 돌아간다면, 그곳에서는 이 변화들을 볼 수 없을 것입니다.

```bash
echo "This is a powerful artifact from another dimension" > dimensional-artifact.txt

git add dimensional-artifact.txt 
git commit -m "Create a powerful interdimensional artifact"

echo "#### Feature Dimension" >> README.md 
echo "We've discovered a powerful artifact in this reality" >> README.md

git add README.md 
git commit -m "Document the discovery of the artifact"
```

## 현실 병합하기

이제 평행 현실에서 놀라운 발견을 마쳤으니, 이를 메인 우주로 가져올 시간입니다. 이 과정을 병합 (Merging) 이라고 합니다
이 명령어는 Git 에게 feature-dimension의 모든 변경 사항을 가져와 master에 적용하도록 지시합니다. 평행 현실에서의 발견을 메인 우주로 통합하는 것과 같습니다.
현재 브랜치에 병합된 모든 브랜치 목록을 보고 싶다면 git branch --merged 명령어를 사용할 수 있습니다. 어떤 브랜치들이 통합되었는지 추적하는 데 유용합니다.

```bash
git switch master / git checkout master

git merge feature-dimension
# Updating <hash1>..<hash2> Fast-forward README.md | 2 ++ dimensional-artifact.txt | 1 + 2 files changed, 3 insertions(+) create mode 100644 dimensional-artifact.txt

cat dimensional-artifact.txt 
cat README.md
```

## 차원 포털 닫기

발견한 것들을 메인 현실로 성공적으로 가져왔으므로, 이제 평행 차원으로 통하는 포털을 닫을 수 있습니다. Git 용어로는 더 이상 필요하지 않은 브랜치를 삭제하는 것입니다.
-d 플래그는 브랜치를 삭제하도록 Git 에 지시하지만, 해당 브랜치가 완전히 병합된 경우에만 삭제합니다. 이는 병합되지 않은 변경 사항을 실수로 잃어버리지 않도록 하는 안전장치입니다. Git 은 현재 브랜치에 변경 사항이 병합된 경우에만 -d를 통한 삭제를 허용합니다.
만약 완전히 병합되지 않은 브랜치를 삭제하려고 하면 Git 이 경고를 보낼 것입니다. 그런 경우에도 정말로 삭제하고 싶다면 대문자 -D 플래그를 사용하여 강제로 삭제할 수 있습니다: git branch -D feature-dimension. 이 명령어는 병합 상태와 관계없이 브랜치를 강제로 삭제합니다. 이 강력한 기능은 주의해서 사용해야 합니다! 브랜치의 변경 사항이 정말로 필요 없다는 확신이 들 때만 사용하세요
원격 브랜치를 포함한 모든 브랜치를 보고 싶다면 git branch -a를 사용할 수 있습니다

```bash
git branch -d feature-dimension
git branch
```