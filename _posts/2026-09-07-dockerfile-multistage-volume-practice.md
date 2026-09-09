---
layout: post
title: "Docker 실습 정리: Dockerfile부터 이미지 레이어와 OverlayFS까지"
date: 2026-09-07 18:00:00 +0900
last_modified_at: 2026-09-09 12:00:00 +0900
category: [backend-cloud]
tags: [Docker, Dockerfile, MariaDB, Volume, Bind-Mount, Nginx, Multi-Stage-Build, ENTRYPOINT, CMD, PID-1, SIGTERM, OverlayFS, rootfs, learning-note]
---

Docker를 처음 접했을 때는 이미지와 컨테이너가 단순히 “프로그램을 격리해서 실행하는 기능” 정도로만 보였다. 이번에는 명령을 직접 실행하면서 Dockerfile, 볼륨, PID 1, 이미지 레이어와 OverlayFS가 실제로 어떻게 이어지는지 확인했다.

이 글은 개념만 나열하지 않고 **무엇을 했는지 → 왜 하는지 → 실행 명령 → 확인 결과 → Docker 원리와의 연결** 순서로 정리한 실습 기록이다.

## 1. MariaDB 컨테이너 실행과 데이터 확인

가장 먼저 MariaDB를 컨테이너로 실행했다. 애플리케이션과 데이터베이스 컨테이너가 이름으로 통신할 수 있도록 네트워크도 만들었다.

```bash
docker network create skala
docker pull mariadb:latest

docker run -d \
  --name mariadb \
  --network skala \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=skala \
  mariadb:latest
```

컨테이너 안의 MariaDB에 접속해 데이터베이스와 테이블을 만들고 데이터를 조회했다.

```bash
docker exec -it mariadb mariadb -uroot -ppassword
```

> `password`는 로컬 실습용 예시값이다. 실제 서비스에서는 강한 비밀번호와 별도의 비밀 관리 방식을 사용해야 한다.

```sql
SHOW DATABASES;
USE skala;

CREATE TABLE key_value (
  key_name VARCHAR(50) PRIMARY KEY,
  value_text VARCHAR(100)
);

INSERT INTO key_value VALUES ('hello', 'docker');
SELECT * FROM key_value;
```

DBeaver에서도 `localhost:3306`으로 접속해 같은 데이터를 확인했다. 이 실습을 통해 컨테이너 안에서 실행되는 데이터베이스도 포트를 공개하면 호스트의 일반 프로그램에서 접근할 수 있다는 것을 확인했다.

## 2. 볼륨과 Bind Mount로 데이터 연결

컨테이너는 삭제할 수 있는 실행 단위이므로 중요한 데이터를 컨테이너 내부에만 두면 곤란하다. 호스트의 디렉터리와 컨테이너의 디렉터리를 연결해 파일이 어떻게 공유되는지 확인했다.

```bash
mkdir -p data
echo "hello volume" > data/info.txt

docker run --rm \
  -v "$(pwd)/data:/data" \
  ubuntu:22.04 \
  cat /data/info.txt
```

컨테이너에서 `/data/info.txt`를 읽었지만 실제 파일은 호스트의 `./data`에 있었다. 즉, 볼륨은 특정 파일 형식이 아니라 **저장 공간을 컨테이너와 연결하는 방법**이다.

- 이미지: 애플리케이션 실행에 필요한 읽기 전용 설계도
- 컨테이너: 이미지를 실행한 인스턴스
- 볼륨/Bind Mount: 컨테이너 밖에 데이터를 보관하거나 공유하는 저장 공간

## 3. Dockerfile로 Ubuntu 실행 환경 만들기

다음 Dockerfile을 빌드해 Ubuntu 22.04 기반 이미지를 만들었다.

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl lsb-release
```

```bash
docker buildx build --tag linux-container:1.0 .
docker run -it --name linux-container linux-container:1.0 /bin/bash
lsb_release -a
```

결과에서 `Ubuntu 22.04 LTS`를 확인했다. Dockerfile은 컨테이너 자체가 아니라 **이미지를 만드는 절차를 기록한 파일**이라는 점을 이해할 수 있었다.

## 4. ARG, LABEL, EXPOSE, WORKDIR, COPY

같은 Dockerfile을 조금씩 변경하며 주요 명령을 확인했다.

```dockerfile
ARG UBUNTU_VERSION=22.04
FROM ubuntu:${UBUNTU_VERSION}

