---
layout: post
title: "모델 저장부터 FastAPI 서빙까지: scikit-learn·PyTorch·Keras 실습"
date: 2026-09-16 17:00:00 +0900
category: [ai]
tags: [Model-Serving, MLOps, Scikit-learn, PyTorch, Keras, FastAPI, Pickle, TorchScript, TensorFlow, Uvicorn, Performance-Test, learning-note]
---

지난 글에서는 모델 저장과 서빙 실습에서 무엇을 다룰지 미리 살펴봤다. 이번에는 Iris 데이터를 이용해 scikit-learn, PyTorch, Keras 모델을 직접 학습하고 저장한 뒤 FastAPI로 제공했다.

이번 실습의 핵심은 프레임워크 사용법을 외우는 것이 아니었다. 서로 다른 프레임워크에서도 다음 구조가 반복된다는 것을 확인하는 과정이었다.

```text
모델 학습
→ 파일로 저장
→ 새로운 프로세스에서 복원
→ 서버 시작 시 한 번 로드
→ HTTP 요청으로 예측
→ JSON 응답 반환
```

## 실습 환경

- macOS Apple Silicon
- Python 3.11.15
- uv 가상환경
- scikit-learn 1.9.1
- Keras 3.15.1
- TensorFlow 백엔드
- FastAPI·Uvicorn

각 실습은 별도의 가상환경에서 진행했다. 시스템의 기본 Python 버전에 영향을 받지 않도록 `uv`로 Python 3.11 환경을 만들었다.

## 1. scikit-learn 모델을 pickle로 저장하기

첫 번째 실습에서는 Iris 데이터로 `RandomForestClassifier`를 학습했다. Iris 데이터에는 꽃받침과 꽃잎의 길이·너비, 그리고 붓꽃의 품종이 들어 있다.

```text
입력 4개
- 꽃받침 길이
- 꽃받침 너비
- 꽃잎 길이
- 꽃잎 너비

출력 3개
- setosa
- versicolor
- virginica
```

모델 학습 명령은 다음과 같다.

```bash
command python train_and_pickle.py
```

실행 결과 테스트 데이터 30개 중 28개를 맞혔다.

```text
[학습 직후] 테스트 정확도: 0.9333 (28/30)
[pickle] 저장 완료: model.pkl
[데이터] 테스트셋 저장 완료: test_data.npz
```

<p align="center"><img src="{{ '/assets/images/model-serving-day1/01-sklearn-train-pickle.png' | relative_url }}" alt="scikit-learn 모델 학습과 pickle 저장 결과"></p>

생성된 파일의 역할은 다음과 같다.

- `model.pkl`: 학습이 끝난 RandomForest 모델
- `test_data.npz`: 복원 결과를 확인할 테스트 데이터와 정답

pickle 저장은 게임의 세이브 파일과 비슷하다. 학습이 끝난 모델의 상태를 파일로 저장해 두면 다시 학습하지 않고도 이어서 사용할 수 있다.

> pickle 파일은 역직렬화 과정에서 코드를 실행할 수 있으므로 신뢰할 수 있는 파일만 로드해야 한다.

## 2. 새로운 프로세스에서 모델 복원하기

다음으로 새로운 Python 프로세스에서 `model.pkl`을 불러왔다.

```bash
command python predict_from_pickle.py
```

결과는 다음과 같았다.

```text
학습 직후 테스트 정확도:       0.9333
pickle 복원 후 테스트 정확도:  0.9333
결과 일치 ✅
```

30개 중 한 샘플은 실제 클래스가 2였지만 모델은 1로 예측했다. 이것은 문제가 아니다. 이번 단계의 목적은 정확도 100%가 아니라 **저장하기 전과 복원한 후의 모델이 동일하게 작동하는지** 확인하는 것이기 때문이다.

## 3. scikit-learn 모델을 FastAPI로 제공하기

저장된 모델을 다른 프로그램에서도 사용할 수 있도록 FastAPI 서버에 연결했다.

서버는 시작 시 `model.pkl`을 한 번만 읽는다.

```text
서버 시작
→ model.pkl 1회 로드
→ 메모리에 모델 보관
→ 이후 모든 요청에서 재사용
```

요청마다 모델 파일을 다시 읽으면 디스크 접근과 복원 비용이 반복된다. FastAPI의 `lifespan`을 사용하면 서버가 시작될 때 모델을 준비하고 요청이 들어올 때마다 같은 객체를 사용할 수 있다.

서버 실행 명령은 다음과 같다.

```bash
command uvicorn app:app --host 127.0.0.1 --port 8331
```

`/health` 엔드포인트에서 서버와 모델 상태를 확인했다.

```json
{
  "status": "ok",
  "scikit_learn_version": "1.9.1",
  "model_type": "RandomForestClassifier",
  "n_estimators": 100
}
```

FastAPI가 자동 생성한 `/docs` 화면에서는 별도의 API 테스트 프로그램 없이 요청을 보낼 수 있었다.

