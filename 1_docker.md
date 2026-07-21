# Docker로 Spring Boot + MySQL 배포하기 🐳

### 멋사 백엔드 스터디 · Docker를 처음 만지는 사람을 위한 가이드

> 이 문서는 위에서 아래로 순서대로 읽으면 됩니다.
"개념 → 용어 → 코드 한 줄씩 → 명령어 → 자주 나는 에러" 순서예요.
코드는 **복붙하기 전에 주석부터 읽는 걸** 강력 추천합니다. 왜 이렇게 쓰는지 알아야 에러가 났을 때 고칠 수 있어요.
> 

---

## Part 0. 그래서 Docker가 대체 뭔데?

### "제 컴퓨터에선 됐는데요?" 문제

여러분이 만든 Spring 프로젝트를 친구 노트북에서 돌린다고 해봅시다. 그런데 친구는:

- Java 버전이 다르고 (친구는 17, 나는 21)
- MySQL이 안 깔려 있고
- 깔려 있어도 버전이 다르고
- 운영체제가 Windows / Mac / Linux 로 제각각

그래서 **"내 컴퓨터에선 되는데 왜 너는 안 돼?"** 라는 상황이 계속 생깁니다. 서버(Lightsail)에 올릴 때도 똑같아요. 서버 환경이 내 노트북이랑 다르니까요.

### Docker의 해결책: "환경을 통째로 상자에 담는다"

Docker는 **내 앱 + 앱이 필요로 하는 모든 것(Java, 라이브러리, 설정)을 하나의 상자(컨테이너)에 담아서** 어디서든 똑같이 실행되게 만들어줍니다.

> 🚢 **비유:** 옛날엔 배로 짐을 나를 때 물건 모양이 제각각이라 싣기 힘들었어요.
그런데 **규격화된 컨테이너 박스**에 담기 시작하니, 안에 뭐가 들었든 배·기차·트럭 어디든 똑같이 실을 수 있게 됐죠.
Docker의 "컨테이너"가 딱 이겁니다. 안에 뭐가 들었든 Docker만 깔려 있으면 어디서든 똑같이 돌아가요.
> 

### 그럼 가상머신(VM)이랑 뭐가 달라?

VM은 **운영체제 전체를 통째로** 가상으로 만들어서 무겁고 느립니다.
Docker 컨테이너는 **OS는 공유하고 앱 실행에 필요한 부분만** 격리해서 훨씬 가볍고 빠릅니다. 노트북에서 컨테이너 여러 개 띄워도 부담이 적어요.

---

## Part 1. 딱 6개 용어만 먼저 외우자

이 6개만 알면 오늘 실습 내용의 90%가 이해됩니다.

| 용어 | 한 줄 정의 | 비유 |
| --- | --- | --- |
| **이미지 (Image)** | 실행에 필요한 걸 다 담아 얼려놓은 '설계도/틀' | 붕어빵 **틀** |
| **컨테이너 (Container)** | 이미지를 실제로 실행한 '살아있는 인스턴스' | 틀로 찍어낸 **붕어빵** |
| **Dockerfile** | 이미지를 어떻게 만들지 적은 '레시피' | 붕어빵 틀 **만드는 법** |
| **볼륨 (Volume)** | 컨테이너가 꺼져도 데이터를 살려두는 '외장 창고' | 데이터 **금고** |
| **네트워크 (Network)** | 컨테이너끼리 서로 통신하는 '통로' | 컨테이너 간 **전화선** |
| **Docker Compose** | 컨테이너 여러 개를 한 파일로 지휘하는 '지휘자' | **오케스트라 지휘자** |

### 특히 중요한 3가지 짚고 넘어가기

**① 이미지 vs 컨테이너 (제일 헷갈림)**
이미지는 '틀'이라서 하나로 여러 컨테이너를 찍어낼 수 있어요. 붕어빵 틀 하나로 붕어빵 여러 개 만드는 것처럼요. `docker run`을 하면 이미지 → 컨테이너가 됩니다.

**② 볼륨이 왜 필요해?**
컨테이너는 껐다 켜면 안에 있던 데이터가 다 날아갑니다. (붕어빵을 버리면 안에 든 팥도 같이 사라지듯) MySQL 데이터가 컨테이너 안에만 있으면, 컨테이너 재시작할 때 DB가 통째로 날아가요. 그래서 데이터는 **볼륨(외장 창고)** 에 따로 저장해서 컨테이너가 죽어도 살아남게 합니다.

