---
layout: post
title: "Docker 기초 실습: Dockerfile부터 Multi-Stage Build와 볼륨까지"
date: 2026-09-07 18:00:00 +0900
category: [cloud, docker]
tags: [Docker, Dockerfile, Multi-Stage-Build, Nginx, Volume, ENTRYPOINT, CMD, learning-note]
---

오늘은 Dockerfile의 주요 명령어를 직접 사용해 이미지를 빌드하고 컨테이너를 실행했다. 이미지 빌드뿐 아니라 포트 연결, 일반 사용자 권한, `ENTRYPOINT`와 `CMD`, 볼륨 연결까지 단계별로 확인했다.

## 1. Vue 프로젝트 Multi-Stage Build

Vue 프로젝트는 Node.js 환경에서 빌드한 뒤, 생성된 `dist` 파일만 Nginx 이미지로 옮기는 Multi-Stage 방식으로 구성했다.

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

Multi-Stage Build를 사용하면 빌드 도구와 소스 전체를 최종 이미지에 넣지 않고 실행에 필요한 결과물만 담을 수 있다.

## 2. ARG로 Ubuntu 버전 지정

`ARG`를 사용해 빌드할 Ubuntu 버전을 외부에서 전달했다.

```dockerfile
ARG UBUNTU_VERSION=22.04
FROM ubuntu:${UBUNTU_VERSION}

RUN apt-get update && apt-get install -y curl lsb-release

ARG UBUNTU_VERSION
RUN echo "현재 빌드에 사용된 ubuntu version: ${UBUNTU_VERSION}"
```

```bash
docker build --tag linux-container:1.0 \
  --build-arg UBUNTU_VERSION=24.04 .
```

컨테이너에서 `lsb_release -a`를 실행해 Ubuntu 24.04가 적용된 것을 확인했다.

![ARG로 Ubuntu 24.04를 적용한 결과]({{ '/assets/images/dockerfile-practice/02-arg-ubuntu-24-clean.png' | relative_url }})

## 3. 이미지 메타데이터와 포트

`LABEL`로 이미지 설명을 등록하고 `EXPOSE`로 이미지가 사용할 포트를 기록했다. 공개 글에서는 실제 개인 이메일 대신 예시 주소를 사용했다.

```dockerfile
LABEL maintainer="maintainer@example.com"
LABEL description="SKALA Linux Version"

EXPOSE 8080
EXPOSE 80/tcp
```

```bash
docker image inspect --format='{{json .Config.Labels}}' linux-container:1.0
docker image inspect --format='{{json .Config.ExposedPorts}}' linux-container:1.0
```

`EXPOSE`는 이미지의 포트 정보를 기록할 뿐 실제 호스트 포트를 열지는 않는다. 실제 연결은 `docker run`의 `-p 8888:80` 같은 옵션이 담당한다.

## 4. WORKDIR와 COPY로 페이지 교체

Nginx 기본 페이지를 준비한 `index.html`로 교체했다.

```dockerfile
WORKDIR /var/www/html
COPY index.html .

CMD ["nginx", "-g", "daemon off;"]
```

브라우저에서 직접 만든 SKALA Container 페이지가 표시되는 것을 확인했다.

![사용자 정의 Nginx 페이지]({{ '/assets/images/dockerfile-practice/03-custom-nginx-page.png' | relative_url }})

## 5. 일반 사용자로 Nginx 실행

컨테이너를 root가 아닌 `skala` 사용자로 실행해 보았다. 처음에는 Nginx가 사용하는 디렉터리에 쓰기 권한이 없어 다음 오류가 발생했다.

```text
Permission denied: /var/log/nginx/error.log
Permission denied: /var/lib/nginx/body
```

필요한 디렉터리의 소유자를 변경하고 Nginx 리스닝 포트를 8080으로 바꿨다.

```dockerfile
RUN chown -R skala:skala /var/lib/nginx /var/log/nginx /run
USER skala
```

이 과정을 통해 `USER`만 변경해서는 충분하지 않고 애플리케이션이 사용하는 파일과 포트 권한도 함께 고려해야 한다는 것을 확인했다.

## 6. ENTRYPOINT와 CMD 비교

