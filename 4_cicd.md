# CI/CD 편 (GitHub Actions 자동 배포)

> 지난 편까지 우리 앱은 `https://내도메인.com` 으로 자물쇠까지 딱 뜨게 배포됐습니다.
> 그런데 코드를 고칠 때마다 이걸 반복하고 있죠?
> 1. 내 컴퓨터에서 `git push`
> 2. 서버에 SSH 접속
> 3. `git pull`
> 4. `docker compose up -d --build`
>
> 매번 서버 들어가서 손으로 치는 거, 귀찮고 실수도 나요. 오늘은 이 **2~4번을 자동으로** 만듭니다. 목표: **`main` 브랜치에 push만 하면 서버 배포까지 알아서 끝나기.**

---

## Part 0. CI/CD가 뭔데?

용어부터 딱 정리하고 갑시다. 어렵게 외울 필요 없어요.

- **CI (Continuous Integration, 지속적 통합):** 코드를 올리면 자동으로 **빌드하고 테스트**해서 "이거 문제 없어?"를 확인하는 것.
- **CD (Continuous Deployment, 지속적 배포):** 그렇게 확인된 코드를 자동으로 **서버에 배포**하는 것.

우리는 오늘 이 둘을 합쳐서, **push 한 번 → 서버에 자동 배포**되는 파이프라인을 만듭니다.

### 오늘 할 일 지도

```
내 컴퓨터              GitHub                        우리 서버(Lightsail)
git push  ──────>  [GitHub Actions 실행]  ──SSH──>  git pull
main 브랜치          (워크플로우 파일이            docker compose up -d --build
                     시키는 대로 자동 진행)
```

> 핵심 아이디어: 지난 편까지 **우리가 손으로 하던 서버 작업**을, GitHub Actions라는 로봇 비서한테 대신 시키는 겁니다. "push 되면 → 서버 접속해서 → 이 명령들 쳐줘" 라고 대본(워크플로우 파일)을 써두는 거예요.

---

## Part 1. GitHub Actions가 뭔데? (30초 요약)

**GitHub Actions**는 GitHub이 무료로 제공하는 자동화 도구입니다. 내 레포에 `.github/workflows/` 폴더를 만들고 그 안에 YAML 파일(대본)을 두면, **특정 사건이 일어날 때마다** GitHub이 그 대본을 자동으로 실행해줘요.

- **언제 실행?** 예: "`main` 브랜치에 push 될 때마다"
- **무엇을 실행?** 예: "서버에 SSH 접속해서 배포 명령 실행"
- **어디서 실행?** GitHub이 빌려주는 임시 컴퓨터(runner)에서. 우리는 이 runner가 우리 서버에 접속하도록 시킬 거예요.

> 무료 사용량이 넉넉해서(public 레포는 사실상 무제한) 스터디 규모에선 요금 걱정 안 해도 됩니다.

---

## Part 2. 배포 방식 정하기 — 우리는 "SSH 접속" 방식

자동 배포에는 여러 방식이 있는데, 우리 상황(Lightsail + docker compose)에 제일 단순하고 직관적인 건 **"GitHub Actions가 서버에 SSH로 들어가서 우리가 원래 치던 명령을 대신 쳐주는"** 방식입니다.

```
GitHub Actions runner  ──(SSH 접속)──>  Lightsail 서버
                                         └ git pull
                                         └ docker compose up -d --build
```

이 방식의 장점: **지난 편까지 손으로 하던 것과 똑같아요.** 새로운 개념이 거의 없어서 이해하기 쉽습니다. runner가 그냥 "나 대신 서버 들어가서 그 명령 쳐주는 사람"인 거죠.

> 그러려면 runner가 우리 서버에 접속할 수 있어야 하고, 그러려면 **SSH 키**가 필요합니다. Part 3에서 이걸 준비해요.

---

## Part 3. 서버 접속용 SSH 키 준비하기

GitHub Actions가 우리 서버에 들어오려면 열쇠(SSH 키)가 필요합니다. 지난 편에서 브라우저 SSH를 썼다면 키가 없을 수 있으니, **배포 전용 키를 새로 하나** 만들어서 넣어줄게요. (원래 쓰던 키를 GitHub에 올리는 것보다 안전해요.)

### ① 서버에 접속

Lightsail 콘솔 → 인스턴스 → **Connect using SSH** (브라우저 접속)

