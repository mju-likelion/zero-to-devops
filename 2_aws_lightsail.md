# Lightsail로 Spring + MySQL 배포하기

### 멋사 백엔드 스터디 · Docker 편 다음 이야기

> 앞에서 만든 **Docker + Spring + MySQL** 프로젝트를 이제 진짜 인터넷에 올려봅니다.
내 노트북에서만 돌던 앱을 전 세계 누구나 `http://주소:8080` 으로 접속할 수 있게 만드는 게 목표예요.
Docker 편처럼 위에서 아래로 순서대로 따라오면 됩니다.
> 

---

## Part 0. Lightsail이 뭔데?

### 서버가 필요한 이유

내 노트북에서 앱을 띄우면, 노트북을 끄는 순간 접속이 끊깁니다. 24시간 켜져 있는 남의 컴퓨터(=서버)가 필요해요. 그 "항상 켜져 있는 컴퓨터를 빌려주는 서비스"가 클라우드입니다.

### Lightsail = "쉬운 버전의 AWS 서버"

AWS에는 EC2라는 강력한 서버 서비스가 있는데, 설정할 게 너무 많아서 초보자한텐 벽이 높아요. **Lightsail은 그걸 쉽게 만든 버전**입니다.

> **비유:** EC2가 "재료 하나하나 직접 골라 주문하는 수제버거"라면,
Lightsail은 **"세트 메뉴"** 예요. CPU·메모리·저장공간·트래픽을 한 세트로 묶어서 **월 정액 하나로** 딱 정해줍니다. 계산이 단순해서 처음 배우기 좋아요.
> 

참고: 익숙해지면 EC2로 넘어가는 게 자연스러운 수순입니다. 개념은 거의 그대로 이어져요.

### 요금 (2026년 기준, 꼭 알아둘 것)

- Linux 인스턴스: 월 **$5**(0.5GB), **$7**(1GB), **$12**(2GB RAM) ... 순으로 올라감
- **우리는 2GB RAM($12) 플랜 권장** — Spring 빌드 + MySQL 둘 다 돌리려면 메모리가 필요해요. 1GB 이하면 빌드하다 멈춥니다(뒤에서 설명).
- **신규 AWS 계정은 3개월 무료** 체험 가능 (위 플랜들 대상)
- **중요:** 인스턴스를 '중지(stop)'만 해도 요금은 계속 나갑니다. 안 쓸 거면 **삭제(delete)** 해야 과금이 멈춰요. (스터디 끝나고 방치하면 요금 폭탄!)

---

## Part 1. 인스턴스(서버) 만들기

**Lightsail 콘솔 접속** → `https://lightsail.aws.amazon.com` → **Create instance** 버튼

선택할 것들:

| 항목 | 선택 | 이유 |
| --- | --- | --- |
| **Region (지역)** | **Seoul (ap-northeast-2)** | 한국에서 가까울수록 속도가 빠름 |
| **Platform** | **Linux/Unix** | 우리는 Ubuntu를 쓸 거라서 |
| **Blueprint** | **OS Only → Ubuntu 24.04 LTS** | 여기 주의! 아래 설명 |
| **Plan (플랜)** | **2GB RAM / 2 vCPU** ($12) | Spring 빌드 + MySQL 동시 실행에 필요 |
| **Instance name** | 아무 이름 (예: `melsa-study`) | 알아보기 쉬운 이름 |

> **Blueprint에서 꼭 `OS Only`를 고르세요.** 위쪽에 'Apps + OS'(WordPress 등 미리 깔린 것들)가 있는데, 우리는 Docker를 직접 설치할 거라 **깨끗한 Ubuntu만** 필요합니다. 앱이 미리 깔린 걸 고르면 오히려 꼬여요.
> 

**Create instance** 누르면 1~2분 뒤 서버가 만들어집니다. 상태가 `Running`이 되면 성공.

---

## Part 2. 고정 IP(Static IP) 붙이기

**문제:** Lightsail 인스턴스의 공인 IP는 서버를 껐다 켜면 **바뀝니다.** 도메인 연결해놨는데 IP가 바뀌면 접속이 다 끊겨요.

**해결:** **고정 IP(Static IP)** 를 붙이면 IP가 절대 안 바뀝니다. 그리고 **인스턴스에 붙여둔 상태면 무료**예요. (안 붙이고 방치하면 과금되니 주의)

**방법:**

1. Lightsail 콘솔 → **Networking** 탭
2. **Create static IP**
3. 방금 만든 인스턴스(`melsa-study`)에 attach(연결)
4. 만들어진 고정 IP를 **메모해두기** (예: `43.201.xx.xx`) — 앞으로 이 IP로 접속합니다

> 처음부터 이걸 해두면, 나중에 도메인 붙일 때(다음 편 예고) 훨씬 편해요.
> 

---

## Part 3. 서버에 접속하기 (SSH)

SSH = 내 컴퓨터에서 원격 서버 안으로 들어가는 '보안 통로'입니다. 두 가지 방법:

### 방법 A. 브라우저 접속 (제일 쉬움, 초보 추천)

