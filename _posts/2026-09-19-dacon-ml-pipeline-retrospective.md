---
layout: post
title: "DACON 고객 분류 프로젝트를 다시 보며 배운 ML Pipeline Validation"
date: 2026-09-19
categories:
  - Data Science
  - Machine Learning
tags:
  - DACON
  - PyTorch
  - Transformers
  - Validation
  - Data Processing
---

DACON 고객 세그먼트 분류 프로젝트에서는
여러 종류의 고객 데이터를 통합하고,
A~E 다섯 개의 고객 Segment를 예측하는
Transformer 기반 분류 pipeline을 실험했다.

당시에는 모델 구조와 class imbalance 대응 방법을
계속 추가하는 데 많은 시간을 사용했다.

프로젝트가 끝난 뒤 코드를 다시 검토하면서,
좋은 모델을 선택하는 것만큼
데이터가 학습과 평가 과정에 어떻게 들어가는지
검증하는 것이 중요하다는 점을 다시 확인하게 되었다.

이 글에서는 당시 사용했던 방법 자체보다,
프로젝트를 다시 보며 발견한 문제와
그 과정에서 배운 점을 정리하려고 한다.

## 여러 데이터를 고객 단위로 통합하기

원본 데이터는 하나의 테이블로 구성되어 있지 않았다.

여러 Parquet 파일에 나뉜 정보를
고객 ID를 기준으로 모아
하나의 모델 입력으로 만들어야 했다.

구현한 pipeline에서는 먼저
`ID`와 `Segment`가 존재하는 데이터에서
고객과 정답 label을 수집했다.

그다음 동일한 고객 ID가 등장하는 여러 파일을 탐색하면서
`ID`와 target인 `Segment`를 제외한 값을 모아
고객별 하나의 입력 데이터로 통합했다.

대략적인 흐름은 다음과 같았다.

1. 여러 Parquet 파일 로딩
2. 고객 ID와 Segment 수집
3. 동일 ID의 여러 데이터 탐색
4. feature들을 고객별 하나의 입력으로 통합
5. Segment A~E를 classification label로 사용
6. Transformer 입력으로 변환

이 과정에서 처음으로
"모델을 학습하기 전에 데이터를 어떤 단위로 정의할 것인가"가
중요한 설계 문제라는 것을 경험했다.

## 고객 데이터를 Text Classification 문제로 변환하기

고객별로 모은 여러 feature를
하나의 긴 text 형태로 변환한 뒤,
`klue/roberta-large`를 이용해
5-class classification을 수행했다.

이 방식의 장점은
서로 다른 형태의 데이터를 하나의 입력 표현으로 통합해
Transformer를 적용할 수 있다는 점이었다.

하지만 프로젝트를 다시 보면
명확한 한계도 있었다.

모델 입력 길이를 128 token으로 제한했기 때문에,
여러 데이터 소스를 하나의 긴 text로 합치더라도
뒤쪽 정보는 truncation되어
모델에 전달되지 않을 가능성이 있었다.

즉 데이터를 많이 넣는 것과
모델이 실제로 그 정보를 사용하는 것은
같은 문제가 아니었다.

만약 다시 구현한다면
모든 값을 단순 연결하기보다
중요 feature를 명시적으로 선택하거나,
구조화된 feature와 text feature를 분리하는 방법을
먼저 검토할 것 같다.

## Class Imbalance에 대응하기 위한 실험

Segment별 데이터 수의 차이를 고려하기 위해
여러 가지 방법을 실험했다.

- Class-weighted loss
- Focal loss
- Text augmentation
- Embedding-space ADASYN
- Minority-class focused model
- Prediction threshold 조정
- Main model과 minority model의 ensemble

당시에는
"소수 클래스의 데이터를 더 잘 맞히기 위해
어떤 방법을 추가할 수 있을까?"를 중심으로 접근했다.

Class별 F1과 Macro F1을 계산하고,
전체 accuracy뿐 아니라
각 클래스의 성능을 따로 확인하는 평가 코드도 구성했다.

이 과정 자체는
불균형 데이터 문제를 다뤄보는 좋은 경험이었다.

하지만 나중에 코드를 다시 보면서
방법을 많이 추가하는 것보다 먼저 확인해야 할 것이 있다는 것을 알게 되었다.

## Data Split보다 Augmentation이 먼저 실행된 문제

가장 중요하게 다시 보게 된 부분은
data augmentation의 실행 시점이었다.

당시 pipeline에서는 고객 데이터를 통합하는 과정에서
text augmentation과 oversampling을 먼저 수행한 뒤,
그 결과를 train과 validation으로 나누는 구조가 포함되어 있었다.

이 방식에서는
원본 고객 데이터와 그 고객에서 만들어진 augmented sample이
서로 다른 split에 들어갈 가능성이 있다.

그렇게 되면 validation set이
완전히 독립적인 데이터라고 보기 어려워질 수 있다.

더 안전한 구조는 다음과 같다.

1. 원본 고객 ID 단위로 train / validation 분리
2. validation 데이터는 그대로 유지
3. train 데이터에만 augmentation / oversampling 적용
4. 모델 학습
5. untouched validation set으로 평가

