---
title: "Customer Segment Classification"
description: "A Transformer-based classification experiment integrating multi-source customer data for five-class customer segment prediction."
tech: "Python · PyTorch · KLUE/RoBERTa · Transformers · Machine Learning"
featured: false
---

## Overview

DACON 신용카드 고객 세그먼트 분류 경진대회에서
여러 종류의 고객 데이터를 통합해 A~E의 다섯 개 고객 세그먼트를 예측하는
Transformer 기반 분류 파이프라인을 실험했습니다.

7월부터 12월까지 제공된 고객 데이터를 ID 기준으로 모으고,
회원정보에 포함된 `Segment`를 target label로 사용했습니다.

여러 데이터 소스의 값을 고객별 하나의 입력으로 구성한 뒤,
`klue/roberta-large`를 fine-tuning하여
5-class classification을 수행하는 방식을 사용했습니다.

## Data Processing

학습 데이터는 회원정보, 신용정보, 승인매출정보, 청구정보,
잔액정보, 채널정보, 마케팅정보, 성과정보 등
여러 종류의 Parquet 파일로 구성되어 있었습니다.

각 데이터에서 동일한 고객 ID에 해당하는 정보를 수집하고,
`ID`와 target인 `Segment`를 제외한 feature들을 하나의 고객 입력으로 통합했습니다.

주요 처리 과정은 다음과 같습니다.

- 여러 Parquet 데이터 로딩
- 고객 ID 기준 데이터 통합
- 회원정보의 `Segment`를 classification label로 사용
- 고객별 여러 데이터 소스를 하나의 모델 입력으로 구성
- A~E label을 0~4 class로 변환

## Modeling

분류 모델로 `klue/roberta-large`를 사용했습니다.

고객별 통합 데이터를 text 형태로 변환한 뒤 tokenization하고,
5개의 고객 세그먼트를 예측하도록 sequence classification model을 구성했습니다.

학습 과정에서는 클래스 불균형과 학습 안정성을 다루기 위해
다음 방법들을 실험했습니다.

- Class-weighted loss
- Focal loss
- Text augmentation
- Embedding-space oversampling
- Early stopping
- Gradient accumulation
- Minority-class focused model
- Main model과 minority model의 ensemble

## Evaluation

단순 accuracy뿐 아니라
클래스별 예측 성능을 확인할 수 있도록 평가 구조를 구성했습니다.

사용한 주요 metric은 다음과 같습니다.

- Macro F1
- Class-wise F1
- Accuracy

또한 5-fold cross-validation을 구현해
단일 validation split에만 의존하지 않고
여러 fold에서 모델을 평가하는 방식을 실험했습니다.

## Limitations

프로젝트 이후 코드를 다시 검토하면서
학습 및 추론 pipeline에 몇 가지 개선이 필요한 부분을 확인했습니다.

여러 달과 여러 종류의 고객 데이터를 하나의 긴 text로 통합했지만,
모델 입력 길이를 128 token으로 제한했기 때문에
전체 고객 정보가 충분히 모델에 전달되지 않았을 가능성이 있습니다.

또한 augmentation 및 oversampling이
train/validation split 이전에 적용되는 구조가 포함되어 있어
보다 엄격한 validation을 위해서는
원본 고객 단위로 먼저 데이터를 분리한 뒤
train set에만 augmentation을 적용하도록 수정할 필요가 있습니다.

Test 데이터에는 정답 `Segment`가 포함되지 않기 때문에,
training과 inference 단계의 preprocessing 역시
명확하게 분리하는 구조가 필요합니다.

이러한 이유로 당시의 validation metric이나 competition score를
현재 프로젝트의 성능 성과로 제시하지 않습니다.

## What I Learned

이 프로젝트를 통해 모델 자체뿐 아니라
데이터와 label이 어떤 흐름으로 학습 pipeline에 들어가는지
검증하는 과정이 중요하다는 점을 배웠습니다.

특히 여러 데이터 소스를 하나의 고객 단위 데이터로 구성하고,
class imbalance에 대응하며,
cross-validation과 다양한 학습 방법을 실험하는 경험을 했습니다.

프로젝트 이후 코드를 다시 검토하면서
좋은 metric을 얻는 것보다
데이터 분할, preprocessing, evaluation,
inference 과정이 일관되고 재현 가능하도록 설계하는 것이
머신러닝 실험에서 더 중요하다는 점을 확인했습니다.
