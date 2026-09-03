---
layout: post
title: "은행 고객 이탈 예측: 모델 최적화와 SHAP 해석"
date: 2026-09-03 17:10:00 +0900
category: [ai, data-analysis]
tags: [Machine-Learning, Classification, Bank-Churn, Random-Forest, XGBoost, LightGBM, Hyperparameter-Tuning, SHAP, Python, learning-note]
---

이번 실습에서는 은행 고객의 정보와 거래 내역으로 고객 이탈 여부를 예측했다. 데이터 전처리와 파생변수 생성부터 Decision Tree, Random Forest, XGBoost, LightGBM 비교, 하이퍼파라미터 탐색, SHAP 해석까지 분류 모델 개발의 전체 흐름을 경험하는 것이 목표였다.

실행 결과가 포함된 전체 코드는 아래 노트북에서 확인할 수 있다.

- [실습 노트북 내려받기]({{ '/files/bank-churn-model-optimization.ipynb' | relative_url }})

## 1. 문제와 데이터 이해

예측 대상 `Exited`는 고객 이탈 여부를 나타낸다.

- `Exited = 1`: 이탈 고객
- `Exited = 0`: 유지 고객

입력 변수에는 신용점수, 국가, 성별, 나이, 계좌 잔액, 이용 상품 수, 신용카드 보유 여부, 활동 회원 여부, 추정 연봉, 계좌 개설일 등이 포함됐다.

고객 이름인 `Surname`과 식별자인 `CustomerId`는 모델 학습에서 제외했다. 이 값들은 고객을 구분할 수는 있지만 새로운 고객에게 일반화할 수 있는 행동 특성으로 보기 어렵기 때문이다.

## 2. 파생변수와 전처리

날짜를 변환한 뒤 기준일과 계좌 개설일의 차이로 가입 기간을 계산했다. 기존 변수 사이의 관계를 표현하기 위해 다음 변수도 추가했다.

| 파생변수 | 의미 |
|:--|:--|
| `accountDuration_days` | 계좌 개설 후 경과 일수 |
| `salaryPerAge` | 나이 대비 추정 연봉 |
| `balancePerProduct` | 이용 상품 하나당 계좌 잔액 |
| `creditScorePerAge` | 나이 대비 신용점수 |
| `isZeroBalance` | 잔액이 0인지 여부 |
| `tenureAgeRatio` | 생애 기간 대비 은행 거래 기간 |
| `engagementScore` | 상품·카드·활동 여부를 결합한 참여도 |

데이터는 Train 60%, Validation 20%, Test 20%로 나눴다. 연속형 결측치는 Train 중앙값, 범주형 결측치는 `Unknown`으로 대체했다. 이후 부호 보존 로그 변환, Min-Max Scaling, One-Hot Encoding을 적용했다.

```text
원본 데이터
  → 불필요 변수 제거와 날짜 변환
  → 파생변수 생성
  → Train / Validation / Test 분리
  → Train 기준 결측치 처리와 Scaling
  → 범주형 변수 One-Hot Encoding
  → 변수 선택
  → 모델 학습과 평가
```

중앙값과 스케일은 반드시 Train 데이터에서만 학습한 뒤 Validation과 Test에 적용했다. 전체 데이터를 이용해 전처리 기준을 계산하면 평가 데이터의 정보가 모델 학습에 들어가는 데이터 누수가 발생할 수 있다.

## 3. 어떤 평가 지표를 볼 것인가

고객 이탈처럼 클래스 비율이 불균형한 문제에서는 Accuracy만으로 모델을 평가하기 어렵다. 유지 고객만 예측해도 Accuracy가 높게 나올 수 있기 때문이다.

이번 실습에서는 다음 지표를 함께 확인했다.

- **Recall**: 실제 이탈 고객 중 모델이 찾아낸 비율
- **Precision**: 모델이 이탈로 예측한 고객 중 실제 이탈 고객의 비율
- **F1 Score**: Precision과 Recall의 조화평균
- **ROC-AUC**: 여러 임계값에서 이탈 고객을 유지 고객보다 높게 평가하는 능력

이탈 방지 캠페인에서 놓치는 고객의 비용이 크다면 Recall을, 접촉 비용이 크다면 Precision을 더 중요하게 볼 수 있다.

## 4. Decision Tree로 고객군 규칙 확인

먼저 깊이가 3인 Decision Tree를 학습했다.