### ② 배포 전용 키 만들기 (서버 안에서)

```bash
# 배포용 키 쌍 생성 (비밀번호는 그냥 Enter로 비워둠 - 자동화라 사람이 입력 못 함)
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -N ""
```

이러면 두 개가 생깁니다.

- `~/.ssh/deploy_key` → **개인키** (GitHub한테 줄 열쇠)
- `~/.ssh/deploy_key.pub` → **공개키** (서버 자물쇠에 등록할 것)

### ③ 공개키를 서버에 등록 (이 서버가 이 열쇠를 받아들이도록)

```bash
# 공개키를 인증된 키 목록에 추가
cat ~/.ssh/deploy_key.pub >> ~/.ssh/authorized_keys
```

### ④ 개인키 내용 복사해두기 (GitHub에 넣을 것)

```bash
# 개인키 전체 출력 - 이 내용을 통째로 복사
cat ~/.ssh/deploy_key
```

> 출력되는 내용을 **`-----BEGIN...` 부터 `-----END...` 줄까지 통째로** 복사하세요. 한 줄이라도 빠지면 접속이 안 됩니다. 이 값은 곧 GitHub Secrets에 넣을 거예요. **절대 코드에 직접 쓰거나 커밋하면 안 됩니다.**

---

## Part 4. GitHub Secrets에 비밀값 넣기

방금 만든 개인키, 서버 IP, 사용자 이름 같은 **민감한 값들**은 코드에 그대로 쓰면 안 됩니다. GitHub이 제공하는 금고인 **Secrets**에 넣어두고, 워크플로우에서 이름으로 꺼내 씁니다.

### 설정 위치

내 레포 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

### 넣을 값 3개

| 이름(Name) | 값(Value) | 설명 |
| --- | --- | --- |
| `SERVER_HOST` | 내 고정 IP (예: `43.201.xx.xx`) | 접속할 서버 주소 |
| `SERVER_USER` | `ubuntu` | Lightsail 기본 사용자 이름 |
| `SSH_PRIVATE_KEY` | Part 3 ④에서 복사한 개인키 전체 | 서버 접속 열쇠 |

> Secrets에 넣은 값은 **다시 볼 수 없습니다** (수정만 가능). GitHub이 로그에도 자동으로 가려주기 때문에 안전해요. 이게 `.env`를 깃에 안 올리던 것과 같은 원리(비밀은 코드 밖에)입니다.

---

## Part 5. 워크플로우 파일 작성하기 (자동화 대본)

이제 대본을 씁니다. **내 컴퓨터의 프로젝트 폴더**에서 작업해요 (서버 아님!).

### ① 폴더와 파일 만들기

프로젝트 루트에 `.github/workflows/` 폴더를 만들고 그 안에 `deploy.yml` 파일을 생성합니다.

```
내프로젝트/
├── .github/
│   └── workflows/
│       └── deploy.yml   ← 이 파일을 만듭니다
├── docker-compose.yml
├── Caddyfile
└── ...
```

### ② deploy.yml 내용

```yaml
name: Deploy to Lightsail

# 언제 실행할지: main 브랜치에 push 될 때
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest    # GitHub이 빌려주는 임시 컴퓨터

    steps:
      # 서버에 SSH 접속해서 배포 명령 실행
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/내프로젝트
            git pull origin main
            docker compose up -d --build
```

> `${{ secrets.XXX }}` 부분이 Part 4에서 넣어둔 금고 값을 꺼내 쓰는 문법이에요. `appleboy/ssh-action`은 "SSH 접속해서 script 실행"을 대신 해주는, 많이 쓰이는 공개 도구(action)입니다. `script:` 아래 명령들이 바로 **우리가 손으로 치던 그 명령**이에요.

> `cd ~/내프로젝트` 의 경로를 **본인 서버의 실제 프로젝트 폴더 이름**으로 바꾸세요. (지난 편에서 `git clone` 했을 때 생긴 폴더 이름)

---

## Part 6. 실행 & 확인

### ① 워크플로우 파일을 push

```bash
# 내 컴퓨터에서
git add .github/workflows/deploy.yml
git commit -m "CI/CD: 자동 배포 워크플로우 추가"
git push origin main
```

### ② GitHub에서 실행 지켜보기

내 레포 → **Actions** 탭 → 방금 push로 실행된 워크플로우가 보입니다. 클릭하면 진행 상황이 실시간으로 뜹니다.

