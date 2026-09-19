---
title: "AICOSS — Long-Document Question Answering System"
description: "A long-document LLM question answering system integrating retrieval, specialist execution, orchestration, and evidence validation."
tech: "Python · LLM · RAG · Orchestration · Runtime Integration · Validation"
featured: true
github: "https://github.com/dodams258/aicoss"
---

## Overview

AICOSS는 긴 문서를 대상으로 질문에 필요한 정보를 찾고,
여러 retrieval 및 specialist 경로를 통해 답변을 생성하는
LLM 기반 Question Answering 시스템입니다.

독일 THU(Technische Hochschule Ulm)에서 진행한
4인 팀 Capstone Project로,
긴 문서를 한 번에 처리하기 어려운 LLM의 한계를 보완하기 위해
질문 특성에 따라 여러 처리 경로를 선택하고
결과를 통합하는 구조를 구현했습니다.

* **Period:** 2026.07 – 2026.08
* **Team:** 4 members
* **Program:** AICOSS – THU Capstone Project

## Problem

긴 문서를 LLM에 그대로 전달하는 방식은
context length와 처리 비용의 제약을 받으며,
문서 전체에서 필요한 근거를 안정적으로 수집하기 어렵습니다.

또한 질문의 종류에 따라
단순 retrieval만으로 충분한 경우와,
여러 section의 관계를 비교하거나
문서 전체 내용을 종합해야 하는 경우가 달랐습니다.

따라서 하나의 고정된 처리 방식보다
질문에 맞는 specialist와 retrieval 경로를 선택하고,
각 경로의 결과가 실제 질문과 문서 근거에 맞는지 검증하는
runtime 구조가 필요했습니다.

## System Architecture

시스템은 질문의 answer operation을 기준으로
적절한 retrieval 또는 specialist 경로를 선택하고,
각 경로의 결과를 검증 가능한 공통 실행 흐름으로 연결하는 구조로 설계했습니다.

주요 흐름은 다음과 같습니다.

1. 사용자 질문 입력
2. Router를 통한 처리 경로 결정
3. Structured Scan 또는 RAG 기반 정보 검색
4. 필요한 specialist 실행
5. 실행 결과 및 evidence 수집
6. identity, evidence, coverage 검증
7. 검증을 통과한 결과만 downstream execution path로 전달

단순히 각 모듈을 호출하는 것뿐 아니라,
각 단계가 동일한 질문과 실행 context를 유지하는지 확인하는 것이 중요했습니다.

## My Contribution

프로젝트에서 orchestration과 runtime integration을 중심으로 작업했습니다.

### Orchestration

Router가 선택한 경로에 따라
Structured Scan, RAG 및 specialist를 실행하고
그 결과를 하나의 runtime 흐름으로 연결했습니다.

주요 작업은 다음과 같습니다.

* Router 기반 specialist / retrieval route 실행
* 실행 context 및 question identity 전달
* 여러 specialist 결과 aggregation
* route dependency 연결 및 검증
* invalid result가 final answer까지 전달되지 않도록 fail-closed 처리

### Runtime Integration

개별적으로 구현된 specialist가
실제 Question Answering runtime에서 동작할 수 있도록
공통 실행 구조에 통합했습니다.

각 component 사이에서
입력과 출력의 contract가 유지되는지 확인하고,
잘못된 route나 서로 다른 질문의 결과가 섞이지 않도록
runtime validation을 추가했습니다.

### Cross-section Specialist

문서의 서로 다른 section 사이의 관계를 분석해야 하는 질문을 처리하기 위해
Cross-section Specialist의 runtime integration을 담당했습니다.

한 section의 정보만 사용하는 대신
여러 section에서 수집한 evidence를 연결해
질문에 필요한 관계를 분석할 수 있도록 구성했습니다.

### Global Synthesis Specialist

문서 전체 내용을 종합해야 하는 질문을 위해
Global Synthesis Specialist를 runtime에 통합했습니다.

일부 section의 정보만으로
문서 전체를 대표하는 답변이 생성되지 않도록
여러 section에서 충분한 evidence가 수집되었는지 확인하는
coverage validation을 강화했습니다.

## Validation

이 프로젝트에서는 LLM이 답변을 생성했다는 사실만으로
결과를 유효하다고 판단하지 않도록 하는 데 집중했습니다.

주요 검증 항목은 다음과 같습니다.

* Question identity validation
* Route identity validation
* Evidence validation
* Document coverage validation
* Runtime contract validation
* Deterministic execution checks
* Fail-closed handling

검증 조건을 통과하지 못한 실행 결과는
최종 synthesis 단계로 전달하지 않도록 구성했습니다.

통합 이후에는 자동화된 테스트를 통해
기존 기능과 새 runtime integration이 함께 동작하는지 확인했습니다.

## Result

여러 retrieval 및 specialist component를 연결하기 위한
orchestration contract와 runtime integration 경로를 구현하고,
실행 결과의 identity, evidence, coverage를 검증하는 구조를 강화했습니다.

이를 통해 단순히 RAG를 호출하는 데서 끝나는 것이 아니라,
여러 AI component가 함께 동작할 때
어떤 결과를 신뢰하고 다음 단계로 전달할 것인지까지
시스템 수준에서 다루는 경험을 할 수 있었습니다.

## What I Learned

LLM 기반 시스템에서는
모델이나 retrieval 성능뿐 아니라
여러 component 사이의 contract와 execution flow를
명확하게 관리하는 것이 중요하다는 점을 배웠습니다.

특히 잘못된 중간 결과가 이후 단계로 전달되면
최종 답변 전체의 신뢰성이 떨어질 수 있기 때문에,
evidence와 coverage를 검증하고
문제가 있는 결과를 차단하는 구조가 필요했습니다.

이 프로젝트를 통해
LLM/RAG 기능을 개별적으로 구현하는 것뿐 아니라
여러 AI component를 실제 software runtime에 통합하고
검증하는 경험을 할 수 있었습니다.