**③ Compose를 왜 써?**
우리는 컨테이너가 2개예요 (Spring 앱 + MySQL). 이걸 하나하나 `docker run` 명령어로 띄우고 네트워크 연결하고... 하면 명령어가 길고 복잡합니다. Compose는 이 모든 걸 **`docker-compose.yml` 파일 하나**에 적어두고 `docker compose up` 한 방으로 다 띄웁니다.

---

## Part 2. 우리가 오늘 만들 구조

```
┌─────────────────────────────────────────────┐
│          Lightsail 서버 (Ubuntu)              │
│                                               │
│   ┌─────────────────────────────────────┐    │
│   │           Docker Compose            │    │
│   │                                     │    │
│   │   ┌──────────┐      ┌──────────┐    │    │
│   │   │   app    │─────>│    db    │    │    │
│   │   │ (Spring) │ 호출 │ (MySQL)  │    │    │
│   │   │  :8080   │      │  :3306   │    │    │
│   │   └──────────┘      └────┬─────┘    │    │
│   │                          │          │    │
│   │                     ┌────▼─────┐    │    │
│   │                     │  볼륨    │    │    │
│   │                     │(DB 데이터)│    │    │
│   │                     └──────────┘    │    │
│   └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
        ▲
        │ http://서버IP:8080
     사용자
```

**핵심:** `app` 컨테이너와 `db` 컨테이너는 Compose가 자동으로 만들어준 네트워크 안에서 **서비스 이름(`app`, `db`)으로 서로를 찾습니다.** 이게 뒤에서 제일 중요해요.

---

## Part 3. 코드 한 줄씩 뜯어보기

프로젝트 폴더 구조는 이렇게 됩니다:

```
my-project/
├── src/                    ← Spring 소스코드
├── build.gradle
├── gradlew
├── gradle/
├── Dockerfile              ← ① 이미지 만드는 레시피
├── docker-compose.yml      ← ② 컨테이너들 지휘하는 파일
├── .env                    ← ③ 비밀번호 등 민감정보 (깃에 올리면 안 됨!)
└── .gitignore              ← ④ .env를 깃에서 제외
```

### ① Dockerfile — Spring 앱을 이미지로 만드는 레시피

> 이 파일은 "내 Spring 코드를 어떻게 실행 가능한 이미지로 구울지"를 적은 설명서예요.
`멀티 스테이지`라는 방식을 씁니다. **빌드하는 방과 실행하는 방을 나누는** 거예요.
(빌드엔 무거운 JDK가 필요하지만, 실행엔 가벼운 JRE면 충분하니까 최종 이미지를 가볍게 만들려고요.)
> 