- 노란 점 = 실행 중
- 초록 체크 = 성공
- 빨간 X = 실패 (클릭해서 로그 보면 어디서 막혔는지 나옴)

### ③ 진짜 자동 배포 테스트

이제 코드를 아무거나 조금 고쳐서 push 해보세요.

```bash
# 예: 화면 문구 하나 수정 후
git add .
git commit -m "테스트: 자동 배포 확인"
git push origin main
```

Actions 탭에서 워크플로우가 **자동으로 도는** 게 보이고, 초록 체크가 뜬 뒤 `https://내도메인.com` 에 들어가면 수정한 내용이 반영돼 있습니다. **이제 서버에 SSH로 안 들어가도 돼요!** 🎉

---

## Part 7. 자주 나는 에러 & 해결법

### Actions 로그에 `ssh: handshake failed` / `permission denied`

**원인:** SSH 키 문제 (1순위).
**해결:**
- `SSH_PRIVATE_KEY` Secret에 개인키를 `-----BEGIN` ~ `-----END` 줄까지 **통째로** 넣었는지 확인.
- 공개키(`deploy_key.pub`)를 서버 `~/.ssh/authorized_keys`에 등록했는지 확인 (Part 3 ③).

### `Host key verification failed`

**원인:** 서버를 처음 접속하는 거라 runner가 서버를 신뢰 안 함.
**해결:** `appleboy/ssh-action`은 보통 알아서 처리하지만, 안 되면 워크플로우 `with:`에 다음 줄 추가.

```yaml
          fingerprint: ""   # 또는 host key 검증을 건너뛰는 옵션 사용
```

### 접속은 됐는데 `cd: ~/내프로젝트: No such file or directory`

**원인:** `script`의 경로가 서버 실제 폴더명과 다름.
**해결:** 서버에서 `ls ~` 로 실제 폴더 이름 확인 후 `deploy.yml`의 `cd` 경로 수정.

### `git pull` 에서 인증 오류 (private 레포)

**원인:** 비공개 레포는 서버가 GitHub에서 코드를 당겨올 때 인증이 필요함.
**해결:** 레포를 public으로 하거나, 서버에 배포용 Deploy Key/토큰을 설정. 스터디에선 public으로 시작하는 걸 추천.

### 워크플로우가 아예 실행이 안 됨

**원인:** 파일 위치나 이름이 틀림.
**해결:** 경로가 정확히 `.github/workflows/deploy.yml` 인지 확인 (`.github` 앞의 점, 폴더 이름 철자 주의). `on: push: branches: [ main ]` 의 브랜치 이름이 실제 기본 브랜치와 같은지도 확인 (`master`인 경우도 있음).

### 빌드 중 서버가 멈춤 (`Killed`)

**원인:** 메모리 부족(OOM). 자동 배포라도 빌드는 서버에서 도는 건 똑같음.
**해결:** Lightsail 편 Part 4의 스왑 설정이 되어 있는지 확인.

---

## 오늘의 3줄 요약

1. **CI/CD = push 한 번에 자동 배포.** GitHub Actions가 "우리 대신 서버에 SSH 들어가서 `git pull` + `docker compose up -d --build`를 쳐주는 로봇 비서"다.
2. **준비물 = 배포 전용 SSH 키 + GitHub Secrets 3개**(`SERVER_HOST`, `SERVER_USER`, `SSH_PRIVATE_KEY`). 비밀값은 코드 밖 금고에 (`.env`를 깃에 안 올리던 것과 같은 원리).
3. **대본 = `.github/workflows/deploy.yml`.** `main`에 push되면 자동 실행. Actions 탭에서 초록 체크 뜨면 성공. 이제 서버에 손으로 안 들어가도 된다.

이제 여러분은 **코드만 고치고 push하면**, 배포는 알아서 됩니다. 개발에만 집중할 수 있게 된 거예요. 🤖

---

### 다음 편 예고

배포는 이제 자동으로 되는데... **배포된 앱이 새벽에 조용히 죽으면 우리는 그걸 어떻게 알까요?** 다음 편에서는 **서버 상태를 자동으로 감시(모니터링)하고, 앱이 죽으면 디스코드/슬랙으로 알림이 오게** 만듭니다. 폰에 띠링 울리는 그거요. 🔔
