---
layout: post
title: "Kubernetes Pod부터 ServiceAccount와 RBAC까지: 인증과 권한의 흐름 이해하기"
date: 2026-10-06 16:30:00 +0900
category: [devops]
tags: [Kubernetes, Pod, Service, Label, Selector, Kubeconfig, ServiceAccount, RBAC, DevOps, learning-note]
---

Kubernetes를 처음 공부하면 Pod, Service, Label, API Server, ServiceAccount, RBAC처럼 생소한 용어가 한번에 등장한다. 개념을 하나씩 볼 때는 알 것 같지만, 서로 어떻게 연결되는지 이해하기가 쉽지 않았다.

오늘은 직접 Kubernetes 클러스터에 Pod를 배포하고, Pod 내부에 마운트된 ServiceAccount 인증 정보로 API Server를 호출했다. 또한 RBAC 권한이 없을 때의 `Forbidden`과 Role을 연결한 뒤의 성공 결과를 비교했다.

오늘 학습한 전체 흐름은 다음과 같다.

```text
Kubernetes를 사용하는 이유
  → Pod와 Service
  → Label과 Selector
  → Kubernetes API Server
  → kubeconfig의 URL·인증서·토큰
  → ServiceAccount
  → Role·RoleBinding
  → ClusterRoleBinding
```

## 1. Kubernetes는 어디에 쓰는가

Docker가 애플리케이션을 컨테이너로 포장하고 실행하는 도구라면, Kubernetes는 많은 컨테이너를 서버 여러 대에서 안정적으로 운영하는 도구다.

예를 들어 주문 API를 Pod 3개로 운영한다고 해보자.

```text
원하는 상태: Pod 3개
현재 상태:   Pod 2개
                 ↓
Kubernetes가 Pod 1개를 새로 생성
```

Kubernetes는 특정 Pod 하나를 수리하며 영구적으로 유지하지 않는다. Pod에 문제가 생기면 새 Pod로 교체하여 전체 서비스가 원하는 상태를 유지한다.

이 관점에서 Kubernetes는 DevOps 전체를 의미하는 것은 아니지만, 배포·확장·장애 복구·운영을 자동화하는 핵심 DevOps 도구라고 볼 수 있다.

## 2. Pod는 무엇인가

Pod는 Kubernetes가 관리하는 가장 작은 실행 단위다. 실제 Spring Boot나 FastAPI 서버 프로그램은 Pod 내부의 Container에서 실행된다.

```text
Node(물리 서버 또는 VM)
└── Pod
    └── Container
        └── Spring Boot 서버
```

입문 단계에서는 Pod를 하나의 서버처럼 생각해도 흐름을 이해하는 데 큰 문제는 없다. 다만 엄밀하게는 “서버 프로그램이 든 컨테이너를 Kubernetes가 배포하고 관리하는 단위”다.

## 3. Label과 Selector로 Service가 Pod를 찾는다

Pod는 재생성될 때마다 이름과 IP가 달라질 수 있다. Service가 Pod의 IP를 직접 기억한다면, Pod가 교체될 때마다 연결 정보를 바꿔야 한다. Kubernetes는 이 문제를 Label과 Selector로 해결한다.

Pod에는 Label을 붙인다.

```yaml
metadata:
  labels:
    app: web
    env: prod
```

Service는 Selector로 Label이 일치하는 Pod를 고른다.

```yaml
spec:
  selector:
    app: web
    env: prod
```

조건이 여러 개라면 모두 일치해야 한다.

```text
Service(app=web, env=prod)
  ├── Pod 1(app=web, env=prod)
  └── Pod 2(app=web, env=prod)
```

이때 Pod가 Service의 소유물이 되는 것은 아니다. Service가 현재 Label 조건이 일치하는 Pod를 논리적으로 선택하는 구조다. 그래서 Pod가 교체되어도 같은 Label만 붙어 있으면 Service가 새 Pod를 다시 찾을 수 있다.

## 4. kubectl도 결국 API를 호출한다

`kubectl get pods`를 실행하면 kubectl이 클러스터의 내부 정보를 직접 읽는 것처럼 보인다. 실제로는 Kubernetes API Server에 HTTP 요청을 보낸다.

```text
kubectl
  → Kubernetes API Server
  → 인증·인가·검증
  → etcd에 상태 저장
  → Controller가 원하는 상태를 구현
```

Kubernetes API는 대략 Core API와 API Group으로 나뉜다.

```text
Core API
/api/v1/pods
/api/v1/services
/api/v1/configmaps

API Group
/apis/apps/v1/deployments
/apis/batch/v1/jobs
/apis/networking.k8s.io/v1/ingresses
```

