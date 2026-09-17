---
layout: post
title: "모델 서빙 Day 2: Airflow부터 KServe·Kubeflow까지 운영 자동화 실습"
date: 2026-09-17 18:00:00 +0900
category: [ai]
tags: [Model-Serving, MLOps, Airflow, Ray-Serve, vLLM, Kubernetes, KServe, Kubeflow-Pipelines, Argo-Workflows, learning-note]
---

Day 1에는 scikit-learn, PyTorch, Keras 모델을 파일로 저장하고 FastAPI로 서빙했다. 단일 서버에서 모델이 요청을 받고 예측을 반환하는 가장 기본적인 구조였다.

Day 2에는 여기서 한 단계 더 나아갔다. 모델 하나를 API로 여는 데서 끝나지 않고, **학습 작업을 어떤 순서로 실행할지, 요청이 늘어날 때 어떻게 확장할지, Kubernetes에서 모델을 어떻게 배포할지, 정확도 기준을 통과한 모델만 어떻게 자동으로 배포할지**를 실습했다.

이번 실습 전체를 한 줄로 정리하면 다음과 같다.

```text
Airflow로 작업 흐름 자동화
→ Ray Serve로 분산 서빙 구조 확인
→ vLLM으로 LLM API 실행
→ KServe로 Kubernetes 모델 배포
→ Kubeflow Pipelines로 학습·평가·조건부 배포 자동화
```

## 왜 이런 실습이 필요했을까?

모델이 노트북이나 Python 파일에서 잘 작동하는 것과 실제 서비스에서 안정적으로 운영되는 것은 다른 문제다.

운영 환경에서는 다음 질문에 답해야 한다.

- 학습, 평가, 배포 작업을 매번 사람이 실행해야 할까?
- 어떤 작업을 먼저 실행하고 어떤 작업을 병렬로 처리할까?
- 요청이 많아지면 여러 프로세스나 서버에 어떻게 나눌까?
- 컨테이너가 중단되면 누가 다시 실행할까?
- 새 모델의 정확도가 기준보다 낮아도 배포해도 될까?
- 실행 기록과 실패 지점은 어디에서 확인할까?

이 문제들을 해결하는 과정이 MLOps이며, 이번 실습에서는 각 도구가 담당하는 역할을 직접 확인했다.

## 실습 환경

- macOS Apple Silicon
- Docker Desktop
- Python 3.11
- Apache Airflow
- Ray Serve
- vLLM과 Qwen2.5-0.5B-Instruct
- kind 기반 로컬 Kubernetes
- KServe 0.18
- Kubeflow Pipelines 2.17

## 1. Airflow: ML 작업을 DAG로 자동화하기

Airflow 실습에서는 ML 파이프라인을 DAG(Directed Acyclic Graph)로 정의했다. DAG는 작업 사이의 선후 관계를 표현하되 순환은 허용하지 않는 구조다.

실습한 ML 파이프라인의 흐름은 다음과 같았다.

```text
t1_ingest
→ t2_preprocess
→ ┌─ t3_train_logreg ─┐
  └─ t4_train_rf ─────┘
→ t5_evaluate_select
→ t6_serve_smoke_test
```

데이터 수집과 전처리 후 Logistic Regression과 Random Forest를 병렬로 학습하고, 평가 단계에서 더 좋은 모델을 선택한 다음 간단한 서빙 테스트를 수행했다.

Airflow 화면에 접속해 Scheduler, Triggerer, DAG Processor 등 주요 구성요소가 정상인지 확인했다.

<p align="center"><img src="{{ '/assets/images/model-serving-day2/01-airflow-dashboard.png' | relative_url }}" alt="Airflow 대시보드와 주요 구성요소 상태"></p>

수동 실행과 예약 실행 모두 성공했고, 실제 시작 시각에서도 `t3_train_logreg`와 `t4_train_rf`가 같은 시각에 병렬로 시작한 것을 확인했다.

```text
08:32:05  t1_ingest             success
08:32:06  t2_preprocess         success
08:32:07  t3_train_logreg       success
08:32:07  t4_train_rf           success
08:32:08  t5_evaluate_select    success
08:32:09  t6_serve_smoke_test   success
```