```docker
# ============================================
#  [1단계] 빌드 스테이지 — 여기서 jar 파일을 만든다
# ============================================

# eclipse-temurin:21-jdk 이미지를 기반으로 시작.
#   - temurin: 무료 오픈소스 Java 배포판
#   - 21: Java 버전 (우리 프로젝트가 Java 21이니까 맞춰줌)
#   - jdk: 빌드하려면 컴파일러가 포함된 JDK가 필요
#   - AS builder: 이 스테이지에 'builder'라는 별명을 붙임 (2단계에서 씀)
FROM eclipse-temurin:21-jdk AS builder

# 컨테이너 안에서 작업할 폴더를 /app 으로 지정.
# 이후 명령어들은 전부 이 폴더 기준으로 실행됨.
WORKDIR /app

# 내 프로젝트 파일들을 컨테이너 안으로 복사한다.
# (아래 순서대로 복사하는 이유: Docker는 '변하지 않는 것 먼저' 복사하면
#  캐시를 재활용해서 빌드가 빨라짐. 소스코드는 자주 바뀌니까 맨 나중에.)
COPY gradlew .
COPY gradle gradle
COPY build.gradle settings.gradle ./

# gradlew(그래들 실행 파일)에 실행 권한을 줌.
# 이거 안 하면 'Permission denied' 에러가 남.
RUN chmod +x gradlew

# 이제 소스코드를 복사 (제일 자주 바뀌는 부분이라 마지막에)
COPY src src

# 그래들로 실제 빌드! bootJar = 실행 가능한 jar 파일을 만든다.
#   - clean: 이전 빌드 결과물 삭제하고 새로
#   - --no-daemon: 컨테이너 빌드에선 데몬(백그라운드 프로세스) 안 씀
# 결과물: /app/build/libs/ 안에 xxx.jar 생성됨
RUN ./gradlew clean bootJar --no-daemon

# ============================================
#  [2단계] 실행 스테이지 — 만든 jar만 가져와서 실행
# ============================================

# 이번엔 jre(실행 전용, JDK보다 훨씬 가벼움) 이미지로 새로 시작.
# 빌드 도구들은 다 버리고 실행에 필요한 것만 남겨서 이미지를 슬림하게!
FROM eclipse-temurin:21-jre
WORKDIR /app

# 1단계(builder)에서 만든 jar 파일만 쏙 빼서 이 스테이지로 복사.
# --from=builder : "builder라는 별명의 스테이지에서 가져와라"
# *.jar : 이름이 뭐든 jar 파일을 app.jar 라는 이름으로 복사
COPY --from=builder /app/build/libs/*.jar app.jar

# 이 컨테이너가 8080 포트를 쓴다고 '문서상' 표시.
# (실제 외부 연결은 compose의 ports에서 함. 여긴 일종의 안내판)
EXPOSE 8080

# 컨테이너가 시작될 때 실행할 명령어.
# = 터미널에서 java -jar app.jar 를 친 것과 같음
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### ② docker-compose.yml — 컨테이너 2개를 지휘하는 파일 (가장 중요!)

> 이 파일 하나로 MySQL과 Spring을 동시에 띄우고, 서로 연결하고, 데이터도 저장합니다.
**주석 하나하나 다 읽으세요.** 여기가 오늘의 핵심입니다.
> 

```yaml
# services: 우리가 띄울 컨테이너들의 목록.
# 여기 적은 'db', 'app'이 곧 컨테이너의 이름이자, 서로를 부르는 주소가 됨.
services:

  # ─────────────────────────────
  #  db: MySQL 데이터베이스 컨테이너
  # ─────────────────────────────
  db:
    # 직접 만들 필요 없이 Docker Hub에서 공식 MySQL 8.0 이미지를 가져옴
    image: mysql:8.0

    # MySQL 컨테이너에 넘겨줄 환경변수들.
    # ${...} 는 아래 .env 파일에서 값을 가져온다는 뜻 (비번을 여기 직접 안 씀!)
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}  # 루트(최고관리자) 비번
      MYSQL_DATABASE: ${MYSQL_DATABASE}            # 시작할 때 만들 DB 이름
      MYSQL_USER: ${MYSQL_USER}                    # 앱이 쓸 일반 계정
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}            # 그 계정 비번

    # MySQL 실행 옵션. 한글이 안 깨지게 utf8mb4 문자셋을 기본으로 지정.
    # (이거 안 하면 한글이 ??? 로 저장되는 대참사 발생)
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci

    # 볼륨 연결: 컨테이너 안의 DB 데이터 폴더(/var/lib/mysql)를
    # 'mysql-data'라는 볼륨(외장 창고)에 저장.
    # → 컨테이너를 껐다 켜도 데이터가 살아남음!
    volumes:
      - mysql-data:/var/lib/mysql

    # 헬스체크: "이 DB가 진짜 접속 받을 준비가 됐는지" 주기적으로 확인.
    # MySQL은 컨테이너가 떠도 실제로 준비되기까지 몇 초 걸리는데,
    # 그 사이에 Spring이 먼저 붙으러 가면 에러남. 그래서 준비 신호가 필요.
    healthcheck:
      # mysqladmin ping = "DB야 살아있니?" 하고 물어보는 명령
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s   # 10초마다 물어봄
      timeout: 5s     # 5초 안에 답 없으면 실패로 침
      retries: 5      # 5번까지 재시도

    # 컨테이너가 (에러 등으로) 죽으면 자동으로 다시 살림.
    # (단, 내가 일부러 멈춘 경우는 재시작 안 함)
    restart: unless-stopped

  # ─────────────────────────────
  #  app: Spring Boot 앱 컨테이너
  # ─────────────────────────────
  app:
    # image 대신 build: . → 현재 폴더의 Dockerfile로 이미지를 직접 빌드
    build: .

    # 포트 매핑: "서버의 8080 포트 → 컨테이너의 8080 포트" 연결.
    # 왼쪽이 바깥(서버), 오른쪽이 안(컨테이너).
    # 이게 있어야 외부에서 http://서버IP:8080 으로 접속 가능.
    ports:
      - "8080:8080"

    # Spring에게 넘겨줄 환경변수. application.yml에서 이 값들을 받아 씀.
    environment:
      # ★★★ 제일 중요 ★★★
      # DB 주소의 호스트가 'localhost'가 아니라 'db'!
      # 'db'는 위에서 정의한 MySQL 컨테이너의 서비스 이름.
      # 컨테이너 안에서 localhost는 '자기 자신'이라 DB를 못 찾음.
      # Compose 네트워크에선 서비스 이름이 곧 주소가 됨.
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/${MYSQL_DATABASE}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul&characterEncoding=UTF-8
      #   ?뒤 옵션 설명:
      #   - useSSL=false: 로컬/내부 통신이라 SSL 끔
      #   - allowPublicKeyRetrieval=true: MySQL 8 인증 방식 때문에 필수!
      #     (이거 없으면 'Public Key Retrieval is not allowed' 에러남)
      #   - serverTimezone=Asia/Seoul: 시간을 한국 시간으로
      #   - characterEncoding=UTF-8: 한글 인코딩
      SPRING_DATASOURCE_USERNAME: ${MYSQL_USER}
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_PASSWORD}

    # depends_on: "db가 준비된 다음에 app을 띄워라"
    # condition: service_healthy → 위 헬스체크가 통과할 때까지 기다림.
    # (이게 없으면 Spring이 DB보다 먼저 떠서 'Connection refused'로 죽음)
    depends_on:
      db:
        condition: service_healthy

    restart: unless-stopped