RUN apt-get update && apt-get install -y curl lsb-release nginx

ARG UBUNTU_VERSION
RUN echo "Ubuntu version: ${UBUNTU_VERSION}"

LABEL maintainer="maintainer@example.com"
LABEL description="Linux practice image"

EXPOSE 80/tcp
WORKDIR /var/www/html
COPY index.html .

CMD ["nginx", "-g", "daemon off;"]
```

`ARG` 값을 빌드 시점에 바꿔 Ubuntu 24.04 이미지도 만들었다.

```bash
docker build --tag linux-container:1.0 \
  --build-arg UBUNTU_VERSION=24.04 .
```

![ARG로 Ubuntu 24.04를 적용한 결과]({{ '/assets/images/dockerfile-practice/02-arg-ubuntu-24-clean.png' | relative_url }})

각 명령의 역할은 다음과 같다.

- `ARG`: 이미지 빌드 중에만 사용하는 변수
- `LABEL`: 설명과 작성자 같은 이미지 메타데이터
- `EXPOSE`: 이미지가 사용할 포트를 기록하는 메타데이터
- `WORKDIR`: 이후 명령의 기준 디렉터리
- `COPY`: 빌드 컨텍스트의 파일을 이미지 안으로 복사
- `CMD`: 컨테이너 시작 시 실행할 기본 명령

`EXPOSE`만으로는 호스트와 포트가 연결되지 않는다. 실제 연결은 실행할 때 `-p 8888:80`과 같이 지정해야 한다.

```bash
docker run -d --name linux-container -p 8888:80 linux-container:1.0
```

직접 만든 HTML이 Nginx를 통해 표시되는 것도 확인했다.

![사용자 정의 Nginx 페이지]({{ '/assets/images/dockerfile-practice/03-custom-nginx-page.png' | relative_url }})

## 5. 일반 사용자로 Nginx 실행하며 권한 이해하기

컨테이너를 root가 아닌 `skala` 사용자로 실행해 보았다. 단순히 `USER skala`만 추가하자 Nginx가 필요한 디렉터리를 사용할 수 없어 `Permission denied` 오류가 발생했다.

```text
Permission denied: /var/log/nginx/error.log
Permission denied: /var/lib/nginx/body
```

필요한 디렉터리의 소유권과 포트를 조정한 뒤 정상 실행할 수 있었다.

```dockerfile
RUN chown -R skala:skala /var/lib/nginx /var/log/nginx /run
USER skala
```

여기서 배운 점은 `USER`만 바꾸면 보안 설정이 끝나는 것이 아니라는 점이다. 애플리케이션이 사용하는 파일, 디렉터리와 포트 권한까지 함께 맞아야 한다.

## 6. ENTRYPOINT와 CMD 비교

Alpine 이미지에서 `ping`을 실행하며 두 명령의 역할을 비교했다.

```dockerfile
FROM alpine:latest
ENTRYPOINT ["ping"]
CMD ["-c", "3", "localhost"]
```

```bash
docker build -f Dockerfile.entrypoint -t ping-container:1.0 .
docker run --rm ping-container:1.0
docker run --rm ping-container:1.0 -c 3 google.com
docker run --rm --entrypoint echo ping-container:1.0 "ENTRYPOINT 교체 성공"
```

![ENTRYPOINT와 CMD 실행 결과]({{ '/assets/images/dockerfile-practice/04-entrypoint-cmd.png' | relative_url }})

- `ENTRYPOINT`: 컨테이너의 핵심 실행 프로그램
- `CMD`: 기본 명령 또는 `ENTRYPOINT`에 전달할 기본 인자
- `docker run` 뒤의 값: 기본 `CMD` 대체
- `--entrypoint`: `ENTRYPOINT` 자체 대체

## 7. PID 1과 SIGTERM

컨테이너 안에서 처음 실행되는 프로세스는 PID 1이 된다. Docker는 `docker stop`을 실행하면 먼저 PID 1에 `SIGTERM`을 보내 정상 종료할 기회를 준다.

세 가지 `CMD`를 비교했다.

```dockerfile
# Exec Form: Python이 직접 PID 1
CMD ["python3", "webserver.py"]
```

```dockerfile
# Shell Form: shell이 PID 1, Python은 자식 프로세스
CMD ["/bin/sh", "-c", "python3 -u webserver.py"]
```

```dockerfile
# exec 사용: shell이 Python으로 치환되어 Python이 PID 1
CMD ["/bin/sh", "-c", "exec python3 -u webserver.py"]
```

```bash
docker exec linux-container ps -ef
docker stop linux-container
docker logs linux-container
```

Exec Form과 `exec`를 사용한 방식에서는 Python이 PID 1이었고, 종료 로그에서 `SIGTERM 신호 수신`과 처리 완료를 확인했다. 반면 단순 Shell Form은 shell이 PID 1이어서 신호가 애플리케이션까지 올바르게 전달되지 않을 수 있었다.

이 차이는 서버가 종료 전에 진행 중인 요청을 마치고 연결과 자원을 정리하는 **Graceful Shutdown**과 연결된다.

## 8. Vue Multi-Stage Build

Vue 프로젝트는 Node.js로 빌드해야 하지만, 빌드 결과인 정적 파일을 서비스할 때 Node.js 전체가 필요한 것은 아니다. 그래서 빌드 단계와 실행 단계를 분리했다.

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist/ /usr/share/nginx/html/
```