이 경험 이후에는
augmentation 기법 자체보다
"그 기법을 pipeline의 어느 단계에 적용하는가"가
더 중요할 수 있다는 점을 의식하게 되었다.

## Cross-validation이 있다고 해서 자동으로 안전한 것은 아니다

프로젝트에서는 5-fold cross-validation도 구현했다.

처음에는 여러 fold에서 평가하면
단일 train/validation split보다
더 신뢰할 수 있는 결과를 얻을 수 있다고 생각했다.

하지만 cross-validation 역시
그 전에 데이터가 어떻게 만들어졌는지가 잘못되어 있다면
문제를 해결해주지 못한다.

이미 augmentation된 데이터를 fold로 나눈다면,
각 fold의 독립성이 충분하지 않을 수 있다.

결국 중요한 것은
"cross-validation을 사용했는가"보다
"어떤 데이터 단위로 split했고,
split 이전에 어떤 preprocessing이 수행되었는가"였다.

## Train과 Test의 Preprocessing은 같은 함수라고 끝이 아니다

코드를 다시 검토하면서
training과 inference preprocessing을
더 명확하게 분리할 필요도 확인했다.

학습 데이터에는 정답인 `Segment`가 존재하지만
test 데이터에는 당연히 Segment가 존재하지 않는다.

그런데 학습용 데이터 통합 함수가
`ID`와 `Segment`가 함께 존재하는 데이터를 기준으로
처리 대상을 찾도록 만들어져 있다면,
그 함수를 그대로 test 데이터에도 사용하는 것은 안전하지 않다.

공통 feature 처리 로직은 재사용하더라도,

- Train preprocessing
- Test / inference preprocessing

의 entry point와 검증 조건은
분리하는 편이 더 명확하다.

이 부분을 다시 보면서
training code가 실행되는 것과
실제 inference pipeline이 끝까지 일관되게 동작하는 것은
별개의 검증 대상이라는 것을 배웠다.

## 초기 실험에서 가장 크게 놓쳤던 것

초기 코드 중에는 실제 Segment label이 연결되기 전,
임시로 random label을 생성해
pipeline을 실행했던 버전도 있었다.

당시에는 모델 학습 코드를 먼저 실행해보는 것에 집중했지만,
random label을 사용하는 상태에서는
validation metric 자체가 실제 문제의 성능을 의미할 수 없다.

이후 실제 Segment를 데이터에서 가져오는 구조로 수정했지만,
이 경험은 프로젝트를 다시 평가할 때
가장 중요한 기준 중 하나가 되었다.

모델이 정상적으로 학습되고
숫자로 metric이 출력된다는 사실만으로는
그 metric이 의미 있다고 볼 수 없다.

먼저 확인해야 하는 것은 다음과 같다.

- Label은 실제 정답과 연결되어 있는가?
- Train과 validation 데이터는 독립적인가?
- Preprocessing 과정에서 leakage가 없는가?
- 평가 대상과 실제 inference 대상의 처리 과정이 일관적인가?
- Metric이 실제 해결하려는 문제를 측정하고 있는가?

## 그래서 성능 수치를 포트폴리오에 쓰지 않는 이유

이 프로젝트에서는
Macro F1, class-wise F1, cross-validation 등
여러 평가 방법을 구현했다.

하지만 프로젝트 이후 pipeline을 다시 검토하면서
당시의 metric을 현재 포트폴리오에서
성능 성과로 제시하지 않기로 했다.

검증 과정에 개선해야 할 부분이 있는 상태에서
숫자만 가져와 성능처럼 제시하는 것은
프로젝트를 정확하게 설명하는 방법이 아니라고 판단했기 때문이다.

대신 이 프로젝트에서 얻은 경험은
다음과 같은 부분에 있다고 생각한다.

- 여러 데이터 소스를 고객 단위로 통합한 경험
- Transformer 기반 classification pipeline 구현
- Class imbalance 대응 방법 실험
- Class-wise metric과 Macro F1 평가
- Cross-validation 구현
- 기존 pipeline을 다시 검토하고 문제점을 찾은 경험

## 프로젝트를 다시 보며 배운 점

당시에는 새로운 모델이나
augmentation, ensemble 같은 방법을 추가하면
성능이 좋아질 것이라는 생각을 많이 했다.

하지만 프로젝트를 다시 검토하면서
머신러닝에서 더 먼저 확인해야 할 것은
모델보다 데이터와 평가 구조라는 점을 배웠다.

특히 다음 순서를 중요하게 생각하게 되었다.

1. 문제와 target을 정확히 정의한다.
2. 원본 데이터 단위로 train / validation을 먼저 분리한다.
3. preprocessing과 augmentation의 적용 범위를 제한한다.
4. validation set을 독립적으로 유지한다.
5. metric이 실제 문제를 제대로 측정하는지 확인한다.
6. 그 이후에 모델과 학습 방법을 비교한다.

DACON 프로젝트는
완성도 높은 최종 모델을 만들었다는 의미보다,
내가 머신러닝 pipeline을 직접 구성하고
나중에 그 pipeline을 비판적으로 다시 검토해보면서
validation의 중요성을 배운 프로젝트로 남았다.