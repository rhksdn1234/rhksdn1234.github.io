+++
title = "Rocky Linux 가상서버에서 GitHub 연결하기 (설치부터 push까지)"
date = 2026-06-08
draft = false
tags = ["Git", "GitHub", "Rocky Linux", "Linux"]
categories = ["DevOps"]
+++

## 들어가며

Rocky Linux 가상서버에서 작업한 내용을 GitHub로 올리려면 단순히 명령어를 외우는 것보다, 각 단계가 *왜* 필요한지를 이해하는 편이 훨씬 오래 기억에 남는다. 이 글에서는 Git 설치부터 첫 push까지의 전 과정을, 중간에 막히기 쉬운 인증 문제까지 포함해 정리한다.

전체 흐름은 다음과 같다.

1. Git 설치
2. 사용자 정보 설정
3. 저장소 연결 (clone 또는 remote 등록)
4. GitHub 인증 (Personal Access Token)
5. commit & push

---

## 1. Git 설치

Rocky Linux는 RHEL 계열이라 패키지 관리자로 `dnf`를 사용한다. 다음 명령으로 Git을 설치한다.

​```bash
sudo dnf install git -y
​```

설치가 끝나면 버전을 확인해 정상 설치 여부를 점검한다.

​```bash
git --version
​```

`git version 2.x.x` 형태로 출력되면 정상이다.

---

## 2. 사용자 정보 설정

Git은 커밋을 기록할 때 "누가 작업했는지"를 함께 저장한다. 그래서 처음 한 번은 이름과 이메일을 등록해야 한다. 이 정보는 GitHub 로그인 계정과 별개로, 커밋 로그에 남는 작성자 표시일 뿐이다.

​```bash
git config --global user.name "rhksdn1234"
git config --global user.email "your-email@example.com"
​```

`--global` 옵션은 이 서버 전체에서 기본값으로 쓰겠다는 의미다. 저장소마다 다르게 쓰고 싶다면 옵션 없이 해당 폴더에서 설정하면 된다.

---

## 3. 저장소 연결

작업 방식에 따라 두 가지 경우가 있다.

### (A) 이미 GitHub에 저장소가 있는 경우 — clone

GitHub에 올라가 있는 저장소를 그대로 내려받는다. 이러면 원격 연결까지 자동으로 잡힌다.

​```bash
git clone https://github.com/rhksdn1234/저장소이름.git
cd 저장소이름
​```

### (B) 로컬에서 새로 시작하는 경우 — init + remote

서버에서 먼저 작업을 시작했다면, 폴더를 Git 저장소로 초기화한 뒤 원격 주소를 직접 등록한다.

​```bash
git init
git remote add origin https://github.com/rhksdn1234/저장소이름.git
​```

`origin`은 원격 저장소에 붙이는 기본 별칭이다. 앞으로 `git push origin main`처럼 쓸 때 이 이름을 가리킨다.

---

## 4. GitHub 인증 — 여기서 가장 많이 막힌다

과거에는 push할 때 GitHub 아이디와 비밀번호를 넣으면 됐지만, **2021년 8월부터 계정 비밀번호 인증이 폐지되었다.** 그래서 비밀번호 자리에 일반 비밀번호를 넣으면 다음과 같은 에러가 난다.

​```
remote: Password authentication is not supported for Git operations.
fatal: Authentication failed
​```

해결책은 **Personal Access Token(PAT)** 발급이다. 토큰은 비밀번호를 대신하는 일회성 인증 키라고 보면 된다.

### 토큰 발급 방법

1. GitHub 접속 후 `Settings → Developer settings → Personal access tokens → Tokens (classic)` 이동
   (또는 바로 `https://github.com/settings/tokens`)
2. **Generate new token (classic)** 선택
3. 권한(scope)에서 다음 두 가지를 반드시 체크
   - **repo** : 저장소 읽기/쓰기 권한
   - **workflow** : `.github/workflows/` 안의 워크플로우 파일을 다룰 때 필요
4. 생성된 토큰(`ghp_...`)을 복사해 안전한 곳에 보관 (페이지를 벗어나면 다시 볼 수 없다)

> **주의:** `repo`만 체크하고 `workflow`를 빠뜨리면, 워크플로우 파일을 push할 때 아래 에러가 난다.
>
> ​```
> refusing to allow a Personal Access Token to create or update workflow ... without `workflow` scope
> ​```
>
> GitHub Actions를 함께 쓰는 경우라면 `workflow` 권한을 처음부터 같이 체크하는 것이 좋다.

---

## 5. commit & push

이제 작업한 파일을 커밋하고 GitHub로 올린다.

​```bash
git add .
git commit -m "first commit"
git branch -M main
git push -u origin main
​```

각 명령의 의미는 다음과 같다.

- `git add .` : 변경된 모든 파일을 커밋 대상으로 등록(staging)
- `git commit -m "..."` : 변경 내용을 하나의 기록으로 확정
- `git branch -M main` : 기본 브랜치 이름을 `main`으로 지정
- `git push -u origin main` : `origin`의 `main` 브랜치로 업로드 (`-u`는 추적 연결 설정)

push 도중 인증 창이 뜨면 다음과 같이 입력한다.

- **Username** : GitHub 아이디
- **Password** : 앞서 발급한 **토큰** (계정 비밀번호 아님)

> 토큰을 붙여넣어도 터미널에 아무 글자도 보이지 않는 것이 정상이다. 보안상 입력값이 표시되지 않을 뿐, 정상 입력되고 있다.

---

## 매번 토큰 입력이 번거롭다면

push할 때마다 토큰을 입력하기 귀찮다면, 자격 증명을 저장해 둘 수 있다.

​```bash
git config --global credential.helper store
​```

다만 이 방식은 토큰을 평문 파일로 저장하므로, 공용 서버나 운영 서버에서는 보안에 유의해야 한다.

---

## 마치며

정리하면 **설치 → 사용자 설정 → 저장소 연결 → 토큰 인증 → push** 다섯 단계다. 이 중 처음 해보는 사람이 가장 많이 막히는 지점은 단연 4번 인증 단계다. "비밀번호를 넣었는데 왜 안 되지?"라는 의문이 든다면, 십중팔구 토큰이 답이다.

한 번 토큰 인증까지 뚫어두면, 그다음부터는 `add → commit → push` 세 줄이면 어디서든 작업한 내용을 GitHub에 올릴 수 있다.