```bash
docker build -t vue-frontend:latest .
docker run -d --name vue-frontend -p 8080:80 vue-frontend:latest
```

![Vue Multi-Stage 빌드 결과]({{ '/assets/images/dockerfile-practice/01-vue-multistage-clean.png' | relative_url }})

흐름은 다음과 같다.

```text
Vue 소스
  → Node.js 단계에서 빌드
  → dist 생성
  → Nginx 단계로 dist만 복사
  → 최종 웹서비스 이미지 완성
```

Multi-Stage Build를 사용하면 빌드 도구와 원본 소스를 최종 이미지에서 제외할 수 있어 이미지가 단순해지고 역할도 명확해진다.

## 9. Python 웹서버와 Bind Mount

Python 웹서버를 실행하는 이미지를 만들고 호스트의 `mydata`와 컨테이너의 `/mydata`를 연결했다.

```bash
docker build -f Dockerfile.python -t linux-container:1.0 .
mkdir -p mydata

docker run -d \
  --name linux-container \
  -p 8080:8080 \
  -v "$(pwd)/mydata:/mydata" \
  linux-container:1.0
```

호스트에서 `mydata`에 넣은 `webserver.py`가 컨테이너의 `/mydata`에서도 보였고, 브라우저에서 `Welcome to SKALA World` 화면이 표시됐다.

![볼륨을 연결한 Python 웹서버]({{ '/assets/images/dockerfile-practice/05-volume-webserver.png' | relative_url }})

Python은 Docker에 꼭 필요한 도구라서 설치한 것이 아니다. 이번 실습에서 동적인 웹서버 프로세스와 종료 신호를 확인하기 위한 예제 애플리케이션이었다. 정적 HTML만 제공한다면 Nginx만으로도 충분하다.

## 10. Docker 이미지의 내부 구조 확인

이번에는 이미지를 실행하지 않고 파일로 저장한 뒤 내부를 직접 열어 보았다.

```bash
docker image history indepth-container:1.0
docker save indepth-container:1.0 -o indepth-container.tar

mkdir extracted
tar xvf indepth-container.tar -C extracted
file extracted/*
python3 -m json.tool extracted/manifest.json
```

`manifest.json`, `index.json`, `oci-layout`, `blobs/sha256` 구조를 확인했다. 이미지가 실행 파일 하나가 아니라 메타데이터와 여러 파일시스템 레이어로 구성되어 있다는 의미다.

