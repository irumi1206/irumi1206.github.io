---
title: "Git"
hideSummary: true
---

### Git

Git은 프로젝트의 변경 이력을 commit 단위로 관리하는 Version Control System이다. Git을 사용하면 특정 시점의 프로젝트 상태를 기록하고, 하나의 프로젝트에서 여러 작업 흐름을 분리해서 진행한 뒤 다시 합칠 수 있다. GitHub은 Git과는 별개의 서비스로, Git Repository를 원격 서버에 저장하고 여러 사용자가 이를 공유하고 협업할 수 있도록 한다. GitHub과 같이 원격 서버에 존재하는 저장소를 Remote Repository, 사용자의 컴퓨터에서 직접 작업하는 저장소를 Local Repository라고 한다.

### Local Repository와 Commit

Local Repository에서 실제 파일을 수정하고 있는 공간을 Working Directory라고 한다. 파일을 수정했다고 해서 해당 변경사항이 바로 Git의 새로운 버전으로 기록되는 것은 아니다. 먼저 commit에 포함할 변경사항을 Staging Area에 올리고, 이후 commit을 통해 해당 상태를 하나의 변경 이력으로 확정하는 과정을 거친다. 따라서 기본적인 Git의 변경 흐름은 다음과 같다.

Working Directory → Staging Area → Local Repository

이를 실제로 수행해보면 다음과 같다.

```bash
git clone <repository-url>        # Remote Repository를 Local Repository로 clone
cd <repository-name>

echo "first change" >> notes.txt  # Working Directory의 파일 수정

git status                        # 현재 Repository의 상태 확인
git diff                          # Working Directory와 Staging Area 사이의 변경사항 확인

git add notes.txt                 # notes.txt를 Staging Area에 추가
                                  # 모든 변경 파일은 'git add .'으로 추가 가능

git status
git diff --staged                 # 마지막 commit과 Staging Area 사이의 변경사항 확인

git commit -m "Add first note"    # Staging Area의 상태를 새로운 commit으로 기록
```

`git status`를 통해 현재 어떤 파일이 수정되었고 어떤 변경사항이 staging되어 있는지 확인할 수 있으며, `git diff`와 `git diff --staged`를 이용하면 각 단계 사이에 어떤 변경사항이 존재하는지 확인할 수 있다. Commit은 특정 시점의 프로젝트 상태를 기록한 단위이며, 새로운 commit을 만들면 이전 commit과 연결되면서 하나의 변경 이력이 형성된다.

```text
A → B → C
```

여기서 C는 B 이후에 만들어진 commit이고, B는 A 이후에 만들어진 commit이다. 따라서 여러 번 commit을 수행하면 각 시점의 프로젝트 상태와 변경 이력을 순차적으로 관리할 수 있다.

### Branch

지금까지는 하나의 변경 흐름만을 기준으로 commit을 만들어 왔다. 하지만 기존 작업을 유지한 상태에서 서로 다른 기능이나 버전을 동시에 개발하고 싶을 경우 Branch를 사용할 수 있다. Branch는 하나의 작업 흐름을 의미하며, Git 내부에서는 해당 흐름의 가장 마지막 commit을 가리키는 reference로 관리된다. Git Repository를 처음 만들면 일반적으로 기본 branch로 `main`이 사용된다. `main`은 Git에서 특별한 의미를 가지는 예약어가 아니라 기본 작업 흐름에 관례적으로 사용하는 branch 이름이며, Repository 설정에 따라 다른 이름을 사용할 수도 있다.

현재 `main`에서 새로운 commit을 만들면 새로운 branch가 만들어지는 것이 아니라 기존 `main`이 새로운 commit을 가리키도록 이동한다.

```bash
echo "main work" >> notes.txt
git add notes.txt
git commit -m "Update main work"  # 현재 main에서 새로운 commit C 생성
```

```text
A → B → C
        ↑
       main
```

이 상태에서 새로운 branch를 만들면 처음에는 `main`과 새로운 branch가 같은 commit C를 가리킨다.

```bash
git switch -c feature-login      # 현재 C를 기준으로 feature-login 생성 후 이동
git branch                       # Local branch 목록과 현재 branch 확인
```

