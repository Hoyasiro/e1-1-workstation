# 커스텀 이미지 빌드와 포트 매핑 로그

기록 시각: 2026-11-04 (KST). 터미널: WSL2 Ubuntu, bash.

## 1. 프로젝트 구조

```
~/codyssey/my-web/
├── Dockerfile
└── site/
    └── index.html
```

## 2. Dockerfile

```dockerfile
FROM nginx:alpine
LABEL org.opencontainers.image.title="my-custom-nginx"
ENV APP_ENV=dev
COPY site/ /usr/share/nginx/html/
```

| 줄 | 커스텀 포인트의 목적 |
|---|---|
| `FROM nginx:alpine` | 가벼운 nginx 공식 이미지를 베이스로 선택 |
| `LABEL ...` | 이미지에 제목 메타데이터를 붙여 식별 |
| `ENV APP_ENV=dev` | 실행 환경을 환경 변수로 표시 |
| `COPY site/ ...` | nginx 기본 페이지를 내 정적 페이지로 교체 |

## 3. 빌드

```
$ docker build -t my-web:1.0 .
...
 => [2/2] COPY site/ /usr/share/nginx/html/
 => => naming to docker.io/library/my-web:1.0

$ docker images my-web
IMAGE        ID             DISK USAGE   CONTENT SIZE   EXTRA
my-web:1.0   f2b44703f341       93.6MB         26.3MB   U
```

## 4. 포트 충돌 (트러블슈팅)

8080 포트를 쓰는 `web-test` 컨테이너가 실행 중인 상태에서 같은 포트로 실행했다.

```
$ docker run -d -p 8080:80 --name my-web-8080 my-web:1.0
docker: Error response from daemon: failed to set up container networking: ...
Bind for 0.0.0.0:8080 failed: port is already allocated
```

해결: `docker rm -f my-web-8080 web-test`로 실패한 컨테이너와 8080을 쓰던 컨테이너를 지우고 다시 실행했다.

## 5. 두 포트로 실행 (8080, 8081)

```
$ docker run -d -p 8080:80 --name my-web-8080 my-web:1.0
$ docker run -d -p 8081:80 --name my-web-8081 my-web:1.0

$ docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                                     NAMES
be4764268059   my-web:1.0   "/docker-entrypoint.…"   53 seconds ago   Up 52 seconds   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   my-web-8081
d22a2fd66daa   my-web:1.0   "/docker-entrypoint.…"   57 seconds ago   Up 55 seconds   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   my-web-8080
```

## 6. 접속 확인

```
$ curl -s http://localhost:8080
<h1>Hello from my custom nginx!</h1>

$ curl -s http://localhost:8081
<h1>Hello from my custom nginx!</h1>
```

이미지 하나(`my-web:1.0`)로 컨테이너 두 개를 만들었고, 두 컨테이너 모두 내부에서는 80번을 쓰지만 호스트 포트(8080, 8081)가 달라 충돌 없이 실행된다.

## 아직 남은 증거

- [ ] 브라우저에서 `http://localhost:8080`, `http://localhost:8081` 접속 화면 캡처 (주소창 포함)
