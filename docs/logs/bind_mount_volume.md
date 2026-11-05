# 바인드 마운트와 볼륨 검증 로그

터미널: WSL2 Ubuntu, bash. 작업 폴더: `~/codyssey/bind-test`

## 1. 바인드 마운트: 호스트 수정이 바로 반영되는지

```
$ mkdir -p ~/codyssey/bind-test/site
$ cd ~/codyssey/bind-test
$ echo '<h1>Bind mount v1</h1>' > site/index.html

$ docker run -d --name bind-web -p 8082:80 -v "$(pwd)/site:/usr/share/nginx/html" nginx:alpine
Unable to find image 'nginx:alpine' locally
alpine: Pulling from library/nginx
Status: Downloaded newer image for nginx:alpine
785767f2038cb649ca2822a7fbbe02a2b3c78fefc97095f7b47bfd975ca7aed5
```

변경 전:

```
$ curl -s http://localhost:8082
<h1>Bind mount v1</h1>
```

호스트(내 PC)에서 파일만 수정한다. 컨테이너는 재시작하지 않고, 이미지도 다시 빌드하지 않는다.

```
$ echo '<h1>Bind mount v2 - edited on host</h1>' > site/index.html
```

변경 후:

```
$ curl -s http://localhost:8082
<h1>Bind mount v2 - edited on host</h1>

$ docker exec bind-web cat /usr/share/nginx/html/index.html
<h1>Bind mount v2 - edited on host</h1>

$ docker inspect -f '{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{end}}' bind-web
bind /home/qphanmj/codyssey/bind-test/site -> /usr/share/nginx/html
```

결론: 호스트 폴더가 컨테이너 폴더에 직접 연결되어 있어서, 호스트에서 고친 내용이 컨테이너에 즉시 반영된다.

## 2. 볼륨: 컨테이너를 삭제해도 데이터가 남는지

```
$ docker volume create mydata
mydata

$ docker volume ls
DRIVER    VOLUME NAME
local     mydata

$ docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
004e94ac6c22d0ff8dd979e236c7a208e7f2311d4d618d99ecf6920771e9fe1d

$ docker exec vol-test bash -c "echo hi > /data/hello.txt && cat /data/hello.txt"
hi
```

컨테이너 삭제:

```
$ docker rm -f vol-test
vol-test

$ docker ps -a --filter name=vol-test
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

새 컨테이너에 같은 볼륨 연결:

```
$ docker run -d --name vol-test2 -v mydata:/data ubuntu sleep infinity
e0df9d4c6cac42f1a65a550fb87ea3167ebdb3c3ca228fd52417bd2ce9db06ab

$ docker exec vol-test2 cat /data/hello.txt
hi

$ docker volume inspect mydata
[
    {
        "CreatedAt": "2026-11-04T15:01:01Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
        "Name": "mydata",
        "Options": null,
        "Scope": "local"
    }
]
```

결론: 컨테이너(`vol-test`)를 삭제해도 볼륨(`mydata`)에 저장한 `hello.txt`는 남아 있고, 새 컨테이너(`vol-test2`)에서 그대로 읽힌다.

## 3. 비교: 볼륨이 아닌 경로에 쓴 파일

```
$ docker run -d --name novol-test ubuntu sleep infinity
$ docker exec novol-test bash -c "echo hi > /tmp/hello.txt && cat /tmp/hello.txt"
hi

$ docker rm -f novol-test
novol-test

$ docker run -d --name novol-test2 ubuntu sleep infinity
$ docker exec novol-test2 cat /tmp/hello.txt
cat: /tmp/hello.txt: No such file or directory
```

결론: 볼륨에 연결하지 않은 경로(`/tmp`)에 쓴 파일은 컨테이너와 함께 사라진다.
