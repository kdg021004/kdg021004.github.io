---
layout: post
title: "긴 문서 QA 시스템에서 Orchestration과 Validation을 설계하며 배운 것"
date: 2026-09-19
categories:
  - AI
  - Development
tags:
  - LLM
  - RAG
  - Orchestration
  - Validation
  - AICOSS
---

AICOSS 프로젝트에서는 LLM이 한 번에 처리하기 어려운 긴 문서를 대상으로
질문의 종류에 따라 서로 다른 retrieval 및 processing path를 사용하는
Question Answering architecture를 개발했다.

내가 주로 담당한 부분은 개별 모델의 성능보다는
여러 component를 실제 실행 흐름으로 연결하는 orchestration과
그 결과를 검증하는 runtime integration이었다.

## 왜 하나의 RAG 경로만으로는 부족했는가

일반적인 RAG는 질문과 관련성이 높은 일부 chunk를 검색하고
그 정보를 LLM에 전달하는 방식으로 동작한다.

이 방식은 특정 사실을 찾는 질문에는 적합하지만,
문서 전체를 대상으로 하는 질문에서는 문제가 생길 수 있다.

예를 들어 다음과 같은 질문은
몇 개의 관련 chunk만 검색해서는 답을 신뢰하기 어렵다.

- 문서 전체에서 특정 항목은 몇 번 등장하는가?
- 여러 section의 내용은 서로 어떤 관계가 있는가?
- 문서 전체에서 가장 큰 값은 무엇인가?
- 특정 내용이 문서 어디에도 존재하지 않는가?

이런 질문은 검색 정확도뿐 아니라
필요한 범위의 문서를 실제로 확인했는지가 중요하다.

## 질문에 따라 실행 경로를 나누기

AICOSS에서는 질문이 요구하는 operation에 따라
실행 범위를 다르게 가져가는 구조를 사용했다.

단순한 local fact는 targeted retrieval을 사용할 수 있지만,
aggregation이나 global synthesis처럼
문서 전체 또는 여러 section을 확인해야 하는 질문은
더 넓은 coverage가 필요하다.

전체적인 실행 흐름은 다음과 같이 생각할 수 있다.

1. 질문을 분석하고 route를 결정한다.
2. route에 필요한 retrieval 또는 scan을 실행한다.
3. specialist가 필요한 경우 해당 처리 경로를 실행한다.
4. 실행 결과와 evidence를 수집한다.
5. identity와 coverage를 검증한다.
6. 검증된 결과만 다음 실행 단계로 전달한다.

내 작업은 이 과정에서 여러 component가
동일한 질문과 execution context를 유지하도록 연결하는 데 집중됐다.

## Question Identity를 유지해야 했던 이유

여러 component를 연결하다 보면
각 결과가 실제로 현재 질문에서 생성된 것인지 확인해야 한다.

단순히 자료형이 맞는다는 이유만으로
결과를 다음 단계에 전달하면,
다른 질문이나 다른 route에서 생성된 결과가
잘못 섞일 가능성이 있다.

그래서 runtime에서 question identity와 route identity를
전달하고 검증하는 구조가 필요했다.

이 경험을 통해 interface contract는
단순히 함수의 input/output type만 의미하는 것이 아니라,
그 데이터가 어떤 실행에서 만들어졌는지까지 포함할 수 있다는 점을 배웠다.

## Evidence와 Coverage 검증

특히 document-wide 질문에서는
답변에 evidence가 존재한다는 사실만으로 충분하지 않았다.

예를 들어 문서 전체의 항목 개수를 계산하는 질문에서
일부 section만 확인하고 결과를 반환한다면
그 결과는 형식적으로는 정상이어도 신뢰할 수 없다.

따라서 다음 두 가지를 구분해야 했다.

- evidence가 존재하는가
- 필요한 문서 범위를 충분히 확인했는가

AICOSS에서는 이러한 coverage 정보를 검증하고,
조건을 만족하지 못한 결과가 그대로 다음 단계로 전달되지 않도록
fail-closed 방식의 처리를 적용했다.

## Fail-closed 방식

LLM 기반 시스템을 만들면서 가장 중요하게 느낀 부분 중 하나는
"결과가 생성되었다"와 "결과를 사용할 수 있다"는
같은 의미가 아니라는 점이었다.

검증에 실패했는데도 fallback 값이나 불완전한 결과를
그대로 downstream으로 전달하면
최종 단계에서는 그 오류를 발견하기 더 어려워진다.

그래서 validation을 통과하지 못한 결과는
정상 결과처럼 처리하지 않는 방향으로 runtime을 구성했다.

## 프로젝트를 통해 배운 점

처음에는 LLM/RAG 시스템에서 가장 중요한 것이
retrieval이나 prompt라고 생각하기 쉬웠다.

하지만 실제로 여러 component를 하나의 시스템으로 연결해보면서
다음 요소들도 동일하게 중요하다는 것을 경험했다.

- component 사이의 contract
- execution identity
- evidence provenance
- document coverage
- validation failure 처리
- deterministic하게 확인할 수 있는 부분과 LLM에 맡길 부분의 분리

결국 AI system을 안정적으로 만들기 위해서는
모델의 출력뿐 아니라
그 출력이 어떤 경로를 통해 만들어졌고
어떤 조건을 만족했는지를 함께 관리해야 했다.

AICOSS 프로젝트는
LLM/RAG 기능 자체를 사용하는 경험에서 한 단계 더 나아가,
여러 AI component를 실제 software runtime 안에서
어떻게 연결하고 검증할 것인지 고민해볼 수 있었던 프로젝트였다.