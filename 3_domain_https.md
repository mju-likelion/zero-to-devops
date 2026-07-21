# 도메인 연결 & HTTPS 편

> 지난 편에서 우리 앱은 `http://고정IP:8080` 으로 인터넷에 떴습니다. 동작은 하지만 두 가지가 아쉬워요.
> 
> 1. IP랑 포트번호를 외우고 있어야 함 (`43.201.xx.xx:8080`... 누가 이걸 기억함?)
> 2. 브라우저 주소창에 자물쇠가 없음 (`http`는 암호화가 안 돼서 중간에 다 들여다볼 수 있어요)
> 
> 오늘은 이 두 개를 **Caddy**라는 도구 하나로 한 번에 해결합니다. 목표: `https://내도메인.com` 으로 자물쇠까지 딱 뜨게 만들기.
> 

---

## Part 0. 오늘 할 일 지도

```
사용자 브라우저 ──> [도메인 DNS] ──> [고정IP:443] ──> [Caddy] ──> [Spring 컨테이너:8080]
                     (이름표)                          (자동 HTTPS + 문지기)
```

순서는 이렇게 갑니다.

1. 도메인 구매하기
2. 도메인이 우리 서버 IP를 가리키도록 DNS 설정 (A 레코드)
3. 방화벽에 80/443 열기
4. `docker-compose.yml`에 Caddy 추가 + Caddyfile 작성
5. 실행 → 자동으로 인증서 발급됨 → 끝

> Caddy가 하는 일 요약: **Let's Encrypt**라는 무료 인증서 발급 기관에 "이 도메인 저 서버가 맞아요" 자동으로 증명하고, 인증서를 받아서, 갱신까지 알아서 해줍니다. Nginx로 같은 걸 하려면 certbot 설정에 스크립트를 따로 짜야 하는데, Caddy는 **Caddyfile 3줄**이면 끝나요.
> 

---

## Part 1. 도메인 구매하기

도메인은 1년에 몇천 원~1만 원대로 살 수 있습니다. 아래 등록기관 중 편한 곳을 고르세요.

| 등록기관 | 특징 |
| --- | --- |
| **가비아** | 한국어 지원, `.co.kr`/`.kr` 등록 가능, 국내 결제 편함 |
| **Namecheap** | `.com`이 저렴한 편, 해외 카드 필요 |
| **Cloudflare Registrar** | 마진 없이 원가 판매, DNS도 Cloudflare로 통합 관리 가능 (조금 익숙해진 뒤 추천) |

구매가 끝나면 해당 등록기관 콘솔의 **DNS 관리(DNS Zone / 네임서버 설정)** 메뉴로 들어가 Part 2로 이어집니다.

---

## Part 2. DNS에 A 레코드 연결하기

**A 레코드** = "이 도메인은 이 IP다"라고 적어두는 인터넷 전화번호부 등록입니다.

1. 도메인 등록기관 콘솔 → **DNS 관리** 메뉴
2. **레코드 추가** → 아래처럼 입력

| 타입 | 호스트 | 값(Value) | TTL |
| --- | --- | --- | --- |
| A | `@` (또는 빈칸, 루트 도메인용) | 내 고정 IP (`43.201.xx.xx`) | 자동 또는 3600 |
| A | `www` | 내 고정 IP (`43.201.xx.xx`) | 자동 또는 3600 |

> `@`는 `내도메인.com` 자체를, `www`는 `www.내도메인.com`을 뜻해요. 둘 다 같은 IP로 연결해두면 사용자가 www를 붙이든 안 붙이든 접속됩니다.
> 

**반영 확인 (DNS 전파는 몇 분~몇 시간 걸릴 수 있음):**

```bash
# 터미널에서 도메인이 내 IP를 가리키는지 확인
dig +short 내도메인.com

# 결과로 내 고정 IP가 나오면 반영 완료
```

---

## Part 3. Caddy가 뭔데? (30초 요약)

|  | Nginx + Certbot | Caddy |
| --- | --- | --- |
| HTTPS 설정 | 인증서 발급 스크립트 따로 실행, cron으로 갱신 등록 | **설정 파일 3줄이면 자동 발급 + 자동 갱신** |
| 설정 파일 | 문법이 복잡한 편 | Caddyfile, 사람이 읽기 쉬운 문법 |
| 우리 상황 | 처음 배우기엔 손이 많이 감 | **초보자 스터디에 딱 맞음** |

