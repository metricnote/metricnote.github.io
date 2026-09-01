---
layout: post
title: "Isolation Forest로 주택 데이터 이상치 찾고 설명하기"
date: 2026-09-01 17:30:00 +0900
category: [ai, data-analysis]
tags: [Machine-Learning, Anomaly-Detection, Isolation-Forest, Decision-Tree, Random-Forest, Feature-Importance, Python, learning-note]
---

이번 실습에서는 캘리포니아 주택 데이터의 지역별 통계를 이용해 이상치를 찾았다. 정답 라벨 없이 Isolation Forest로 이상 후보를 만든 뒤, Decision Tree와 Random Forest를 별도의 **설명 모델**로 붙여 다음 두 질문에 답했다.

1. 어떤 조건을 가진 지역에서 이상치 비율이 가장 높거나 낮은가?
2. 정상과 이상을 구분할 때 어떤 변수가 크게 작용했는가?

딥러닝 강의에서 배운 모델, 학습, 평가의 틀로 돌아보니 중요한 점은 알고리즘 이름보다 역할을 구분하는 일이었다. Isolation Forest는 이상 후보를 발견하고, 얕은 Decision Tree는 그 결과를 규칙으로 요약하며, Random Forest는 변수의 상대적 영향도를 보여준다.

## 1. 이상치 탐지는 무엇이 다른가

일반적인 분류는 정상과 이상이라는 정답을 보고 경계를 학습한다. 하지만 현실에서는 이상 사례가 드물고 라벨을 만들기도 어렵다. 이번 실습처럼 정답이 없는 데이터에서는 **비지도 이상치 탐지**가 출발점이 될 수 있다.

Isolation Forest의 핵심 아이디어는 단순하다.

- 변수를 하나 고르고 임의의 기준으로 데이터를 나눈다.
- 이 과정을 여러 번 반복해 각 관측치가 얼마나 빨리 혼자 분리되는지 본다.
- 적은 분할만으로 고립되는 관측치일수록 이상일 가능성이 높다고 판단한다.

밀집된 정상 영역의 점은 주변에 비슷한 점이 많아 고립시키기 어렵다. 반대로 드문 영역에 있는 점은 비교적 빠르게 분리된다. 여러 개의 무작위 트리를 함께 사용하는 이유는 한 번의 우연한 분할에 의존하지 않기 위해서다.

## 2. 데이터와 실습 조건

분석 단위는 캘리포니아의 지역 단위 주택 통계이며 총 20,640개 행을 사용했다. 원래 실습의 다섯 변수에 위치와 주택 연식을 추가해 총 여덟 변수를 분석했다.

| 변수 | 의미 |
|:--|:--|
| `median_income` | 중위소득 |
| `total_rooms` | 전체 방 수 |
| `population` | 인구수 |
| `households` | 가구수 |
| `median_house_value` | 주택 중위가격 |
| `latitude` | 위도 |
| `longitude` | 경도 |
| `housing_median_age` | 주택 중위 연식 |

전처리와 모델 조건은 기존 실습과 동일하게 유지했다.

```text
연속형 결측값 → 중앙값 대체
8개 변수       → StandardScaler로 Z-score 변환
Isolation Forest
  - n_estimators = 100
  - contamination = 0.05
  - random_state = 42
```

`contamination=0.05`는 모델이 전체 데이터 중 약 5%를 이상치로 판정하도록 기준을 정한다. 따라서 1,032개가 Anomaly, 19,608개가 Normal로 분류됐다.

![Isolation Forest가 만든 정상·이상 레이블 분포]({{ '/assets/images/ml-anomaly-detection/03-label-distribution.png' | relative_url }})

여기서 5%는 데이터에서 저절로 발견된 절대적인 이상 비율이 아니다. 분석자가 지정한 가정이다. 운영 환경에서는 조사 가능한 경보 수, 실제 장애율, 오탐 비용 등을 바탕으로 이 값을 조정해야 한다.

## 3. 시나리오 1: 이상치가 많은 집단의 규칙 찾기

Isolation Forest는 이상 여부를 만들지만 결과를 간단한 업무 규칙으로 설명하기는 쉽지 않다. 그래서 이 레이블을 목표값으로 두고 깊이 3의 Decision Tree를 학습했다.

![Isolation Forest 레이블을 설명하는 깊이 3의 Decision Tree]({{ '/assets/images/ml-anomaly-detection/01-decision-tree-rules.png' | relative_url }})

이 트리는 원래의 이상치 탐지 모델을 대체하는 모델이 아니다. 복잡한 결과를 사람이 읽을 수 있는 조건으로 근사하는 **대리 모델(surrogate model)**이다. 각 잎 노드에서 다음 식으로 Anomaly 비율을 계산했다.

```text
Anomaly 비율 = Anomaly 건수 / (Normal 건수 + Anomaly 건수)
```

### Anomaly 비율이 가장 높은 집단

```text
total_rooms > 7,064
AND population > 4,015.5
AND households > 1,362
```

이 조건에 해당하는 401개 지역 중 395개가 Anomaly로 분류돼 비율은 **98.50%**였다. 방 수, 인구수, 가구수가 모두 큰 대규모 지역이 데이터의 일반적인 패턴에서 벗어날 가능성이 높았다는 뜻이다.

### Anomaly 비율이 가장 낮은 집단

```text
total_rooms <= 6,120
AND median_income <= 10.3126
```

이 조건에 해당하는 19,331개 지역 중 174개가 Anomaly로 분류돼 비율은 **0.90%**였다. 다수의 일반적인 지역이 이 넓은 조건 안에 포함됐다.