실습 중에 Deployment 목록을 조회할 때 `/api/v1/.../deployments`를 사용하면 안 된다는 점도 확인했다. Deployment는 Core API가 아니라 `apps` Group에 속하므로 다음 경로를 사용해야 한다.

```text
/apis/apps/v1/namespaces/{namespace}/deployments
```

## 5. kubeconfig은 접속 정보를 묶어둔다

kubectl은 보통 `~/.kube/config`에서 접속 정보를 읽는다. 이 파일은 Kubernetes 클러스터 안에 있는 파일이 아니라 kubectl을 실행하는 내 컴퓨터의 설정 파일이다.

```yaml
clusters:
  - cluster:
      server: https://api-server.example.com
      certificate-authority-data: <CA_DATA>

users:
  - user:
      token: <JWT_TOKEN>

contexts:
  - context:
      cluster: my-cluster
      user: my-service-account
      namespace: class-1
```

구성요소의 역할은 다음과 같다.

| 요소 | 역할 |
| --- | --- |
| `server` | 어느 Kubernetes API Server에 접속할지 지정 |
| CA 인증서 | 접속한 API Server가 진짜인지 확인 |
| Token | 요청한 User 또는 ServiceAccount의 신원 증명 |
| Context | Cluster·User·Namespace 조합 선택 |

## 6. 인증서, 토큰, RBAC의 차이

오늘 가장 헷갈렸던 부분이었다. 처음에는 인증서가 토큰의 유효성을 보증한다고 생각했지만, 둘은 확인하는 대상이 다르다.

```text
CA 인증서
→ 클라이언트가 API Server의 신원을 확인

JWT Token
→ API Server가 요청 주체의 신원을 확인

RBAC
→ 인증된 주체가 해당 작업을 할 수 있는지 확인
```

회사 출입에 비유하면 더 쉽다.

```text
CA 인증서 = 여기가 진짜 회사 건물인가?
Token       = 이 사람은 누구인가?
RBAC        = 이 사람은 어느 방까지 들어갈 수 있는가?
```

또한 토큰은 단순한 암호화 키가 아니라 요청 주체의 정보가 담긴 서명된 자격 증명 데이터다. API Server는 JWT의 서명과 유효성을 검증한다.

## 7. ServiceAccount는 Pod의 신분이다

사람이 kubectl로 Kubernetes에 접속할 수도 있지만, 실제 운영에서는 Pod 내부의 프로그램이 API Server를 호출하는 경우도 많다. ServiceAccount는 이런 Pod나 자동화 프로그램이 사용하는 Kubernetes 내장 계정이다.

Pod에 ServiceAccount를 지정하면 다음 디렉터리에 인증 정보가 자동으로 마운트된다.

```text
/var/run/secrets/kubernetes.io/serviceaccount/
├── ca.crt
├── namespace
└── token
```

실습에서 만든 Pod에서 확인한 결과는 다음과 같았다.

```text
serviceAccountName=default
namespace=class-1
token, ca.crt 마운트 확인
```

그다음 `skala-admin-sa`로 ServiceAccount를 바꾼 API Server를 직접 호출했다.

```sh
SERVICE_ACCOUNT_DIR=/var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat ${SERVICE_ACCOUNT_DIR}/token)
CACERT=${SERVICE_ACCOUNT_DIR}/ca.crt
NAMESPACE=$(cat ${SERVICE_ACCOUNT_DIR}/namespace)
APISERVER=https://kubernetes.default.svc.cluster.local

curl --cacert "${CACERT}" \
  --header "Authorization: Bearer ${TOKEN}" \
  "${APISERVER}/apis/apps/v1/namespaces/${NAMESPACE}/deployments"
```

이 실습을 통해 Pod 내부의 애플리케이션도 동일한 방식으로 Kubernetes API를 호출한다는 것을 확인했다.

## 8. 실습: 나만의 ServiceAccount와 kubeconfig 만들기

실습에서는 `class-1`, `sk006`을 기준으로 다음 변수를 사용했다.

```sh
NS=class-1
STUDENT=sk006
APP=${STUDENT}-curl
SA=${NS}-${STUDENT}-sa
```

ServiceAccount의 이름은 `class-1-sk006-sa`가 된다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: class-1-sk006-sa
  namespace: class-1