| 데이터 | Accuracy | F1 | Recall | Precision | AUC |
|:--|--:|--:|--:|--:|--:|
| Validation | 0.7905 | 0.5417 | 0.4906 | 0.6047 | 0.7774 |
| Test | 0.8595 | 0.4957 | 0.4265 | 0.5918 | 0.7580 |

로그 변환과 Scaling을 적용하지 않은 원 단위 데이터로 리프 노드를 비교하자 다음과 같은 차이가 나타났다.

- 저위험 리프: 183명, 실제 이탈률 **5.5%**
- 고위험 리프: 43명, 실제 이탈률 **55.8%**

고위험 집단의 이탈률이 저위험 집단보다 약 10배 높았다. 얕은 Decision Tree는 최고 성능 모델이라기보다 위험 고객군을 사람이 이해할 수 있는 조건으로 설명하는 데 유용했다. 단, 표본 수가 작은 리프는 우연한 패턴일 수 있으므로 이탈률과 고객 수를 함께 확인해야 한다.

## 5. Random Forest 성능

여러 트리의 예측을 결합하는 Random Forest는 단일 Decision Tree보다 안정적인 결과를 보였다.

| 데이터 | Accuracy | F1 | Recall | Precision | AUC |
|:--|--:|--:|--:|--:|--:|
| Validation | 0.8119 | 0.6520 | 0.6981 | 0.6116 | 0.8496 |
| Test | 0.8310 | 0.5298 | 0.5882 | 0.4819 | 0.8229 |

Test AUC가 Decision Tree의 0.7580에서 0.8229로 상승했다. `class_weight="balanced"`를 적용해 상대적으로 적은 이탈 고객의 오분류에 더 큰 가중치를 부여했다.

## 6. 수동 설정 모델 비교

Random Forest의 깊이를 달리하고 XGBoost와 LightGBM을 추가해 Validation 데이터에서 비교했다.

| 모델 | Accuracy | F1 | Recall | Precision | AUC |
|:--|--:|--:|--:|--:|--:|
| XGBoost | 0.8143 | 0.5412 | 0.4340 | **0.7188** | **0.8661** |
| Random Forest Deeper | **0.8167** | **0.6516** | 0.6792 | 0.6261 | 0.8536 |
| Random Forest Shallow | 0.7857 | 0.6371 | **0.7453** | 0.5563 | 0.8457 |
| LightGBM | 0.7905 | 0.6000 | 0.6226 | 0.5789 | 0.8380 |

Validation AUC는 XGBoost가 0.8661로 가장 높았다. 하지만 Recall은 0.4340으로 가장 낮았다. 반대로 얕은 Random Forest는 AUC가 조금 낮았지만 Recall이 0.7453으로 가장 높았다.

이 결과는 “가장 좋은 모델”이 업무 목적에 따라 달라진다는 점을 보여준다. 이탈 가능성이 높은 고객의 순위를 잘 정하는 것이 목적이라면 AUC를, 실제 이탈자를 최대한 놓치지 않는 것이 목적이라면 Recall을 우선할 수 있다.

## 7. 세 가지 하이퍼파라미터 탐색 비교

Random Forest의 트리 수, 최대 깊이, 최소 분할 표본 수, 리프 최소 표본 수를 Bayesian Search, Random Search, Grid Search로 탐색했다.

| 탐색 방법 | CV AUC | Validation AUC | Test AUC |
|:--|--:|--:|--:|
| Bayesian Search | 0.8249 | **0.8538** | 0.8258 |
| Random Search | 0.8245 | 0.8424 | 0.8207 |
| Grid Search | **0.8278** | 0.8476 | **0.8261** |

세 방법의 Test AUC 차이는 크지 않았다. Grid Search가 0.8261로 가장 높았지만 Bayesian Search의 0.8258과 사실상 비슷한 수준이었다.

이번 결과만으로 특정 탐색 방법이 항상 우수하다고 결론 내릴 수는 없다. 탐색 방법보다 탐색 범위, 반복 횟수, 교차검증 설계와 평가 지표가 결과에 큰 영향을 준다.

## 8. Bayesian Search 모델 비교

Random Forest, XGBoost, LightGBM에 Bayesian Search를 적용했다.

