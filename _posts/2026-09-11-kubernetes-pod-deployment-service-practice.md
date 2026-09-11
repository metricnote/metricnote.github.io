---
layout: post
title: "Kubernetes 입문 실습: 이미지 배포부터 Pod, Deployment, Service까지"
date: 2026-09-11 16:30:00 +0900
category: [backend-cloud]
tags: [Kubernetes, K8s, Docker, Harbor, Pod, Deployment, ReplicaSet, Service, EndpointSlice, Port-Forward, Rolling-Update, learning-note]
---

Docker에서는 컨테이너 하나를 직접 실행하고 종료했다. 하지만 실제 서비스에서는 컨테이너가 수십, 수백 개로 늘어나고 여러 서버에 흩어진다. 어떤 서버에서 실행할지, 장애가 발생하면 어떻게 복구할지, 사용자가 늘었을 때 몇 개를 추가할지 사람이 계속 판단하기는 어렵다.

이번 실습에서는 직접 만든 이미지를 Harbor Registry에 올린 뒤 Kubernetes의 Pod, Deployment, ReplicaSet, Service로 배포했다. 단순히 명령어를 따라가는 데 그치지 않고 **각 리소스가 왜 필요한지, 서로 어떻게 연결되는지**를 실제 결과와 함께 정리했다.

## 1. Kubernetes란 무엇인가

Kubernetes는 여러 서버에 분산된 컨테이너를 배치하고 운영하는 **컨테이너 오케스트레이션 플랫폼**이다. 원하는 상태를 선언하면 Kubernetes가 현재 상태를 계속 관찰하면서 둘을 일치시킨다.

예를 들어 다음과 같이 선언할 수 있다.

```text
webserver Pod를 항상 2개 실행한다.
컨테이너 이미지는 webserver:2.0을 사용한다.
```

실행 중인 Pod 하나가 장애로 사라지면 Kubernetes는 새로운 Pod를 만들어 다시 2개를 맞춘다. 이미지 버전을 변경하면 기존 Pod를 새 버전의 Pod로 점진적으로 교체한다. 중요한 점은 개발자가 매번 생성 과정을 지시하는 것이 아니라 **원하는 결과를 선언한다는 것**이다.

### Kubernetes를 사용하는 이유

- 여러 서버에 컨테이너를 자동으로 배치한다.
- 필요한 컨테이너 수를 유지하고 확장한다.
- 장애가 발생한 컨테이너를 다시 실행한다.
- 여러 Pod로 요청을 분산한다.
- 서비스를 중단하지 않고 새 버전으로 교체한다.
- YAML로 운영 상태를 기록하고 반복 적용한다.

### 어디에서 사용하는가

Kubernetes는 마이크로서비스, 웹/API 서버, 배치 작업, AI 추론 서비스처럼 여러 컨테이너를 지속적으로 운영해야 하는 환경에서 사용한다. 클라우드의 관리형 Kubernetes인 AWS EKS, Google GKE, Azure AKS뿐 아니라 온프레미스 환경에도 구성할 수 있다.

반대로 컨테이너가 한두 개뿐인 작은 서비스에서는 Kubernetes 운영 복잡도가 더 클 수 있다. Kubernetes는 모든 프로젝트에 무조건 필요한 도구라기보다, **배포 규모와 운영 자동화 요구가 커질 때 효과가 큰 플랫폼**이다.

## 2. 이번 실습의 전체 구조

이번 실습의 흐름은 다음과 같다.

```text
소스 코드
  -> Docker 이미지 빌드
  -> Harbor Registry에 Push
  -> Pod에서 이미지 실행
  -> Deployment가 Pod 개수와 버전 관리
  -> Service가 여러 Pod를 하나의 주소로 연결
```

주요 Kubernetes 리소스의 관계는 다음과 같다.

```text
Deployment
  └─ ReplicaSet
       ├─ Pod 1
       └─ Pod 2

Service
  └─ app 라벨이 일치하는 Pod들을 EndpointSlice로 연결
```