MariaDB 이미지도 같은 방법으로 살펴봤다.

```bash
docker pull mariadb:10.11
docker save mariadb:10.11 -o mariadb.tar
tar xvf mariadb.tar -C extracted
python3 -m json.tool extracted/manifest.json
```

`manifest.json`의 `Layers` 배열에서 MariaDB 이미지가 8개의 파일시스템 레이어로 구성된 것을 확인했다. 첫 번째 레이어를 풀자 다음과 같은 리눅스 rootfs 디렉터리가 나타났다.

```text
bin  boot  dev  etc  home  lib  media  mnt
opt  proc  root run  sbin  srv  sys   tmp  usr  var
```

즉, 컨테이너 안에서 보이는 리눅스 파일시스템은 갑자기 만들어지는 것이 아니라 이미지의 여러 레이어를 합쳐 구성된다.

## 11. OverlayFS로 합쳐진 rootfs 만들기

macOS의 Docker Desktop은 내부 Linux VM에서 컨테이너를 실행한다. 따라서 OverlayFS 마운트를 직접 확인하기 위해 권한이 있는 Ubuntu 컨테이너를 사용했다.

```bash
docker run --rm -it --privileged ubuntu:24.04 /bin/bash
```

실습용 공간과 네 종류의 디렉터리를 만들었다.

```bash
mkdir -p /mnt/ovtest
mount -t tmpfs tmpfs /mnt/ovtest
mkdir -p /mnt/ovtest/{lower,upper,work,merged}
```

- `lower`: 이미지 원본에 해당하는 읽기 전용 레이어
- `upper`: 컨테이너에서 발생한 변경사항
- `work`: OverlayFS가 내부 작업에 사용하는 공간
- `merged`: lower와 upper가 합쳐져 사용자에게 보이는 최종 화면

원본 역할을 하는 파일을 만들고 OverlayFS로 합쳤다.

```bash
echo "AAA from lower" > /mnt/ovtest/lower/a.txt
echo "BBB from lower" > /mnt/ovtest/lower/b.txt

mount -t overlay overlay \
  -o lowerdir=/mnt/ovtest/lower,upperdir=/mnt/ovtest/upper,workdir=/mnt/ovtest/work \
  /mnt/ovtest/merged
```

초기에는 `lower`에 `a.txt`, `b.txt`가 있고 `upper`는 비어 있었다. 그러나 `merged`에서는 두 파일이 모두 보였다. `merged`는 파일을 복사한 별도 폴더가 아니라 여러 계층을 합쳐 보여주는 화면이다.

## 12. Copy-on-Write 확인

`merged`를 통해 기존 파일을 수정했다.

```bash
echo "AAA modified" > /mnt/ovtest/merged/a.txt

cat /mnt/ovtest/lower/a.txt
cat /mnt/ovtest/upper/a.txt
cat /mnt/ovtest/merged/a.txt
```

결과는 다음과 같았다.

```text
lower/a.txt  → AAA from lower
upper/a.txt  → AAA modified
merged/a.txt → AAA modified
```

원본 `lower/a.txt`는 그대로였고, 수정된 파일이 `upper`에 생겼다. 읽기 전용 원본을 바로 수정하지 않고 쓰기가 발생할 때 upper로 복사해 변경하는 방식이 **Copy-on-Write**다.

## 13. 새 파일과 Whiteout 확인

새 파일을 `merged`에 만들면 곧바로 `upper`에 저장됐다.

```bash
echo "CCC new file" > /mnt/ovtest/merged/c.txt
```

```text
lower → a.txt, b.txt
upper → a.txt, c.txt
merged → a.txt, b.txt, c.txt
```

이번에는 기존 `b.txt`를 삭제했다.

```bash
rm /mnt/ovtest/merged/b.txt
ls -l /mnt/ovtest/lower
ls -la /mnt/ovtest/upper
ls -l /mnt/ovtest/merged
```

