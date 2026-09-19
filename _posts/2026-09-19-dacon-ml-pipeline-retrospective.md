---
layout: post
title: "DACON 고객 분류 프로젝트를 다시 보며 배운 Validation의 중요성"
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
데이터가 학습과 평가 과정에 어떻게 들어가는지를
검증하는 것이 중요하다는 점을 다시 확인하게 되었다.

## 여러 데이터를 고객 단위로 통합하기

원본 데이터는 하나의 테이블이 아니라
여러 Parquet 파일에 나뉘어 있었다.

이를 고객 ID를 기준으로 모아
하나의 모델 입력으로 만드는 과정을 구현했다.

대략적인 흐름은 다음과 같았다.

1. 여러 Parquet 파일 로딩
2. 고객 ID와 Segment 수집
3. 동일 ID가 등장하는 데이터 탐색
4. 고객별 feature 통합
5. Segment A~E를 classification label로 사용
6. Transformer 입력으로 변환

이 과정에서
모델을 학습하기 전에
"데이터를 어떤 단위로 정의할 것인가" 자체가
중요한 설계 문제라는 것을 경험했다.

## 고객 데이터를 Text Classification 문제로 변환하기

고객별로 모은 여러 feature를
하나의 긴 text 형태로 변환한 뒤,
`klue/roberta-large`를 이용해
5-class classification을 수행했다.

서로 다른 형태의 데이터를
하나의 입력 표현으로 만들 수 있다는 장점이 있었지만,
다시 보면 한계도 명확했다.

모델 입력 길이를 128 token으로 제한했기 때문에
여러 데이터를 길게 연결하더라도
뒤쪽 정보는 truncation되어
실제로 모델에 전달되지 않을 수 있었다.

즉 데이터를 많이 넣는 것과
모델이 그 정보를 실제로 사용하는 것은
같은 문제가 아니었다.

다시 구현한다면
모든 값을 단순 연결하기보다
중요 feature를 먼저 선택하거나,
구조화된 feature와 text feature를
분리하는 방법을 검토할 것 같다.

## Class Imbalance에 대응하기 위한 실험

Segment별 데이터 수 차이를 고려하기 위해
여러 방법을 실험했다.

- Class-weighted loss
- Focal loss
- Text augmentation
- Embedding-space ADASYN
- Minority-class focused model
- Prediction threshold 조정
- Main model과 minority model의 ensemble

평가에서는 전체 accuracy뿐 아니라
Class별 F1과 Macro F1도 계산했다.

이 과정은 불균형 데이터를 다뤄보는 경험이 되었지만,
나중에 코드를 다시 보면서
기법을 추가하기 전에
pipeline 자체를 먼저 검증해야 한다는 점을 알게 되었다.

## Split과 Augmentation의 순서

가장 중요하게 다시 본 부분은
data augmentation의 실행 시점이었다.

당시 pipeline에는
고객 데이터를 통합하는 과정에서
augmentation과 oversampling을 수행한 뒤,
그 결과를 train과 validation으로 나누는 구조가 포함되어 있었다.

이 경우 원본 데이터와
그 원본에서 만들어진 augmented sample이
서로 다른 split에 들어갈 가능성이 있다.

그렇게 되면 validation set의 독립성이
약해질 수 있다.

더 안전한 구조는 다음과 같다.

1. 원본 고객 ID 단위로 train / validation 분리
2. validation 데이터는 그대로 유지
3. train 데이터에만 augmentation / oversampling 적용
4. 모델 학습
5. untouched validation set으로 평가

프로젝트에서는 5-fold cross-validation도 구현했지만,
cross-validation 역시
그 이전의 데이터 생성 과정이 잘못되어 있다면
자동으로 문제를 해결해주지는 않는다.

결국 중요한 것은
cross-validation을 사용했는가보다
어떤 단위로 split했고
split 이전에 어떤 preprocessing이 수행되었는가였다.

## Train과 Test Preprocessing

training과 inference preprocessing도
더 명확하게 분리할 필요가 있었다.

학습 데이터에는 정답인 `Segment`가 존재하지만
test 데이터에는 정답 label이 존재하지 않는다.

그런데 학습용 데이터 통합 로직이
`ID`와 `Segment`가 함께 존재하는 데이터를 기준으로
처리 대상을 찾는다면,
같은 함수를 그대로 inference에 사용하는 것은
안전하다고 보기 어렵다.

공통 feature 처리 코드는 재사용하더라도,

- Train preprocessing
- Test / inference preprocessing

의 entry point와 검증 조건은
분리하는 편이 더 명확하다.

이 부분을 통해
training code가 실행된다는 것과
실제 inference pipeline이 끝까지 정상적으로 동작한다는 것은
별개의 검증 대상이라는 점을 배웠다.

## 초기 실험에서 놓쳤던 것

초기 코드 중에는
실제 Segment label을 연결하기 전에
임시 random label을 생성해
pipeline을 실행했던 버전도 있었다.

이 상태에서도 모델은 학습되고
metric은 숫자로 출력될 수 있다.

하지만 random label을 사용하는 결과는
실제 분류 문제의 성능을 의미하지 않는다.

이후 실제 Segment를 데이터에서 가져오는 구조로 수정했지만,
이 경험을 통해
"metric이 출력된다"와
"그 metric이 의미 있다"는 전혀 다른 문제라는 것을 알게 되었다.

그래서 현재 포트폴리오에서는
당시의 성능 수치를 결과로 제시하지 않고 있다.

## 프로젝트를 다시 보며 배운 점

이 프로젝트를 진행할 때는
새로운 모델이나 augmentation, ensemble을 추가하면
성능이 좋아질 것이라는 생각을 많이 했다.

하지만 코드를 다시 검토하면서
머신러닝에서는 모델보다 먼저
데이터와 평가 구조를 확인해야 한다는 점을 배웠다.

지금 다시 진행한다면 다음 순서를 우선할 것이다.

1. 문제와 target을 정확히 정의한다.
2. 원본 데이터 단위로 train / validation을 먼저 분리한다.
3. augmentation은 train에만 적용한다.
4. validation set을 독립적으로 유지한다.
5. training과 inference preprocessing을 검증한다.
6. 그 이후에 모델과 학습 방법을 비교한다.

DACON 프로젝트는
높은 성능의 최종 모델을 만들었다는 의미보다,
직접 ML pipeline을 구성하고
그 pipeline을 다시 비판적으로 검토하면서
validation의 중요성을 배운 프로젝트로 남았다.