# 위에서 쓴 'mysql-data' 볼륨을 여기서 공식 선언.
# Docker가 알아서 관리하는 저장 공간이 만들어짐.
volumes:
  mysql-data:
```

### ③ .env — 민감정보를 담는 파일 (절대 깃에 올리지 마세요!)

> Compose는 같은 폴더의 `.env` 파일을 **자동으로** 읽어서 `${...}` 자리에 값을 채워줍니다.
덕분에 비밀번호가 코드(깃)에 안 들어가요. 이게 보안의 핵심입니다.
> 

```bash
# .env
# 이 파일은 절대 깃에 커밋하지 마세요! (아래 .gitignore로 막습니다)

MYSQL_ROOT_PASSWORD=여기에_강력한_루트비번
MYSQL_DATABASE=myapp
MYSQL_USER=appuser
MYSQL_PASSWORD=여기에_강력한_앱비번
```

### ④ .gitignore — .env를 깃에서 제외

```bash
# .gitignore 에 아래 줄 추가

.env

# 그 외 흔히 제외하는 것들
build/
.gradle/
*.log
```

> **왜 이렇게까지 하냐면:** 비밀번호를 코드에 적어서 깃허브에 올리면, 봇들이 몇 분 만에 긁어가서 서버를 털어갑니다. 실제로 자주 일어나는 사고예요. 한번 커밋되면 히스토리에서 지우기도 골치아프고, 비번도 전부 갈아야 합니다. **처음부터 .env로 분리하는 습관**을 들이세요.
> 

### ⑤ application.yml — Spring 설정 (환경변수로 받게)

```yaml
spring:
  datasource:
    # ${환경변수} 로 받음 → 실제 값은 docker-compose가 넣어줌.
    # 코드에 DB 정보를 하드코딩하지 않는 게 핵심.
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      # update: 엔티티에 맞춰 테이블 자동 생성/수정 (학습용으로 편함)
      # (실무에선 validate + 마이그레이션 도구를 권장하지만 스터디에선 update로 시작)
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```

> `build.gradle`에 MySQL 드라이버가 있는지 꼭 확인:
> 
> 
> ```
> runtimeOnly 'com.mysql:mysql-connector-j'
> ```
> 

---

## Part 4. 명령어 사전

| 명령어 | 뜻 |
| --- | --- |
| `docker compose up -d --build` | 이미지 빌드하고 컨테이너들을 백그라운드로 실행. **가장 많이 씀** |
| `docker compose down` | 컨테이너들을 멈추고 삭제 (볼륨=데이터는 유지) |
| `docker compose down -v` | 볼륨까지 삭제 (DB 데이터 다 날아감. 초기화할 때만) |
| `docker compose ps` | 지금 떠 있는 컨테이너 상태 확인 |
| `docker compose logs -f` | 전체 로그를 실시간으로 봄 (`-f`=follow) |
| `docker compose logs -f app` | app 컨테이너 로그만 실시간으로 봄 |
| `docker compose restart app` | app 컨테이너만 재시작 |
| `docker exec -it db mysql -u root -p` | db 컨테이너 안에 들어가서 MySQL 접속 |
| `docker ps` | 실행 중인 모든 컨테이너 보기 |
| `docker images` | 내 컴퓨터에 있는 이미지 목록 |

> `-d`(detached)는 "백그라운드 실행"이라는 뜻. 안 붙이면 터미널이 로그에 붙잡혀서 못 씁니다.
처음엔 `-d` 없이 띄워서 로그를 눈으로 보며 뜨는 과정을 관찰하는 것도 좋은 학습이에요.
> 

---

## Part 5. 실습 순서 요약

```bash
# 1. 로컬에서 먼저 테스트 (서버 올리기 전에 무조건!)
docker compose up -d --build
docker compose logs -f app        # 정상적으로 떴는지 로그 확인
# 브라우저에서 localhost:8080 접속 확인
docker ps # 컨테이너 2개 띄우는 거 확인 -> 스샷 찍어서 PR에 올리기
docker compose down               # 확인 끝나면 정리