`lower`의 원본 `b.txt`는 그대로였지만 `upper`에는 삭제를 표시하는 특수 엔트리가 생성됐고, `merged`에서는 `b.txt`가 보이지 않았다. 이 삭제 표시를 **Whiteout**이라고 한다.

```text
수정 → Copy-on-Write로 upper에 수정본 생성
추가 → upper에 새 파일 생성
삭제 → upper에 Whiteout 생성
원본 → lower에 그대로 유지
```

## 14. 이미지, 컨테이너와 OverlayFS 연결하기

마지막으로 지금까지 확인한 내용을 Docker의 구조와 연결했다.

```text
Dockerfile
  → docker build
  → 여러 읽기 전용 Layer로 구성된 Image
  → docker run
  → 컨테이너별 쓰기 Layer 추가
  → OverlayFS가 레이어들을 하나의 rootfs처럼 표시
  → 그 rootfs에서 Container Process 실행
```

OverlayFS 관점에서는 다음과 같이 대응한다.

- Docker 이미지의 읽기 전용 레이어 → `lower`
- 컨테이너에서 수정·추가·삭제한 내용 → `upper`
- 컨테이너가 실제로 바라보는 `/` → `merged`

같은 이미지로 컨테이너 A와 B를 실행하더라도 각 컨테이너는 자신의 쓰기 레이어를 가진다. 따라서 한 컨테이너에서 변경한 내용이 다른 컨테이너의 파일시스템에 바로 반영되지 않는다.

## 실습 중 해결한 문제

### 컨테이너 이름 충돌

같은 이름의 컨테이너가 남아 있으면 새 컨테이너를 실행할 수 없었다.

```bash
docker ps -a --filter name=linux-container
docker rm -f linux-container
```

### 빌드 실패 후 이전 이미지가 실행된 문제

이미지 빌드가 실패해도 같은 태그의 이전 이미지가 남아 있을 수 있다. 이어서 `docker run`을 실행하면 이전 이미지가 실행되어 변경 내용이 적용되지 않은 것처럼 보인다. 빌드 출력의 `FINISHED`를 확인한 뒤 실행해야 한다.

### Dockerfile 내용 중복

편집 중 Dockerfile 내용이 중복되면서 두 번째 `FROM`이 별도 빌드 단계로 인식됐다. 앞 단계의 `EXPOSE`가 최종 이미지에 포함되지 않아 검사 결과가 `null`로 표시됐다.

```bash
nl -ba Dockerfile
```

줄 번호와 함께 내용을 확인해 중복 부분을 제거했다.

### 일반 사용자 실행 시 Permission denied

`USER`를 일반 사용자로 바꿨지만 Nginx가 필요한 디렉터리에 접근할 수 없었다. 로그를 통해 실패 위치를 찾고 소유권과 포트 설정을 수정했다.

```bash
docker ps -a
docker logs linux-container
```

## 마무리

이번 실습 전에는 이미지와 컨테이너를 완성된 가상환경 정도로 생각했다. 하지만 내부 구조를 직접 풀고 OverlayFS를 구성하면서 다음 흐름을 구체적으로 이해할 수 있었다.

- Dockerfile은 이미지를 만드는 절차다.
- 이미지는 메타데이터와 여러 읽기 전용 파일시스템 레이어로 구성된다.
- 컨테이너는 이미지 위에 자신만의 쓰기 레이어를 추가한다.
- OverlayFS가 이 레이어들을 하나의 rootfs처럼 보여준다.
- 컨테이너의 핵심은 그 파일시스템 안에서 실행되는 프로세스다.
- PID 1과 신호 전달 방식은 애플리케이션의 정상 종료에 영향을 준다.
- 영구 데이터는 컨테이너의 쓰기 레이어가 아닌 볼륨에 보관해야 한다.
- Multi-Stage Build는 빌드 환경과 실행 환경을 분리해 최종 이미지를 단순하게 만든다.

결국 Docker의 전체 흐름은 **Dockerfile → Image Layer → Container Layer → rootfs → Process**로 연결된다. 명령의 사용법뿐 아니라 각 단계가 왜 필요한지를 직접 결과로 확인한 실습이었다.