> 실무에서도 Caddy는 많이 쓰이지만, 트래픽이 아주 큰 서비스는 Nginx를 더 흔히 봅니다. "왜 Nginx 말고 Caddy?"라는 질문을 받으면 "자동 HTTPS 때문에 학습 단계에서 골랐고, 내부 동작 원리(리버스 프록시)는 동일하다"고 답하면 됩니다.
> 

---

## Part 4. 방화벽에 80/443 열기

지난 편에서 8080만 열었죠. 이번엔 **웹 표준 포트** 두 개를 엽니다.

- **80번**: 일반 http 접속용 (Caddy가 자동으로 443으로 리다이렉트 처리해줌)
- **443번**: https 접속용 (진짜 암호화 통신)

### ① AWS 레벨 방화벽 (Lightsail 콘솔)

1. Lightsail 콘솔 → 인스턴스 클릭 → **Networking** 탭
2. **IPv4 Firewall** → **Add rule**
3. **HTTP (80)** 규칙 추가
4. **HTTPS (443)** 규칙 추가

> 두 개 다 Lightsail이 미리 정의해둔 규칙 목록에 있어서, 드롭다운에서 고르기만 하면 됩니다. (지난 편 8080은 `Custom`으로 직접 입력했었죠, 차이가 있어요.)
> 

### ② OS 레벨 방화벽 (ufw, 켜져 있다면)

```bash
sudo ufw status          # inactive면 무시해도 됨
# active라면:
sudo ufw allow 80
sudo ufw allow 443
```

### ③ 기존 8080 포트는 이제 막아도 됨 (선택, 권장)

Caddy가 앞단에서 다 처리하기 때문에, 이제 8080을 외부에 직접 노출할 필요가 없어요. Lightsail 콘솔에서 8080 규칙을 **삭제**하면 앱이 오직 `https://내도메인.com` 으로만 접속 가능해집니다 (더 안전함).

---

## Part 5. Caddy를 docker-compose에 추가하기

### ① Caddyfile 만들기

프로젝트 폴더(서버에서 `cd 내프로젝트` 한 그 위치)에 `Caddyfile`이라는 이름으로 파일을 만듭니다. (확장자 없음!)

```bash
nano Caddyfile
```

내용 입력 (도메인 자리에 본인 도메인):

```
내도메인.com, www.내도메인.com {
    reverse_proxy app:8080
}
```

> `reverse_proxy app:8080` 한 줄이 하는 일: "여기로 들어온 요청을 도커 내부 네트워크의 `app` 컨테이너, 8080 포트로 그대로 넘겨줘." `app`은 지난 편 `docker-compose.yml`에서 Spring 서비스 이름이에요 (본인 파일에서 이름이 다르면 그 이름으로 맞춰주세요).
> 

저장: `Ctrl + O` → `Enter` → `Ctrl + X`

### ② docker-compose.yml 수정하기

기존 파일을 열어서:

```bash
nano docker-compose.yml
```

**`app` 서비스에서 `ports` 항목을 지우거나 주석 처리**합니다 (더 이상 8080을 직접 외부에 열 필요 없음, Caddy를 거쳐가야 하니까):

```yaml
services:
  app:
    build: .
    # ports:
    #   - "8080:8080"      # ← Caddy를 쓰므로 더 이상 외부 노출 불필요
    depends_on:
      - db
    env_file:
      - .env

  db:
    image: mysql:8
    # ... (지난 편 내용 그대로)

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - app

volumes:
  db_data:
  caddy_data:
  caddy_config:
```