# 2. Lightsail 서버에서
git clone <레포주소>
cd <프로젝트폴더>
nano .env                         # 서버에서 .env 직접 작성 (깃엔 없으니까!)
docker compose up -d --build
docker compose logs -f            # 잘 떴는지 확인
# http://서버공인IP:8080 접속!
```

---

## Part 6. 자주 나는 에러 & 해결법 (삽질 방지)

###  `Communications link failure` / `Connection refused`

**원인:** Spring이 MySQL보다 먼저 떠서 접속 실패, 또는 DB 주소를 `localhost`로 씀.
**해결:**

- compose의 URL 호스트가 `localhost`가 아니라 `db`(서비스명)인지 확인
- `depends_on` + `condition: service_healthy` 가 있는지 확인

###  `Public Key Retrieval is not allowed`

**원인:** MySQL 8의 인증 방식 문제.
**해결:** DB URL에 `allowPublicKeyRetrieval=true` 가 들어있는지 확인.

###  한글이 `???` 로 저장됨

**원인:** 문자셋 설정 누락.
**해결:** MySQL `command`의 `utf8mb4` 설정과 URL의 `characterEncoding=UTF-8` 확인.

### 로컬에선 되는데 외부(브라우저)에서 접속이 안 됨

**원인:** Lightsail 방화벽에 8080 포트가 안 열려 있음. (자주 발생!)
**해결:** Lightsail 콘솔 → 인스턴스 → **Networking → IPv4 Firewall** 에서 8080 포트 규칙 추가.

> 이건 서버 안의 ufw(OS 방화벽)와는 **별개인 AWS 레벨 방화벽**이에요. 둘 다 열려 있어야 합니다.
> 

###  빌드 중에 서버가 멈추거나 `Killed` 메시지

**원인:** 메모리 부족(OOM). 작은 Lightsail 인스턴스에서 Gradle 빌드는 메모리를 많이 먹음.
**해결:**

- 인스턴스를 **최소 2GB RAM** 플랜으로 (권장)
- 또는 스왑 메모리 설정:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

###  `permission denied` (gradlew 관련)

**원인:** gradlew에 실행 권한이 없음.
**해결:** Dockerfile에 `RUN chmod +x gradlew` 가 있는지 확인.

###  `docker: command not found` (서버에서)

**원인:** 서버에 Docker 미설치.
**해결:**

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# 이후 재접속(로그아웃 후 재로그인)해야 sudo 없이 docker 사용 가능
```

---

## 오늘의 3줄 요약

1. **Docker = 앱과 환경을 통째로 상자에 담아 어디서든 똑같이 돌리는 도구.** 이미지(틀)로 컨테이너(붕어빵)를 찍어냄.
2. **Compose = 컨테이너 여러 개를 파일 하나로 지휘.** 컨테이너끼리는 서비스 이름(`db`, `app`)으로 서로를 부름. `localhost` 아님!
3. **비밀번호는 `.env`로 분리하고 절대 깃에 안 올림.** 데이터는 볼륨에 저장해서 컨테이너가 죽어도 살림.

수고했어요! 🎉 막히는 부분 있으면 로그부터 (`docker compose logs -f`) 보는 습관!