## 3. Kubernetes 접속 환경 준비

교육용 클러스터의 인증정보는 `~/.kube/config`에 저장됐다. 현재 컨텍스트와 기본 Namespace는 다음처럼 확인했다.

```bash
kubectl config current-context
kubectl config view --minify --output 'jsonpath={..namespace}'
echo
```

이번 실습의 Namespace는 `class-1`이었다. `k`와 `kcns`는 Kubernetes의 기본 명령이 아니라 각각 `kubectl`, Namespace 변경 명령에 붙인 편의용 별칭이다. 별칭이 없는 환경에서는 원래 명령을 사용하면 된다.

```bash
# kcns class-1과 같은 역할
kubectl config set-context --current --namespace=class-1

# k get pods와 같은 역할
kubectl get pods
```

`~/.kube/config`에는 클러스터 인증 토큰이 포함될 수 있으므로 전체 내용을 화면에 출력하거나 캡처해 공개하지 않는 것이 좋다.

## 4. 실습용 YAML 생성

실습 저장소의 템플릿에는 사용자 번호, Namespace, Registry 주소 같은 값이 변수로 들어 있었다. `gen-yaml.sh`는 `env.properties`를 읽고 `.t` 템플릿을 실제 `.yaml` 파일로 변환했다.

```bash
cd ~/workspace/skala-kube
gen-yaml.sh
```

스크립트는 실행한 디렉터리와 하위 디렉터리에서 템플릿을 찾는다. 따라서 `env.properties`가 없는 설치용 디렉터리에서 실행하면 다음 오류가 발생한다.

```text
오류: ./env.properties 파일을 찾을 수 없습니다.
```

이 경우 스크립트 자체가 고장 난 것이 아니라 **실행 위치가 잘못된 것**이다.

## 5. Docker 이미지 빌드와 Harbor Push

다음 네 가지 이미지를 빌드했다.

| 이미지 | 버전 | 역할 |
|---|---:|---|
| `webserver` | `1.0` | Spring 기반 API 서버 |
| `webserver` | `2.0` | FastAPI 기반 상태 확인 서버 |
| `frontend` | `1.0` | Nginx 기반 정적 Frontend |
| `vue-frontend` | `1.0` | Vue 기반 Frontend |

각 실습 디렉터리의 스크립트로 이미지 생성과 Push를 수행했다.

```bash
./docker-build.sh
./docker-push.sh
```

비밀번호를 명령 기록이나 화면에 노출하지 않도록 Registry 로그인에는 표준 입력을 사용할 수 있다.

```bash
printf '%s' "$DOCKER_REGISTRY_PASSWORD" |
docker login "$DOCKER_REGISTRY" \
  -u "$DOCKER_REGISTRY_USER" \
  --password-stdin
```

Spring 이미지 `1.0`이 울산 Harbor Registry에 Push된 것을 확인했다. 빌드 스크립트에서는 교육생 번호, 이미지 이름과 버전을 조합하고 `linux/amd64`, `linux/arm64` 플랫폼을 대상으로 이미지를 생성한다.

![docker-build.sh를 이용한 멀티 아키텍처 이미지 빌드 과정]({{ '/assets/images/kubernetes-intro-practice/01-harbor-webserver-v1.png' | relative_url }})

*`docker-build.sh`의 구성과 Buildx 실행 흐름을 확인한 화면.*

### 컨테이너 이름 충돌

로컬 테스트 중 `frontend`와 `vue-frontend`라는 이름의 이전 컨테이너가 남아 다음 오류가 발생했다.

```text
Conflict. The container name is already in use.
```

이는 이미지 오류가 아니라 Docker 컨테이너 이름의 중복 문제다. 기존 실습 컨테이너를 확인하고 종료한 뒤 제거했다.

```bash
docker ps -a --filter 'name=^/frontend$'
docker stop frontend
docker rm frontend
```