| 모델 | CV AUC | Validation AUC | Validation F1 |
|:--|--:|--:|--:|
| Random Forest | **0.8255** | **0.8539** | **0.6463** |
| XGBoost | 0.8191 | 0.8378 | 0.5529 |
| LightGBM | 0.8221 | 0.8299 | 0.5674 |

수동 설정에서는 XGBoost의 AUC가 가장 높았지만 Bayesian Search에서는 Random Forest가 가장 안정적인 결과를 보였다. 알고리즘 이름만으로 성능을 판단할 수 없으며, 전처리와 탐색 공간이 모델 성능에 함께 영향을 준다는 점을 확인했다.

## 9. SHAP으로 예측 이유 설명하기

Bayesian Search로 최적화한 Random Forest를 SHAP으로 분석했다. Test 데이터에서 평균 절대 SHAP 값이 가장 높은 변수는 다음과 같았다.

1. `Age`
2. `creditScorePerAge`
3. `IsActiveMember`

SHAP 값은 변수의 영향 크기뿐 아니라 방향도 보여준다.

- 양의 SHAP: 이탈 예측을 높이는 방향
- 음의 SHAP: 유지 예측을 높이는 방향
- 절대값이 클수록 해당 고객의 예측에 미친 영향이 큼

### 20번째 고객의 예측 근거

Test 데이터의 20번째 고객은 실제 이탈 고객이었고 모델의 예측 이탈확률은 **82.1%**였다.

이 고객의 이탈 가능성을 높인 주요 요인은 활동 회원이 아니라는 점, 고객 참여도, 나이 대비 신용점수, 나이였다. 특히 `IsActiveMember`가 이탈 방향으로 가장 크게 작용했다.

SHAP을 이용하면 “이 고객은 이탈할 가능성이 높다”에서 끝나지 않고 어떤 특성이 그 판단에 기여했는지 설명할 수 있다. 실제 캠페인에서는 이러한 설명을 상담 전략이나 고객 세분화에 활용할 수 있다.

## 10. 이번 실습에서 배운 점

첫째, 불균형 분류 문제에서는 Accuracy 하나만 보면 안 된다. Recall, Precision, F1, AUC를 업무 목적과 함께 평가해야 한다.

둘째, 가장 높은 AUC를 기록한 모델이 모든 목적에서 최선은 아니다. XGBoost는 높은 AUC와 Precision을 보였지만 Recall이 낮았고, Random Forest는 더 많은 이탈 고객을 찾았다.

셋째, 결측치 대체와 Scaling 같은 전처리는 Train 데이터에서만 학습해야 한다. 평가 데이터의 정보가 전처리 기준에 섞이면 실제보다 성능이 좋아 보일 수 있다.

넷째, 파생변수는 많이 만드는 것보다 고객 행동을 제대로 표현하고 교차검증에서 효과를 확인하는 것이 중요하다.

다섯째, 모델 성능과 설명 가능성을 함께 고려해야 한다. Boosting과 Random Forest로 성능을 비교하고, Decision Tree와 SHAP으로 집단 및 개인 수준의 판단 근거를 설명할 수 있었다.

## 11. 다음 개선 방향

실제 서비스에 적용한다면 다음 작업을 추가할 수 있다.

- 이탈 비율을 유지하는 계층화 데이터 분할
- 업무 비용에 맞춘 분류 임계값 최적화
- Precision-Recall Curve와 PR-AUC 평가
- 시간 순서에 따른 Out-of-Time Validation
- 예측확률 보정과 Calibration Curve 확인
- 파생변수 적용 전후의 교차검증 성능 비교
- 월별 데이터 분포 및 SHAP 중요도 변화 감시

## 마무리

이번 실습에서는 은행 고객 이탈 예측을 통해 데이터 준비, Feature Engineering, 모델 비교, 하이퍼파라미터 최적화, 설명 가능한 AI까지 전체 흐름을 살펴봤다.

수동 모델 중에서는 XGBoost가 Validation AUC 0.8661로 가장 높았고, Bayesian Search 모델 비교에서는 Random Forest가 가장 안정적인 결과를 보였다. SHAP 분석에서는 나이, 나이 대비 신용점수, 활동 회원 여부가 이탈 예측에 중요한 변수로 나타났다.

좋은 예측 모델은 단순히 점수가 높은 모델이 아니다. 새로운 데이터에서도 안정적으로 작동하고, 업무 목적에 적합한 고객을 찾아내며, 그 판단 이유를 설명할 수 있어야 한다.