```text
A → B → C
        ↑
       main
       feature-login
```

`feature-login`에서 새로운 commit D를 만들면 `feature-login`만 D를 가리키도록 이동하고 `main`은 C에 남아 있다.

```bash
echo "login work" > login.txt
git add login.txt
git commit -m "Add login work"
```

```text
A → B → C → D ← feature-login
        ↑
       main
```

여기서 다시 `main`으로 돌아간 뒤 C를 기준으로 또 다른 branch를 만들 수도 있다. 즉 하나의 commit을 공통 시작점으로 여러 작업 흐름이 분화될 수 있다.

```bash
git switch main                  # 다시 C를 가리키는 main으로 이동
git switch -c feature-cache      # 같은 C에서 두 번째 branch 생성

echo "cache work" > cache.txt
git add cache.txt
git commit -m "Add cache work"
```

```text
          D ← feature-login
         /
A → B → C ← main
         \
          E ← feature-cache
```

실제 graph 형태로 확인할 수 있다.

```bash
git log --oneline --decorate --graph --all
```

### Remote Repository와 origin

Local Repository에서 생성한 commit은 기본적으로 자신의 컴퓨터에만 존재한다. 이를 GitHub과 같은 Remote Repository와 공유하기 위해서는 Local과 Remote 사이의 상태를 주고받아야 한다. `git clone`으로 Repository를 받아오면 해당 Remote Repository에는 일반적으로 `origin`이라는 이름이 자동으로 붙는다.

```bash
git remote                       # 등록되어 있는 Remote 이름 확인
git remote -v                    # Remote의 fetch / push 주소 확인
```

Local의 `main`과 GitHub의 `main`은 서로 다른 Repository에 존재하며, Git은 마지막으로 확인한 Remote branch의 상태를 `origin/main` branch로 기록한다.
```bash
echo "local change" >> notes.txt
git add notes.txt
git commit -m "Add local change"
```

```text
A → B → C ← main
    ↑
origin/main
```

여기서 `origin/main`은 GitHub의 실제 `main`을 실시간으로 직접 가리키는 것이 아니라, Local Repository가 마지막으로 확인한 Remote `main`의 상태를 나타낸다.

### Push, Fetch, Pull

Local에서 만든 commit을 Remote Repository에 반영하기 위해서는 `push`를 사용한다. Local `main`의 C를 Remote의 `main`에 push하면 Remote도 동일한 commit까지 이동한다.

```bash
git push origin main             # Local main의 commit을 origin의 main으로 전송
```

처음 Local branch와 Remote branch의 tracking 관계를 지정할 때는 `-u` 옵션을 사용할 수 있다.

```bash
git push -u origin main          # main이 origin/main을 upstream으로 사용하도록 설정
git push                         # 이후에는 Remote와 branch를 생략해서 push 가능
```

반대로 Remote Repository에 다른 사용자가 새로운 commit을 추가한 경우 해당 변경사항을 Local로 가져오기 위해 `fetch`를 사용할 수 있다.

```bash
git fetch origin                 # Remote의 최신 commit과 branch 정보를 Local로 가져옴
```

예를 들어 Local `main`은 B를 가리키고 있고 Remote `main`에는 C가 추가되어 있다고 하면, fetch 이후 C라는 commit과 Remote의 최신 상태는 Local로 받아오지만 현재 작업 중인 `main` 자체는 이동하지 않는다.

```text
A → B → C ← origin/main
    ↑
   main
```

즉 `fetch`는 Remote의 최신 상태를 Local Repository로 가져오는 과정이고, 이를 현재 Local branch에 반영하는 과정은 별도로 필요하다. 이 두 과정을 한 번에 수행하기 위해 `pull`을 사용할 수 있다.

```bash
git pull                         # Remote 변경사항을 가져온 뒤 현재 branch에 통합
```

기본적인 개념에서는 다음과 같이 이해할 수 있다.

```text
git pull ≈ git fetch + git merge
```

다만 Git 설정에 따라 `pull`에서 merge 대신 rebase를 사용하도록 설정할 수도 있다.