XCom에 저장된 정확도는 다음과 같았다.

| 모델 | 정확도 |
|---|---:|
| Logistic Regression | 0.9667 |
| Random Forest | 0.9333 |

또한 세 가지 실행 방식도 비교했다.

- `Sensor`: 특정 파일이 생길 때까지 주기적으로 확인한 뒤 작업 실행
- `Asset`: 생산 DAG가 데이터를 갱신하면 소비 DAG 자동 실행
- `AssetWatcher`: 폴더에 새 CSV 파일이 생긴 이벤트를 감지해 DAG 실행

여기서 배운 핵심은 Airflow가 모델 자체를 더 똑똑하게 만드는 도구가 아니라, **데이터 준비부터 학습·평가·검증까지의 작업 순서를 안정적으로 관리하는 도구**라는 점이다.

## 2. Ray Serve: 여러 배포 단위를 연결해 서비스하기

Ray Serve에서는 `Greeter`, `Shouter`, `Driver`라는 세 배포 단위를 연결했다.

```text
HTTP 요청
→ Driver
→ Greeter가 인사말 생성
→ Shouter가 강조 문장 생성
→ JSON 응답
```

API 호출 결과는 다음과 같았다.

```json
{
  "greeting": "안녕하세요, 사용자님!",
  "shouted": "안녕하세요, 사용자님!!!!",
  "served_by_method": "1b_Docker_Ray컨테이너",
  "hostname": "<container-id>"
}
```

응답의 `hostname`과 실제 Docker 컨테이너 ID가 일치했다. 따라서 요청이 Ray Serve가 실행 중인 컨테이너에서 처리됐음을 확인할 수 있었다. 공개 글에서는 불필요한 식별값을 일반 표기로 대체했다.

`serve status`에서도 Proxy가 `HEALTHY`, 애플리케이션이 `RUNNING`, 세 Deployment가 모두 `HEALTHY`로 표시됐다.

이 실습을 통해 하나의 큰 서버 함수에 모든 기능을 넣지 않고, **기능을 독립적인 Deployment로 나누고 조합할 수 있다**는 것을 배웠다. 각 Deployment의 replica 수를 따로 조정할 수 있다는 점도 실제 트래픽 대응에 중요한 특징이다.

## 3. vLLM: OpenAI 호환 LLM API 실행하기

vLLM 실습에서는 CPU용 컨테이너에서 `Qwen/Qwen2.5-0.5B-Instruct` 모델을 실행했다. Apple Silicon 환경에 맞는 ARM64 이미지를 사용했고, 모델 로딩에는 약 38초가 걸렸다.

<p align="center"><img src="{{ '/assets/images/model-serving-day2/03-vllm-ready.png' | relative_url }}" alt="vLLM 모델 로딩과 준비 완료 화면"></p>

모델 목록 API에서 다음 정보를 확인했다.

```text
id:       assistant
root:     Qwen/Qwen2.5-0.5B-Instruct
max_len:  1024
```

그다음 OpenAI 호환 `/v1/chat/completions` API에 질문을 보냈다. curl 요청과 OpenAI Python SDK 클라이언트가 같은 서버를 사용할 수 있었다.

응답의 `finish_reason`은 `length`였는데, 이는 오류가 아니라 설정한 `max_tokens=64`를 모두 사용해 생성이 중단됐다는 뜻이다. 작은 0.5B 모델이라 모델 서빙의 정의도 완벽하지는 않았다.

여기서 중요한 것은 답변의 품질보다 **기존 OpenAI 형식의 클라이언트를 유지하면서 API 주소만 로컬 vLLM 서버로 바꿀 수 있었다는 점**이다.

vLLM이 대규모 언어 모델 서빙에 적합한 이유도 함께 이해했다.

- PagedAttention으로 KV 캐시 메모리를 블록 단위 관리
- Continuous Batching으로 매 생성 단계마다 활성 요청을 다시 묶어 처리
- OpenAI 호환 API 제공

즉, vLLM은 단순히 모델을 실행하는 도구가 아니라 여러 생성 요청을 효율적으로 처리하기 위한 LLM 전용 서빙 엔진이다.