Lightsail 콘솔에서 인스턴스 클릭 → **Connect using SSH** 버튼 → 브라우저에 검은 터미널 창이 뜸. 끝!

### 방법 B. 내 터미널에서 접속 (익숙해지면)

1. 콘솔 → **Account → SSH keys**에서 기본 키(.pem 파일) 다운로드
2. 내 터미널에서:

```bash
# 키 파일 권한 설정 (최초 1회, 안 하면 접속 거부됨)
chmod 400 ~/Downloads/LightsailDefaultKey.pem

# 접속 (ubuntu = 기본 사용자 이름, 뒤는 내 고정 IP)
ssh -i ~/Downloads/LightsailDefaultKey.pem ubuntu@43.201.xx.xx
```

> 스터디에선 **방법 A(브라우저)** 로 시작하는 걸 추천해요. 키 파일 관리 실수로 삽질하는 시간을 아낄 수 있어요.
> 

접속되면 터미널 앞부분이 `ubuntu@ip-xxx:~$` 처럼 바뀝니다. 이제 여기가 **서버 안**이에요.

---

## Part 4. 서버 초기 세팅

접속했으면 서버에 필요한 것들을 깔아줍니다. **순서대로 복붙**하세요.

### ① 패키지 목록 업데이트

```bash
# 우분투의 설치 가능한 프로그램 목록을 최신으로 갱신
sudo apt update
```

### ② Docker 설치

```bash
# Docker 공식 설치 스크립트를 받아서 바로 실행 (제일 간편)
curl -fsSL https://get.docker.com | sudo sh

# 매번 sudo 없이 docker 명령을 쓰도록 내 계정을 docker 그룹에 추가
sudo usermod -aG docker $USER
```

> 위 `usermod`은 **재접속해야 적용**됩니다. 터미널을 닫았다가 다시 SSH 접속하세요. (브라우저 SSH면 창 닫고 Connect 다시)
> 

재접속 후 설치 확인:

```bash
docker --version          # 버전이 뜨면 성공
docker compose version    # 이것도 버전이 뜨면 성공
```

### ③ 스왑 메모리 설정 (빌드 중 멈춤 방지)

2GB 플랜이라도, Gradle 빌드는 순간적으로 메모리를 많이 먹어서 서버가 뻗을 수 있어요. **스왑**(부족한 메모리를 디스크로 임시 보충)을 잡아두면 안전합니다.

```bash
# 2GB 크기의 스왑 파일 생성
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile        # 보안: 소유자만 접근
sudo mkswap /swapfile           # 스왑 영역으로 포맷
sudo swapon /swapfile           # 스왑 켜기

# 서버 재부팅해도 스왑이 유지되도록 등록
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 확인 (Swap 줄에 2.0Gi 뜨면 성공)
free -h
```

---

## Part 5. 방화벽 열기 — 로컬에선 되는데 외부 접속 안 될 때 범인

**꼭 이해할 것:** 방화벽이 **두 겹**이에요. 둘 다 열려야 외부에서 접속됩니다.

```
외부 사용자 ──> [① AWS 방화벽] ──> [② 서버 OS 방화벽] ──> 우리 앱(8080)
                (Lightsail 콘솔)      (ufw)
```

### ① AWS 레벨 방화벽 (Lightsail 콘솔) — 필수!

1. Lightsail 콘솔 → 인스턴스 클릭 → **Networking** 탭
2. **IPv4 Firewall** 섹션 → **Add rule**
3. **Custom / TCP / Port `8080`** 추가 → 저장

> 이게 초보자 배포 실패 1순위 원인입니다. "로컬 도커에선 되는데 브라우저에서 안 떠요"의 90%가 이 8080 포트를 안 열어서예요. (22번 SSH, 80/443 웹 포트는 보통 기본으로 열려 있음)
> 

### ② OS 레벨 방화벽 (ufw) — 켜져 있다면

Ubuntu의 ufw 방화벽이 켜져 있다면 여기도 열어줘야 해요. (Lightsail 기본 이미지는 대개 꺼져 있어서 생략 가능하지만, 확인차)

```bash
sudo ufw status                 # 상태 확인. 'inactive'면 신경 안 써도 됨
# 만약 active라면:
sudo ufw allow 8080
sudo ufw allow 22               # SSH 포트도 꼭 열어둬야 접속 안 끊김!
```

---

## Part 6. 코드 올리고 배포하기

이제 진짜 배포입니다. 앞 Docker 편에서 만든 프로젝트를 서버로 가져옵니다.

### ① 프로젝트 내려받기

```bash
# 깃허브 레포를 서버로 복제
git clone https://github.com/내아이디/내프로젝트.git

# 프로젝트 폴더로 이동
cd 내프로젝트
```

### ② .env 파일 직접 만들기 (중요!)

`.env`는 `.gitignore`로 깃에서 제외했으니 **서버엔 없습니다.** 여기서 직접 만들어요.

```bash
# nano 편집기로 .env 생성
nano .env
```

편집기가 열리면 아래 내용 입력 (비밀번호는 강력하게!):

```bash
MYSQL_ROOT_PASSWORD=서버용_강력한_루트비번
MYSQL_DATABASE=myapp
MYSQL_USER=appuser
MYSQL_PASSWORD=서버용_강력한_앱비번
```