FastAPI `2.0`, 정적 Frontend와 Vue Frontend가 모두 로컬에서 정상적으로 실행되는 것을 확인했다. 세 이미지는 동일한 결과를 반복한 것이 아니라 서로 다른 컨테이너를 차례대로 검증한 결과다.

### FastAPI 2.0 실행 확인

FastAPI 이미지를 실행해 서버 상태와 Ready 상태를 변경할 수 있는 상태 확인 화면이 열리는지 검증했다.

![FastAPI 2.0 컨테이너 로컬 실행 결과]({{ '/assets/images/kubernetes-intro-practice/02-fastapi-local-test.png' | relative_url }})

*FastAPI `webserver:2.0` 컨테이너의 상태 확인 화면.*

### Nginx 정적 Frontend 실행 확인

다음으로 HTML, CSS와 JavaScript 정적 파일을 Nginx로 제공하는 `frontend:1.0` 이미지를 실행했다.

![Nginx 기반 정적 Frontend 실행 결과]({{ '/assets/images/kubernetes-intro-practice/03-static-frontend.png' | relative_url }})

*Nginx 기반 정적 Frontend 컨테이너의 주문 관리 화면.*

### Vue Frontend 실행 확인

마지막으로 Vue SPA를 빌드한 `vue-frontend:1.0` 이미지를 실행했다. 화면 구성은 정적 Frontend와 유사하지만 구현 방식과 빌드 과정이 다르다.

![Vue Frontend 실행 결과]({{ '/assets/images/kubernetes-intro-practice/04-vue-frontend.png' | relative_url }})

*Vue.js로 구현한 Frontend 컨테이너의 주문 관리 화면.*

## 6. Pod 직접 배포

Pod는 Kubernetes가 컨테이너를 실행하고 관리하는 기본 단위다. 이번 YAML은 Harbor에 Push한 Spring 이미지 `1.0`을 사용하는 Pod를 선언했다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: student01-webserver
  namespace: class-1
  labels:
    app: student01-webserver
spec:
  containers:
    - name: webserver
      image: registry.example.com/class-1/student01-webserver:1.0
      imagePullPolicy: Always
```

YAML을 적용하고 상태 변화를 관찰했다.

```bash
cd ~/workspace/skala-kube/01.pod
kubectl apply -f webserver-pod.yaml
kubectl get pod student01-webserver -w
```

Pod는 `ContainerCreating`을 거쳐 `1/1 Running` 상태가 됐다. `-w`는 리소스 변화를 계속 감시하는 옵션이다. `Ctrl+C`로 감시를 종료해도 Pod는 삭제되지 않는다.

상세 정보와 컨테이너 내부도 확인했다.

```bash
kubectl describe pod student01-webserver
kubectl exec -it student01-webserver -- /bin/bash
```

`describe`의 Events에서는 스케줄링, 이미지 Pull, 컨테이너 생성과 시작 과정을 확인할 수 있었다.

## 7. Port Forward로 Pod에 접속

Pod에는 `192.168.x.x` 형태의 IP가 할당됐지만 이 주소는 클러스터 내부 네트워크에서 사용하는 사설 주소다. Mac의 브라우저가 AWS 클러스터 내부 Pod IP로 직접 접근할 수 없기 때문에 `kubectl port-forward`로 임시 통로를 만들었다.

```bash
kubectl port-forward pod/student01-webserver 8080:8080
```

두 숫자의 의미는 다음과 같다.

```text
localhost:8080 -> Pod:8080
```

브라우저는 `http://localhost:8080`으로 요청하고, `kubectl`이 인증된 Kubernetes 연결을 통해 Pod의 8080번 포트로 전달한다. 포트포워딩은 터미널이 실행되는 동안만 유지되며 `Ctrl+C`를 누르면 통로만 종료된다. Pod 자체는 계속 실행된다.

포트포워딩은 개발과 장애 확인을 위한 임시 접근 방법이다. 실제 사용자에게 서비스를 공개할 때는 일반적으로 Service와 Ingress를 사용한다.

## 8. Deployment와 ReplicaSet