## 4. KServe: Kubernetes에 모델 배포하기

KServe 실습에서는 `kind`로 로컬 Kubernetes 클러스터를 만든 뒤 scikit-learn Iris 모델을 `InferenceService`로 배포했다.

```text
InferenceService 선언
→ KServe Controller가 Predictor 생성
→ Storage Initializer가 모델 다운로드
→ 모델 서버 실행
→ REST 요청으로 예측
```

Storage Initializer는 GCS의 모델 파일을 `/mnt/models`로 내려받았고, KServe 컨테이너는 HTTP 8080과 gRPC 8081에서 서버를 시작했다.

로그에는 scikit-learn 1.0.1로 저장한 모델을 1.5.2에서 불러왔다는 버전 경고가 있었다. 서버는 정상 작동했지만, pickle 계열 모델은 라이브러리 버전 호환성이 영구적으로 보장되지 않는다는 점을 다시 확인했다.

로컬 포트 `8090`을 Predictor 서비스로 연결했다.

<p align="center"><img src="{{ '/assets/images/model-serving-day2/05-kserve-port-forward.png' | relative_url }}" alt="KServe Predictor 서비스 포트 포워딩"></p>

V1 예측 API를 호출한 결과 HTTP `200 OK`와 예측값 `[1, 1]`이 반환됐다.

<p align="center"><img src="{{ '/assets/images/model-serving-day2/04-kserve-predict.png' | relative_url }}" alt="KServe Iris 모델 예측 성공 결과"></p>

FastAPI 실습에서는 서버 프로세스를 직접 실행하고 관리했다. 반면 KServe에서는 원하는 모델과 저장 위치를 선언하면 Kubernetes와 KServe가 실제 Predictor Pod를 만들고 상태를 관리한다.

이 차이를 통해 Kubernetes 기반 서빙에서는 **직접 서버를 켜는 방식에서 원하는 상태를 선언하는 방식으로 관점이 바뀐다**는 것을 배웠다.

## 5. Kubeflow Pipelines: 학습부터 조건부 배포까지 자동화하기

마지막으로 별도의 kind 클러스터에 Kubeflow Pipelines 2.17을 설치했다. 먼저 Argo Workflow와 KFP가 사용하는 CRD를 등록했다.

<p align="center"><img src="{{ '/assets/images/model-serving-day2/06-kubeflow-crd.png' | relative_url }}" alt="Kubeflow Pipelines CRD 설치 완료"></p>

전체 스택을 설치한 뒤 파드 상태는 다음과 같았다.

```text
13개 파드: Running
metadata-writer 1개: ImagePullBackOff
```

`metadata-writer`는 Apple Silicon용 이미지가 없어 실행되지 않았지만 ML Metadata 계보 기록을 담당하는 컴포넌트이므로 이번 파이프라인 실행 자체에는 영향을 주지 않았다.

API 서버의 health endpoint에서는 KFP 버전 `2.17.0`과 정상 응답을 확인했다.

<p align="center"><img src="{{ '/assets/images/model-serving-day2/07-kubeflow-health.png' | relative_url }}" alt="Kubeflow Pipelines API 서버 health 응답"></p>

Python으로 정의한 파이프라인을 `iris_pipeline.yaml`로 컴파일한 뒤 KFP API 서버에 제출했다.

```text
train-step
→ eval-step
→ condition-1: 정확도 >= 0.9
→ deploy-step
```

최종 결과는 `SUCCEEDED`였고 Argo Workflow의 주요 노드도 모두 `Succeeded`였다.

```text
train-step       Succeeded
eval-step        Succeeded
condition-1      Succeeded
deploy-step      Succeeded
```

실제 컴포넌트 로그에서는 다음 결과를 확인했다.

```text
[train_step] 학습 완료 — model artifact 생성
[eval_step] 테스트 정확도: 0.9333
[deploy_step] 정확도 0.9333 — 배포 조건(0.9) 통과, 배포 실행
```

여기서 `dsl.If`는 단순한 코드상의 if 문이 아니었다. 평가 단계의 출력값을 다음 단계의 실행 조건으로 전달해, 정확도가 기준 이상일 때만 배포 파드가 실행되도록 만들었다.