다음 Dockerfile로 두 명령의 역할을 비교했다.

```dockerfile
FROM alpine:latest
ENTRYPOINT ["ping"]
CMD ["-c", "3", "localhost"]
```

기본 실행에서는 `ping -c 3 localhost`가 실행됐다. 실행 시 추가 인자를 전달하자 기본 `CMD`가 교체됐다.

```bash
docker run --rm ping-container:1.0 -c 3 google.com
docker run --rm --entrypoint echo ping-container:1.0 "ENTRYPOINT 교체 성공"
```

![ENTRYPOINT와 CMD 실행 결과]({{ '/assets/images/dockerfile-practice/04-entrypoint-cmd.png' | relative_url }})

- `ENTRYPOINT`: 컨테이너가 실행할 핵심 프로그램
- `CMD`: 핵심 프로그램에 전달할 기본 인자
- `docker run` 뒤의 인자: Dockerfile의 `CMD` 대체
- `--entrypoint`: Dockerfile의 `ENTRYPOINT` 대체

## 7. 볼륨을 연결한 Python 웹서버

Python 웹서버 이미지를 만든 뒤 호스트의 `mydata` 폴더를 컨테이너의 `/mydata`에 연결했다.

```bash
docker build -f Dockerfile.python -t linux-container:1.0 .
mkdir -p mydata

docker run -d \
  --name linux-container \
  -p 8080:8080 \
  -v "$(pwd)/mydata:/mydata" \
  linux-container:1.0
```

브라우저에서 Python 웹서버가 정상 실행되는 것을 확인했다.

![볼륨을 연결한 Python 웹서버]({{ '/assets/images/dockerfile-practice/05-volume-webserver.png' | relative_url }})

이번에는 웹서버 실행과 볼륨 연결까지 진행했다. 호스트와 컨테이너 사이의 파일 공유 및 컨테이너 삭제 후 데이터 유지 여부는 다음 실습에서 확인할 예정이다.

## 실습 중 해결한 문제

### 컨테이너 이름 충돌

같은 이름의 컨테이너가 남아 있으면 새 컨테이너를 실행할 수 없었다.

```bash
docker ps -a --filter name=linux-container
docker rm -f linux-container
```

### 빌드 실패 후 이전 이미지 실행

빌드가 실패해도 같은 태그의 이전 이미지가 남아 있을 수 있다. 이후의 `docker run` 명령이 그 이미지를 실행하면서 변경 내용이 적용되지 않은 것처럼 보였다. 따라서 빌드의 `FINISHED`를 확인한 뒤 실행하는 것이 중요했다.

### Dockerfile 내용 중복

편집 과정에서 Dockerfile 내용이 중복되면서 두 번째 `FROM`이 별도 빌드 단계로 인식됐다. 그 결과 앞 단계의 `EXPOSE` 설정이 최종 이미지에 포함되지 않아 검사 결과가 `null`로 표시됐다.

```bash
nl -ba Dockerfile
```

줄 번호를 출력해 중복된 내용을 찾아 해결했다.

## 오늘 배운 핵심

- Dockerfile은 이미지 생성 과정을 코드로 정의한다.
- `ARG`는 이미지 빌드 시 사용하는 변수다.
- `LABEL`은 이미지의 메타데이터를 기록한다.
- `EXPOSE`는 사용 예정 포트를 기록하고 실제 연결은 `-p`가 담당한다.
- `WORKDIR`와 `COPY`로 이미지 내부에 애플리케이션 파일을 배치할 수 있다.
- 일반 사용자로 실행할 때는 파일과 포트 권한을 함께 설정해야 한다.
- `ENTRYPOINT`는 실행 프로그램, `CMD`는 기본 명령 또는 기본 인자를 정의한다.
- Multi-Stage Build는 빌드 환경과 실행 환경을 분리한다.
- 볼륨은 호스트와 컨테이너 사이의 데이터를 연결하고 유지하는 데 사용한다.
- 문제 해결에는 `docker ps -a`, `docker logs`, `docker inspect`가 유용하다.

직접 오류를 만들고 해결하면서 Dockerfile의 각 명령이 이미지와 컨테이너에 어떤 영향을 주는지 확인할 수 있었다.