> 볼륨 경로 `./Caddyfile:/etc/caddy/Caddyfile` 오타 주의하세요. `caddy_data`는 발급받은 인증서를 저장하는 곳이라, 여기가 날아가면 재배포할 때마다 인증서를 새로 받게 됩니다 (Let's Encrypt는 짧은 시간에 너무 자주 요청하면 잠시 막아버려요). 볼륨으로 반드시 유지하세요.
> 

---

## Part 6. 실행하고 확인하기

```bash
# 캐디 포함해서 다시 빌드 & 실행
docker compose up -d --build

# 캐디가 인증서를 잘 받았는지 로그로 확인
docker compose logs -f caddy
```

로그에 이런 느낌의 메시지가 보이면 성공입니다.

```
certificate obtained successfully
serving initial configuration
```

(`Ctrl + C`로 로그 보기 종료)

### 브라우저에서 확인

```
https://내도메인.com
```

주소창에  자물쇠가 뜨고, 지난 편의 그 앱 화면이 나오면 **완성**입니다. `http://내도메인.com`으로 들어가도 Caddy가 자동으로 `https://`로 리다이렉트시켜줘요.

---

## Part 7. 자주 나는 에러 & 해결법

### Caddy 로그에 인증서 발급 실패 (timeout, challenge failed 등)

**가장 흔한 원인 2가지, 순서대로 확인:**

1. DNS가 아직 우리 서버를 안 가리킴 → `dig +short 내도메인.com`으로 재확인, 시간을 더 기다려보기
2. 80/443 포트가 안 열림 → Part 4의 ①번(AWS 방화벽) 다시 확인

> Let's Encrypt는 80번 포트로 "진짜 이 도메인 주인이 맞는지" 확인하는 절차를 거칩니다. 80번이 막혀 있으면 이 확인 자체가 실패해요.
> 

### `www.내도메인.com`은 안 되고 `내도메인.com`만 됨 (또는 반대)

**원인:** DNS에 A 레코드를 하나만 등록함.
**해결:** Part 2에서 `@`랑 `www` 둘 다 등록했는지 확인.

### Caddyfile 수정했는데 반영이 안 됨

**원인:** 컨테이너를 재시작 안 함.
**해결:**

```bash
docker compose restart caddy
```

### `Let's Encrypt rate limit` 관련 에러

**원인:** 같은 도메인으로 짧은 시간 내 인증서를 너무 여러 번 요청함 (테스트하다가 컨테이너를 계속 지웠다 새로 만들면 발생하기 쉬움).
**해결:** `caddy_data` 볼륨을 지우지 않기 (Part 5의 경고 참고). 급하면 몇 시간 기다렸다 재시도.

### DNS는 맞게 넣은 것 같은데 도메인 접속이 안 됨

**원인:** 보통 DNS 전파 지연이거나 A 레코드 값 오타.
**해결:** `dig +short 내도메인.com` 결과와 Lightsail 고정 IP가 정확히 일치하는지 한 글자씩 비교.

### 예전 8080 주소로 접속하면 아직도 앱이 뜸

**원인:** Part 4의 ③(8080 규칙 삭제)을 안 함, 또는 `docker-compose.yml`의 `ports` 주석 처리를 안 함.
**해결:** 둘 다 확인. (보안상 막아두는 걸 권장하지만, 안 막아도 서비스 자체는 정상 동작합니다.)

---

## 오늘의 3줄 요약

1. **도메인을 구매하고(가비아/Namecheap 등) DNS에 A 레코드로 고정 IP를 등록한다.** `@`와 `www` 둘 다 등록하는 걸 잊지 말 것.
2. **Caddy = Caddyfile 3줄로 리버스 프록시 + 자동 HTTPS.** `docker-compose.yml`에 caddy 서비스 추가하고, 기존 앱의 8080 외부 노출은 없앤다.
3. **방화벽에 80/443 열기(AWS + ufw 둘 다), `docker compose up -d --build`.** 로그에 `certificate obtained successfully` 뜨면 `https://내도메인.com` 완성.

이제 여러분 앱은 **IP도, 포트번호도, 자물쇠 없는 경고도 없이** 진짜 서비스 주소로 접속됩니다. 

---

### 다음 편 예고

지금까지는 코드 고칠 때마다 서버에 직접 SSH로 들어가서 `git pull` + `docker compose up -d --build`를 손으로 쳤죠. 다음 편에서는 **GitHub Actions로 CI/CD를 구성**해서, `main` 브랜치에 push만 하면 서버 배포까지 자동으로 끝나게 만들어봅니다.