## 도구별 역할을 다시 정리하면

| 도구 | 이번 실습에서 맡은 역할 | 핵심 결과 |
|---|---|---|
| Airflow | 작업 순서·예약·이벤트 기반 실행 관리 | 병렬 학습과 파일·Asset 트리거 성공 |
| Ray Serve | 여러 서빙 Deployment 조합 | Greeter·Shouter·Driver 모두 Healthy |
| vLLM | LLM 추론 서버와 OpenAI 호환 API | Qwen 모델의 채팅 응답 성공 |
| KServe | Kubernetes 모델 서빙 자동화 | InferenceService Ready, 예측 `[1,1]` |
| Kubeflow Pipelines | ML 파이프라인을 K8s에서 실행 | 정확도 0.9333, 조건부 배포 성공 |

## 이번 실습에서 배운 것

### 1. 모델 서빙은 API 하나를 만드는 것으로 끝나지 않는다

Day 1의 FastAPI는 모델을 서비스로 바꾸는 출발점이었다. 실제 운영에서는 스케줄링, 병렬 처리, 상태 확인, 자동 복구, 버전 관리, 평가와 배포 조건까지 함께 고려해야 한다.

### 2. 오케스트레이션은 작업을 대신 실행하는 것 이상이다

Airflow와 Kubeflow는 단순히 명령을 순서대로 실행하지 않는다. 작업 사이의 의존성을 기록하고, 병렬 실행을 결정하며, 실패 지점을 보여주고, 이전 단계의 결과를 다음 단계로 전달한다.

### 3. 범용 워크플로와 ML 전용 파이프라인은 역할이 다르다

Airflow는 파일 처리, 데이터 수집, 예약 작업처럼 범용 워크플로에 강하다. Kubeflow Pipelines는 모델 아티팩트, 평가 지표, 조건부 배포처럼 ML 파이프라인에 특화돼 있다. 두 도구는 경쟁 관계라기보다 상황에 따라 함께 사용할 수 있다.

### 4. Kubernetes에서는 선언한 상태를 플랫폼이 유지한다

KServe의 `InferenceService`를 제출하면 Controller가 Predictor Pod를 만들었다. 사용자는 구현 세부사항보다 “이 모델이 실행 중이어야 한다”는 원하는 상태를 선언한다. 이것이 Kubernetes 운영 방식의 핵심이다.

### 5. 자동 배포에는 반드시 품질 기준이 필요하다

파이프라인이 자동화됐다는 이유만으로 모든 모델을 배포해서는 안 된다. 이번에는 정확도 `0.9`를 기준으로 사용했고, 실제 결과 `0.9333`이 기준을 통과했을 때만 배포 단계가 실행됐다.

### 6. 경고와 실패를 구분해서 읽어야 한다

실습 중 여러 경고가 있었지만 모두 같은 의미는 아니었다.

- scikit-learn 버전 경고: 현재 실행은 가능하지만 재현성과 호환성에 위험이 있음
- vLLM의 `finish_reason=length`: 생성 토큰 제한에 도달한 정상 종료
- `metadata-writer`의 ImagePullBackOff: Apple Silicon 이미지 문제지만 이번 실행에는 비필수
- Airflow cycle 오류: DAG 규칙을 어긴 실제 정의 오류

로그에서 오류 문구만 보는 것이 아니라, 해당 컴포넌트의 역할과 전체 실행 결과를 함께 판단해야 한다.

## 마무리

이번 실습을 통해 모델 운영의 범위가 훨씬 선명해졌다.

```text
좋은 모델을 만든다
→ 저장하고 API로 제공한다
→ 작업 흐름을 자동화한다
→ 확장 가능한 방식으로 배포한다
→ 품질을 검사한다
→ 기준을 통과한 모델만 서비스에 반영한다
→ 실행 상태와 결과를 계속 관찰한다
```

결국 MLOps의 목적은 도구를 많이 사용하는 것이 아니다. **개발한 모델이 반복 가능하고 안전한 절차를 거쳐 사용자에게 안정적으로 전달되게 만드는 것**이다. Day 1에서 만든 단일 모델 API가 Day 2에서는 자동화된 운영 파이프라인으로 확장됐다.