저장: `Ctrl + O` → `Enter` → `Ctrl + X` (nano 나가기)

### ③ 실행!

```bash
# 이미지 빌드하고 컨테이너들을 백그라운드로 실행
docker compose up -d --build
```

> 처음 실행은 이미지 다운로드 + Gradle 빌드 때문에 **몇 분 걸립니다.** 정상이에요. 커피 한 잔 ☕
> 

### ④ 잘 떴는지 확인

```bash
# 컨테이너 상태 확인 (app, db 둘 다 Up / healthy 여야 함)
docker compose ps

# 로그 실시간 확인 (Spring이 정상 기동됐는지)
docker compose logs -f app
```

로그에서 `Started ...Application in X seconds` 같은 메시지가 뜨면 성공! (`Ctrl + C`로 로그 보기 종료)

### ⑤ 브라우저에서 접속

```
http://내_고정IP:8080
```

예: `http://43.201.xx.xx:8080`

여기서 앱이 뜨면 **배포 성공입니다!** 전 세계 누구나 이 주소로 접속할 수 있어요.

---

## Part 7. 코드 수정하면 어떻게 다시 올려? (운영)

개발하다 보면 코드가 계속 바뀌죠. 수정한 걸 서버에 반영하는 흐름:

```bash
# 1. (내 컴퓨터에서) 코드 수정 후 깃허브에 push
git add .
git commit -m "기능 수정"
git push

# 2. (서버에서) 최신 코드 받고 다시 빌드/실행
cd 내프로젝트
git pull                          # 최신 코드 가져오기
docker compose up -d --build      # 다시 빌드하고 교체 (자동으로 기존 것 내리고 새로 띄움)
```

> DB 데이터는 볼륨에 저장되니까 이렇게 재배포해도 **데이터는 안 날아갑니다.** (앞 Docker 편의 볼륨 개념 기억나죠?)
> 

---

## Part 8. 자주 나는 에러 & 해결법 (Lightsail 편)

### 브라우저에서 `사이트에 연결할 수 없음` (로컬 도커는 되는데)

**원인:** AWS 방화벽에 8080 포트 안 열림 (1순위!)
**해결:** Part 5의 ① — Lightsail 콘솔 Networking에서 8080 포트 규칙 추가.

### 서버를 껐다 켰더니 IP가 바뀌어서 접속 안 됨

**원인:** 고정 IP를 안 붙임.
**해결:** Part 2 — Static IP 생성 후 인스턴스에 attach.

### 빌드 중 서버가 멈추거나 SSH가 끊기고 `Killed` 뜸

**원인:** 메모리 부족(OOM).
**해결:** 2GB 플랜 사용 + Part 4의 ③ 스왑 설정. (그래도 안 되면 상위 플랜)

### `docker: command not found`

**원인:** Docker 미설치 또는 usermod 후 재접속 안 함.
**해결:** Part 4의 ② 다시 확인. `usermod` 후 **반드시 재접속**.

### `Permission denied` (docker 명령 실행 시 sudo 없이)

**원인:** docker 그룹 적용 전.
**해결:** `sudo usermod -aG docker $USER` 실행 후 **재접속**. 급하면 `sudo docker ...`로 임시 사용.

### `git clone` 할 때 인증 오류 (private 레포)

**원인:** 비공개 저장소는 인증 필요.
**해결:** 레포를 public으로 하거나, GitHub **Personal Access Token** 사용. 스터디에선 public 레포로 시작하는 걸 추천.

### SSH 접속이 아예 안 됨

**원인:** 22번 포트를 실수로 막았거나, ufw로 SSH를 차단.
**해결:** Lightsail 콘솔의 **브라우저 SSH**로 우회 접속 → `sudo ufw allow 22` 확인. (그래서 ufw 켤 때 22번을 꼭 같이 열라고 한 거예요.)

---

## 오늘의 3줄 요약

1. **Lightsail = 월정액 세트 메뉴 서버.** 2GB 플랜 + Ubuntu OS Only로 만들고, **고정 IP를 꼭 붙인다** (안 그러면 IP가 바뀜).
2. **서버 세팅 = Docker 설치 + 스왑 + 방화벽 8080 열기.** 방화벽은 AWS 레벨과 OS 레벨 **두 겹**이라는 걸 기억!
3. **배포 = git clone → .env 직접 작성 → `docker compose up -d --build`.** 수정 후엔 `git pull` + 같은 명령으로 재배포. 데이터는 볼륨에 살아남는다.

배포 성공했으면 이제 여러분 앱이 **진짜 인터넷 위에** 떠 있는 거예요. 
막히면 항상 `docker compose logs -f app`으로 로그부터 보기!

---

### 다음 편 예고

지금은 `http://IP:8080` 처럼 IP랑 포트번호가 보기 안 좋죠? 다음 편에서는 **도메인을 연결하고 HTTPS(자물쇠 )를 적용**해서 `https://내도메인.com` 처럼 깔끔하게 만듭니다. (Caddy라는 도구로 자동 HTTPS를 걸어요.)