```

이 ServiceAccount를 이용하는 별도의 kubeconfig도 생성했다. 그런데 새 kubeconfig로 Pod를 조회하자 다음 오류가 발생했다.

```text
Error from server (Forbidden):
pods is forbidden:
User "system:serviceaccount:class-1:class-1-sk006-sa"
cannot list resource "pods"
```

이 결과는 인증에 실패한 것이 아니다. API Server가 ServiceAccount가 누구인지는 정상적으로 알아냈지만, Pod 목록을 조회할 RBAC 권한이 없어서 거부한 것이다.

```text
Unauthorized = 너가 누구인지 확인하지 못함
Forbidden    = 누구인지는 알지만 이 작업은 할 수 없음
```

## 9. Role과 RoleBinding으로 권한 부여하기

Role은 어떤 리소스에 어떤 작업을 할 수 있는지 정의한다. RoleBinding은 그 Role을 User나 ServiceAccount에게 연결한다.

```text
ServiceAccount
      ↓ RoleBinding
     Role
      ↓
Pod·Service·Deployment에 대한 허용 작업
```

Role의 rule은 `apiGroups`, `resources`, `verbs`로 구성된다.

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

위 설정은 Core API의 Pod에 대해 단건 조회, 목록 조회, 변경 감시를 허용한다.

실습용 Role과 RoleBinding을 적용한 후에는 전용 kubeconfig로 Pod 목록을 조회할 수 있었다. 또한 Pod 안에서 직접 API를 호출하여 네임스페이스 범위를 비교했다.

```text
class-1 Pod 조회 → HTTP 200
default Pod 조회 → HTTP 403 Forbidden
```

이것이 Role의 핵심이다. Role은 특정 네임스페이스 안에서만 권한을 부여한다.

## 10. ClusterRoleBinding은 클러스터 전역에 적용된다

다음 실습에서는 `class-1-sk006-sa`를 기존 `cluster-admin` ClusterRole과 연결했다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: class-1-sk006-sa-clusterrole-binding
subjects:
  - kind: ServiceAccount
    name: class-1-sk006-sa
    namespace: class-1
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

적용 전에는 ServiceAccount 목록 조회가 `Forbidden`이었지만, ClusterRoleBinding 적용 후에는 조회가 성공했다.

```text
ClusterRoleBinding 전 → 403 Forbidden
ClusterRoleBinding 후 → ServiceAccount 목록 조회 성공
```

다만 `cluster-admin`은 클러스터 전체를 제어할 수 있는 매우 강한 권한이다. 실제 운영 환경에서는 실습처럼 편의를 위해 부여해서는 안 된다. 필요한 리소스와 verb만 허용하는 최소 권한 원칙을 적용해야 한다.

## 11. 실습하며 알게 된 보안 주의점

실습에서 토큰을 확인할 때 `kubectl describe secret`가 클러스터 환경에 따라 토큰 원문을 출력할 수 있음을 확인했다. 토큰이 터미널 로그나 공유 화면에 노출되면 즉시 교체해야 한다.

안전한 습관은 다음과 같다.

- JWT 토큰 원문을 터미널에 출력하지 않는다.
- 실제 토큰을 공개 JWT 분석 사이트에 올리지 않는다.
- kubeconfig는 Git에 커밋하지 않는다.
- kubeconfig 파일 권한은 `600`으로 제한한다.
- 토큰이 노출되면 Secret을 삭제·재생성해 교체한다.
- `cluster-admin`은 실습 후 회수한다.

또한 Kubernetes 리소스 이름은 일반적으로 DNS 명명 규칙을 따르므로 대문자 대신 소문자를 사용하는 것이 안전하다.

## 12. 오늘의 핵심 정리

오늘 학습을 통해 Kubernetes를 단순히 “Pod를 실행해 주는 도구”로만 보던 관점이 바뀌었다. Kubernetes는 원하는 상태를 선언하면 그 상태를 유지하고, 모든 작업을 API Server를 통해 제어한다.

핵심을 다시 압축하면 다음과 같다.

```text
Pod
→ 컨테이너를 Kubernetes가 관리하는 실행 단위

Label + Selector
→ Service와 Pod를 IP가 아닌 메타데이터로 연결

API Server
→ 모든 Kubernetes 작업의 단일 접수 창구

CA 인증서
→ 클라이언트가 서버의 신원을 확인

JWT Token
→ API Server가 요청 주체의 신원을 확인

RBAC
→ 인증된 주체의 허용 작업을 판단

RoleBinding
→ 특정 Namespace 범위의 권한

ClusterRoleBinding
→ 클러스터 전역 범위의 권한
```

가장 큰 수확은 인증과 인가를 구분하게 된 것이다. Token은 권한 그 자체가 아니다. Token은 “누구인지”를 보여 주고, RBAC이 “무엇을 할 수 있는지”를 결정한다. 이 구분을 이해하니 `Unauthorized`, `Forbidden`, RoleBinding, ClusterRoleBinding이 하나의 흐름으로 연결되었다.