트리의 임계값은 자연법칙이 아니다. 이번 데이터와 모델 설정을 간단히 설명하기 위해 만들어진 경계다. 데이터가 바뀌거나 `contamination`, 트리 깊이, 난수 시드가 바뀌면 규칙도 달라질 수 있다.

## 4. 시나리오 2: 어떤 변수가 구분에 중요했나

두 번째 질문에는 Random Forest의 Feature Importance를 사용했다. Normal과 Anomaly를 구분하는 여러 결정 트리에서 어떤 변수가 불순도를 많이 줄였는지 합산한 값이다.

![Random Forest 변수 중요도]({{ '/assets/images/ml-anomaly-detection/02-random-forest-feature-importance.png' | relative_url }})

| 순위 | 변수 | 중요도 |
|:--:|:--|--:|
| 1 | `total_rooms` | 0.3073 |
| 2 | `population` | 0.1693 |
| 3 | `households` | 0.1556 |
| 4 | `median_income` | 0.1537 |
| 5 | `median_house_value` | 0.1081 |
| 6 | `housing_median_age` | 0.0381 |
| 7 | `longitude` | 0.0356 |
| 8 | `latitude` | 0.0324 |

가장 큰 변수는 `total_rooms`였고, `population`, `households`, `median_income`이 뒤를 이었다. 상위 세 변수는 모두 지역의 **규모**와 관련되어 있다. 이번 모델은 지리적 위치보다 큰 방 수와 인구·가구 규모의 극단적인 조합에 더 민감하게 반응한 것으로 해석할 수 있다.

다만 Feature Importance가 높다고 해서 그 변수가 이상치를 **발생시켰다**는 뜻은 아니다. 변수 간 상관관계가 있으면 중요도가 나뉘거나 한 변수에 몰릴 수 있고, 불순도 기반 중요도는 값의 분할 후보가 많은 변수에 유리할 수도 있다. 필요하다면 Permutation Importance나 SHAP를 함께 확인해야 한다.

## 5. 딥러닝 학습 원리와 연결해 본 세 가지

이번에는 딥러닝 모델을 학습하지 않았지만, 강의에서 배운 학습 원리는 그대로 적용됐다.

### 모델과 알고리즘, 설명 도구의 역할을 나누기

모델은 입력에서 출력을 만드는 구조이고, 알고리즘은 그 모델이 규칙을 찾는 방법이다. 이번 흐름에서는 Isolation Forest가 이상 후보를 만드는 주 모델이고, Decision Tree와 Random Forest는 결과를 이해하기 위한 설명 도구다. 서로 다른 결과를 하나의 모델 성능처럼 섞어 해석하면 안 된다.

### 하이퍼파라미터는 분석자의 가정이다

딥러닝에서 학습률, 배치 크기, 에포크가 학습 결과를 바꾸듯 이상치 탐지에서는 `contamination`, 트리 수, 표본 추출 방식이 결과를 바꾼다. 특히 이번 결과의 5%는 정답이 아니라 설정값이다. 설정을 바꾸며 탐지 결과와 업무 비용이 얼마나 달라지는지 민감도 분석이 필요하다.

### 학습 결과와 실제 성능을 구분하기

강의에서 Validation과 Test를 분리해 과적합을 확인했듯, 이상치 탐지도 실제 정답이 있다면 별도의 검증이 필요하다. Precision, Recall, PR-AUC처럼 불균형 데이터에 적합한 지표를 사용해야 한다. 정답이 없다면 표본 조사, 도메인 전문가 검토, 시간에 따른 경보 안정성으로 결과를 검증할 수 있다.

## 6. AIOps에 적용한다면

주택 데이터의 한 행을 서버나 서비스의 일정 시간 구간으로 바꾸면 같은 흐름을 AIOps에도 적용할 수 있다.

```text
CPU·메모리·지연시간·오류율 수집
  → 결측·스케일 처리
  → Isolation Forest로 이상 후보 탐지
  → Tree 규칙과 변수 중요도로 원인 후보 설명
  → 운영자 확인 결과를 다시 기준 조정에 반영
```

운영 데이터에서는 특히 다음을 주의해야 한다.

- 장애가 아닌 배포, 이벤트, 월말 처리도 평소와 다르면 이상으로 탐지될 수 있다.
- 시간대와 요일에 따른 정상 패턴을 반영하지 않으면 오탐이 늘어난다.
- 새 서비스나 트래픽 변화로 데이터 분포가 바뀌면 모델을 다시 점검해야 한다.
- 이상 점수만 전달하기보다 영향 변수와 비교 기준을 함께 제공해야 대응이 빨라진다.

## 마무리

이번 실습에서 가장 큰 배움은 이상치를 찾는 것과 설명하는 것이 별개의 문제라는 점이었다. Isolation Forest는 20,640개 지역 중 설정한 5%를 이상 후보로 골랐다. Decision Tree는 대규모 방 수·인구·가구 조건에서 Anomaly 비율이 98.50%까지 높아지는 규칙을 보여줬고, Random Forest는 `total_rooms`, `population`, `households`가 구분에 크게 기여했음을 나타냈다.

하지만 이 결과는 곧바로 오류나 문제 지역을 뜻하지 않는다. 이상치는 **평소와 다른 관측치**이며, 원인을 확인할 출발점이다. 다음 단계에서는 `contamination`별 민감도 비교, Permutation Importance, 실제 검토 라벨을 이용한 평가까지 확장해 보고 싶다.

데이터 출처: [California Housing 데이터셋](https://github.com/ageron/handson-ml2/tree/master/datasets/housing)