```json
{
  "sepal_length_cm": 5.1,
  "sepal_width_cm": 3.5,
  "petal_length_cm": 1.4,
  "petal_width_cm": 0.2
}
```

서버는 다음과 같이 응답했다.

```json
{
  "predicted_class": "setosa",
  "probabilities": {
    "setosa": 1.0,
    "versicolor": 0.0,
    "virginica": 0.0
  }
}
```

<p align="center"><img src="{{ '/assets/images/model-serving-day1/02-sklearn-fastapi-setosa.png' | relative_url }}" alt="scikit-learn FastAPI의 setosa 예측 결과"></p>

versicolor와 virginica 측정값도 차례로 전송했다.

<p align="center"><img src="{{ '/assets/images/model-serving-day1/03-sklearn-fastapi-versicolor.png' | relative_url }}" alt="scikit-learn FastAPI의 versicolor 예측 결과"></p>

<p align="center"><img src="{{ '/assets/images/model-serving-day1/04-sklearn-fastapi-virginica.png' | relative_url }}" alt="scikit-learn FastAPI의 virginica 예측 결과"></p>

세 요청 모두 HTTP `200 OK`와 올바른 품종을 반환했다. 서버 시작 로그의 모델 로드 메시지는 한 번만 출력됐고, 세 예측은 메모리에 있는 같은 모델을 재사용했다.

## 4. PyTorch 모델을 TorchScript로 저장하기

다음으로 작은 신경망 `IrisNet`을 PyTorch로 학습했다. 총 200 epoch 동안 학습했으며, 테스트 데이터 30개 중 29개를 맞혔다.

```text
epoch  50/200  loss=0.0423
epoch 100/200  loss=0.0361
epoch 150/200  loss=0.0324
epoch 200/200  loss=0.0053

[Eager PyTorch] 테스트 정확도: 0.9667 (29/30)
```

학습된 모델을 `torch.jit.script`로 변환해 `iris_model.pt`로 저장했다.

```text
Eager PyTorch 정확도: 0.9667
TorchScript 정확도:   0.9667
```

새로운 프로그램에서 원래 `IrisNet` 클래스 없이 `.pt` 파일만 불러왔을 때도 정확도는 `0.9667`로 같았다.

```text
결과 일치 ✅ — IrisNet 클래스 없이도 컴파일된 .pt 파일만으로 동일하게 재현했다.
```

실행 중 `torch.jit.script`와 `torch.jit.load`가 향후 deprecated될 예정이라는 경고가 나타났다. 이것은 실행 실패가 아니라 최신 PyTorch에서 `torch.export` 사용을 권장한다는 안내다.

## 5. PyTorch FastAPI와 워커 성능 비교

TorchScript 모델과 입력값 변환 기준인 `scaler.npz`를 FastAPI 서버에서 함께 로드했다.

```text
원래 측정값
→ scaler로 값의 크기 조정
→ TorchScript 모델 추론
→ 품종과 확률 반환
```

API 응답에서 런타임이 TorchScript이고 모델이 정상적으로 로드됐음을 확인했다.

```json
{
  "status": "ok",
  "model_loaded": true,
  "runtime": "torchscript"
}
```

그다음 Uvicorn 워커를 1개와 4개로 실행해 같은 부하 테스트를 진행했다. 워커는 요청을 처리하는 별도의 서버 프로세스다.

### 워커 1개 결과

동시성 50 기준 결과는 다음과 같았다.

```text
처리량: 4910.5 req/s
p50:    9.56 ms
p99:   24.69 ms
실패:   0
```

### 워커 4개 결과

```text
처리량: 10361.1 req/s
p50:     4.61 ms
p99:     8.08 ms
실패:    0
```

두 결과를 비교하면 다음과 같다.

| 항목 | 워커 1개 | 워커 4개 | 변화 |
|---|---:|---:|---:|
| 처리량 | 4,910.5 req/s | 10,361.1 req/s | 약 2.1배 증가 |
| p50 | 9.56ms | 4.61ms | 약 52% 감소 |
| p99 | 24.69ms | 8.08ms | 약 67% 감소 |
| 실패 | 0 | 0 | 모두 성공 |

워커가 4개가 됐다고 성능이 정확히 4배가 되지는 않았다. 작업 분배 비용, CPU 코어 수, HTTP 처리 비용과 부하 테스트 프로그램의 한계가 있기 때문이다. 또한 각 워커가 모델을 따로 메모리에 올리므로 메모리 사용량도 증가한다.

중요한 것은 서버 코드를 바꾸지 않고 워커 수만 조정해 처리량과 지연시간이 크게 개선됐다는 점이다.

## 6. Keras 모델 저장과 구조 복원

세 번째 프레임워크로 Keras를 사용했다. TensorFlow 백엔드에서 다음 구조의 신경망을 학습했다.

```text
입력 4개 → Dense 16개 → Dense 8개 → 출력 3개
```

모델을 `iris_model.keras`로 저장한 뒤, 모델 구조를 다시 작성하지 않고 `load_model()`로 복원했다.

<p align="center"><img src="{{ '/assets/images/model-serving-day1/05-keras-model-restore.png' | relative_url }}" alt="Keras 모델 구조와 복원 결과"></p>