Pod를 직접 만들면 해당 Pod가 삭제됐을 때 자동으로 같은 Pod가 복구되지 않는다. Deployment를 사용하면 원하는 Pod 수와 이미지 버전을 선언하고 지속적으로 유지할 수 있다.

실습 전 직접 생성했던 `student01-webserver` Pod를 삭제한 뒤 Deployment를 적용했다.

```bash
kubectl delete pod student01-webserver --ignore-not-found

cd ~/workspace/skala-kube/02.deploy
kubectl apply -f deploy.yaml
```

Deployment가 생성되자 ReplicaSet과 Pod가 차례로 만들어졌다.

```text
Deployment  student01-webserver
  └─ ReplicaSet  student01-webserver-88d954f4b
       └─ Pod  student01-webserver-88d954f4b-xswbg
```

Deployment가 만든 Pod 이름에는 템플릿 해시와 임의 문자열이 붙는다. 이를 통해 직접 만든 `student01-webserver` Pod와 구분할 수 있었다.

### replicas를 1개에서 2개로 확장

`deploy.yaml`의 원하는 Pod 수를 변경했다.

```yaml
spec:
  replicas: 2
```

```bash
kubectl apply -f deploy.yaml
kubectl get pod -l app=student01-webserver
kubectl get replicaset -l app=student01-webserver
```

결과는 다음과 같았다.

```text
NAME                        DESIRED   CURRENT   READY
student01-webserver-88d954f4b   2         2         2
```

- `DESIRED`: 선언한 목표 Pod 수
- `CURRENT`: 현재 생성된 Pod 수
- `READY`: 요청을 처리할 준비가 된 Pod 수

세 값이 모두 2가 되면서 원하는 상태와 실제 상태가 일치했다.

## 9. 이미지 1.0에서 2.0으로 롤링 업데이트

Deployment의 이미지를 Spring `1.0`에서 FastAPI `2.0`으로 변경했다.

```yaml
image: registry.example.com/class-1/student01-webserver:2.0
```

```bash
kubectl apply -f deploy.yaml
kubectl rollout status deployment/student01-webserver
```

업데이트 중에는 새 Pod가 하나씩 생성되고 준비된 뒤 기존 Pod가 종료됐다.

```text
Waiting for deployment rollout to finish...
1 old replicas are pending termination...
deployment "student01-webserver" successfully rolled out
```

완료 후에는 새로운 ReplicaSet이 Pod 2개를 관리하고, 기존 ReplicaSet의 목표 개수는 0이 됐다.

```text
NAME                        DESIRED   CURRENT   READY
student01-webserver-88d954f4b   0         0         0
student01-webserver-c964fd64f   2         2         2
```

기존 ReplicaSet이 삭제되지 않고 남아 있는 이유는 업데이트 이력과 롤백을 지원하기 위해서다. 이것이 컨테이너를 직접 중지하고 다시 실행하는 방식과 Deployment의 중요한 차이다.

## 10. Service와 EndpointSlice

Pod는 재생성될 때 이름과 IP가 바뀔 수 있다. 사용자가 개별 Pod IP를 직접 사용하면 Pod가 교체될 때마다 접속 주소도 바꿔야 한다. Service는 여러 Pod 앞에서 변하지 않는 ClusterIP와 이름을 제공한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: student01-webserver
spec:
  type: ClusterIP
  selector:
    app: student01-webserver
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

Service를 배포하고 연결 대상을 확인했다.

```bash
kubectl apply -f service.yaml
kubectl get service student01-webserver
kubectl get endpoints student01-webserver
kubectl get endpointslice \
  -l kubernetes.io/service-name=student01-webserver \
  -o wide
```

실습 결과 Service에는 `10.100.x.x`라는 ClusterIP가 할당됐고 EndpointSlice에는 Deployment가 만든 Pod IP 두 개가 등록됐다.

```text
Service ClusterIP: 10.100.x.x
Pod endpoints:     192.168.x.35, 192.168.x.231
Ports:             8080, 8081
```