### Merge와 Conflict

서로 다른 Branch에서 진행한 작업을 하나의 흐름으로 다시 합치기 위해 `merge`를 사용할 수 있다. 예를 들어 C에서 `feature` branch와 `main` branch가 각각 나뉘어 D와 E라는 commit을 만들었다고 하면 다음과 같다.

```text
          D ← feature
         /
A → B → C
         \
          E ← main
```

`feature`의 변경사항을 `main`에 합치려면 `main`으로 이동한 뒤 `merge`를 수행한다.

```bash
git switch main                  # 변경사항을 받을 branch로 이동
git merge feature                # feature의 변경 이력을 main에 통합
```

두 branch가 서로 다른 commit을 가지고 있다면 Git은 두 흐름을 합치며 필요한 경우 Merge Commit을 생성한다.

```text
          D ─────┐
         /       │
A → B → C        F ← main
         \       │
          E ─────┘
```

Merge 과정에서 같은 파일을 수정했다고 해서 항상 문제가 발생하는 것은 아니다. 서로 다른 부분을 수정했다면 Git이 자동으로 두 변경사항을 함께 적용할 수 있다. 하지만 동일한 부분을 서로 다르게 수정하여 어느 내용을 선택해야 하는지 판단할 수 없는 경우 Conflict가 발생한다. 예를 들어 같은 파일에서 `color=blue`라는 한 줄을 `main`에서는 `color=green`, `feature`에서는 `color=red`로 수정하고 merge하면 다음과 같은 Conflict marker가 파일에 나타날 수 있다.

```text
<<<<<<< HEAD
color=green
=======
color=red
>>>>>>> feature
```

이 경우 사용자가 직접 원하는 최종 상태로 파일을 수정한 뒤 해당 파일을 다시 staging하고 commit하여 Conflict 해결을 완료한다.

```bash
git status                       # Conflict가 발생한 파일 확인

# Conflict marker를 직접 수정한 뒤
git add <conflicted-file>        # 해결한 파일을 Staging Area에 추가
git commit                       # Conflict 해결 결과를 commit
```

### Rebase

Rebase 역시 서로 다른 Branch의 변경사항을 통합하기 위한 방법이지만, Merge와는 commit history를 만드는 방식이 다르다. 다음과 같이 C에서 `feature`와 `main`이 나뉘어 각각 D와 E를 만들었다고 하자.

```text
          D ← feature
         /
A → B → C
         \
          E ← main
```

이 상태에서 `feature` branch를 `main`을 기준으로 rebase하면 `feature`에서 C 이후에 수행했던 변경사항을 `main`의 최신 commit E 이후에 다시 적용한다.

```bash
git switch feature              # rebase할 branch로 이동
git rebase main                 # feature의 변경사항을 main의 최신 commit 이후에 다시 적용
```

결과적으로 기존 D commit 자체가 그대로 이동하는 것이 아니라, D와 같은 변경사항을 가지는 새로운 commit D'가 E 이후에 생성된다.

```text
A → B → C → E → D'
                ↑
             feature
```

따라서 D와 D'는 같은 변경사항을 포함할 수 있지만 서로 다른 commit이기 때문에 commit hash도 달라진다. Merge는 기존에 branch가 나뉘었던 구조를 유지한 채 두 흐름을 합치는 반면, Rebase는 한 branch의 commit들을 다른 branch의 최신 commit 이후에 다시 적용하여 history를 선형적으로 정리한다.

```text
Merge

          D ─────┐
         /       │
A → B → C        F
         \       │
          E ─────┘


Rebase

A → B → C → E → D'
```

따라서 Merge는 실제 작업이 분기되고 다시 합쳐진 이력을 그대로 남기고 싶을 때 사용할 수 있고, Rebase는 작업 내용을 유지하면서 commit history를 보다 단순한 형태로 정리하고 싶을 때 사용할 수 있다. 다만 Rebase는 기존 commit을 기반으로 새로운 commit을 생성하여 history를 변경하기 때문에, 이미 Remote에 push되어 여러 사용자가 공유하고 있는 branch를 임의로 rebase하는 것은 주의해야 한다.