모델 요약에서 학습 대상 파라미터가 243개임을 확인했다.

```text
첫 번째 Dense 층: 80개
두 번째 Dense 층: 136개
출력 Dense 층:    27개
총 학습 대상:     243개
```

저장 전후 정확도도 동일했다.

```text
학습 직후 정확도: 1.0000
복원 후 정확도:   1.0000
결과 일치 ✅
```

`.keras` 파일에는 모델의 구조, 가중치와 설정이 함께 포함되므로 복원 프로그램에서 신경망을 다시 정의할 필요가 없었다.

## 7. Keras 모델을 FastAPI로 제공하기

마지막으로 Keras 모델과 `scaler.npz`를 FastAPI에 연결했다. 서버는 8341 포트에서 실행했다.

```bash
command uvicorn app:app --host 127.0.0.1 --port 8341
```

서버 시작 로그에서 Keras 모델이 한 번만 로드되는 것을 확인했다.

```text
[startup] Keras 모델 로드 완료 (keras 3.15.1, 백엔드=tensorflow)
```

세 가지 품종을 차례로 요청한 결과 모두 HTTP `200 OK`와 올바른 클래스를 반환했다.

<p align="center"><img src="{{ '/assets/images/model-serving-day1/06-keras-setosa.png' | relative_url }}" alt="Keras FastAPI의 setosa 예측 결과"></p>

<p align="center"><img src="{{ '/assets/images/model-serving-day1/07-keras-versicolor.png' | relative_url }}" alt="Keras FastAPI의 versicolor 예측 결과"></p>

<p align="center"><img src="{{ '/assets/images/model-serving-day1/08-keras-virginica.png' | relative_url }}" alt="Keras FastAPI의 virginica 예측 결과"></p>

첫 번째 요청의 지연시간은 약 78ms였고 이후 약 52ms, 38ms로 줄었다. TensorFlow가 첫 요청에서 내부 계산을 준비하는 워밍업 비용이 포함됐을 가능성이 있다. 다만 지연시간은 실행 환경마다 달라지므로 이번 실습에서는 수치 자체보다 세 요청이 모두 정상 처리됐다는 점을 성공 기준으로 삼았다.

## 세 가지 저장 형식 비교

| 프레임워크 | 저장 결과 | 이번 실습에서 확인한 점 |
|---|---|---|
| scikit-learn | `model.pkl` | 학습된 Python 모델 객체를 복원 |
| PyTorch | `iris_model.pt` | 원래 모델 클래스 없이 TorchScript 추론 |
| Keras | `iris_model.keras` | 모델 구조와 가중치를 함께 복원 |

저장 방식은 달랐지만 서버의 전체 동작은 같았다.

```text
모델 파일과 전처리 기준 준비
→ 서버 시작 시 한 번 로드
→ 입력 JSON 검증
→ 입력값 전처리
→ 모델 추론
→ 결과를 JSON으로 반환
```

## 실습을 통해 이해한 것

### 모델 정확도와 서버 성능은 다르다

정확도는 모델이 정답을 얼마나 잘 맞히는지 나타낸다. 처리량과 지연시간은 서버가 요청을 얼마나 빠르게 처리하는지 나타낸다. 정확한 모델이어도 서버가 느릴 수 있고, 빠른 서버여도 모델의 예측 품질이 낮을 수 있다.

### 모델과 전처리는 함께 배포해야 한다

PyTorch와 Keras API에서는 모델뿐 아니라 `scaler.npz`도 사용했다. 학습할 때 적용한 전처리와 서비스에서 적용하는 전처리가 다르면 같은 모델이어도 예측 결과가 달라질 수 있다.

### 모델은 요청마다 다시 로드하지 않는다

모든 FastAPI 실습에서 서버 시작 시 모델을 한 번만 로드했다. 이후 요청은 메모리에 있는 모델을 재사용했다. 모델이 커질수록 이 원칙은 더 중요해진다.

### 워커 증가는 처리량과 메모리 사이의 선택이다

워커 4개는 워커 1개보다 처리량이 높고 지연시간이 낮았다. 그러나 각 프로세스가 모델을 별도로 가지므로 메모리도 더 사용한다. 실제 운영에서는 CPU, 메모리, 모델 크기와 요청량을 보고 워커 수를 결정해야 한다.

## 마무리

Day 1 실습을 통해 노트북이나 학습 스크립트 안에 있던 모델이 실제 API 서비스가 되는 과정을 확인했다.

```text
학습 → 저장 → 복원 → API 서빙 → 부하 테스트
```

scikit-learn, PyTorch, Keras의 문법과 저장 파일은 달랐지만, **모델을 한 번 준비하고 여러 요청에서 재사용한다**는 서빙의 기본 원리는 동일했다.

다음 단계에서는 Airflow를 사용해 데이터 수집, 전처리, 학습, 평가와 서빙 작업을 순서와 의존성에 따라 자동으로 실행하는 오케스트레이션을 실습할 예정이다.