Service가 Pod 이름을 직접 추적하는 것이 아니라 `app=student01-webserver` 라벨을 선택한다는 점이 중요하다. Pod가 교체돼 IP가 달라지면 EndpointSlice가 새로운 IP로 자동 갱신된다.

`v1 Endpoints is deprecated` 경고도 확인했다. 이는 오류가 아니라 최신 Kubernetes에서 기존 Endpoints 대신 `discovery.k8s.io/v1`의 EndpointSlice 사용을 권장한다는 뜻이다.

Service를 통한 연결도 확인했다.

```bash
kubectl port-forward service/student01-webserver 8080:8080
```

![Service 포트포워딩으로 FastAPI 2.0에 접속한 결과]({{ '/assets/images/kubernetes-intro-practice/05-service-port-forward.png' | relative_url }})

`kubectl port-forward service/...`는 연결 확인에는 유용하지만 실제 로드밸런싱 시험과는 다르다. 포트포워딩 과정에서는 Service 뒤의 Pod 하나를 선택해 임시 통로를 만든다. 실제 클러스터 내부에서 ClusterIP로 들어오는 서비스 트래픽은 여러 Endpoint로 전달된다.

## 11. 강의자료와 실제 명령이 달랐던 이유

실습 중 강의자료의 명령과 실제 사용한 명령에 몇 가지 차이가 있었다.

| 강의자료 예시 | 실제 명령 | 이유 |
|---|---|---|
| `k` | `kubectl` | `k` 별칭이 등록되지 않은 환경에서도 실행 가능 |
| `{학번}` | `student01` | 템플릿 자리에 실제 교육생 번호 적용 |
| `vi webserver-pod.yaml` | `cat webserver-pod.yaml` | 생성된 파일을 수정하지 않고 확인만 할 때 사용 |
| `kubectl get pod -w` | `kubectl get pod student01-webserver -w` | 같은 Namespace의 다른 교육생 Pod 제외 |
| `8080:80` | `8080:8080` | 현재 애플리케이션이 컨테이너 8080 포트에서 실행 |
| `--address 0.0.0.0` | 기본값 `127.0.0.1` | 주변 네트워크에 공개하지 않고 로컬에서만 접속 |

강의자료는 원리를 설명하는 공통 예시이고, 실제 명령은 현재 이미지, 사용자 번호, Namespace와 포트에 맞춰야 한다. 특히 포트는 자료의 숫자를 그대로 복사하기보다 애플리케이션이 실제로 어느 포트에서 대기하는지 확인해야 한다.

## 12. 실습을 마치며

이번 실습을 통해 Kubernetes를 단순히 “컨테이너를 실행하는 도구”로 보는 것에서 벗어날 수 있었다.

- Pod는 컨테이너를 실행하는 기본 단위다.
- Deployment는 원하는 Pod 개수와 버전을 선언한다.
- ReplicaSet은 실제 Pod 수를 유지한다.
- 롤링 업데이트는 실행 중인 서비스를 점진적으로 새 버전으로 교체한다.
- Service는 바뀌는 Pod들 앞에 안정적인 주소를 제공한다.
- EndpointSlice는 Service가 연결할 실제 Pod IP 목록을 관리한다.
- Port Forward는 외부에 공개되지 않은 Pod나 Service를 로컬에서 잠시 확인하는 통로다.

결국 Kubernetes의 핵심은 다음 한 문장으로 정리할 수 있다.

> 개발자가 원하는 상태를 선언하면 Kubernetes가 현재 상태를 지속적으로 조정해 그 상태를 유지한다.

이미지를 빌드하고 한 컨테이너를 실행하는 것은 Docker의 역할이었다. 이번에는 그 이미지를 바탕으로 Pod를 배치하고, 개수를 확장하고, 버전을 교체하고, 안정적인 Service 주소로 연결하면서 Kubernetes가 실제 운영 문제를 어떻게 해결하는지 확인했다.
