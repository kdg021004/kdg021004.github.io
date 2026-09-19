---
title: "RomanceSlaughter"
description: "A mobile AI application designed to detect potential romance-scam and webcam-phishing risks and warn users."
tech: "React Native · Android · Java · ONNX Runtime · Flask · AI"
featured: true
---

## Overview

RomanceSlaughter는 SNS와 모바일 환경에서 발생할 수 있는
로맨스 스캠 및 웹캠 피싱 위험을 탐지하고
사용자에게 경고하기 위해 개발한 모바일 AI 애플리케이션입니다.

2인 팀으로 프로젝트를 진행했으며,
AI 모델을 단순히 학습하는 것보다
실제 Android 애플리케이션 안에서 모델과 탐지 기능을 연결하는 작업에 집중했습니다.

- **Period:** 2024.06 – 2025.03
- **Team:** 2 members

## Problem

로맨스 스캠과 같은 온라인 사기는
메신저 대화, URL, 이미지 등 여러 형태의 정보에서 위험 신호가 나타날 수 있습니다.

따라서 하나의 입력만 분석하는 방식보다
사용자가 실제로 SNS와 메신저를 사용하는 과정에서
필요한 정보를 수집하고 AI 분석 결과를 전달하는 구조가 필요했습니다.

## My Contribution

프로젝트에서 모바일 애플리케이션과 AI 기능을 연결하는 작업을 중심으로 진행했습니다.

주요 작업은 다음과 같습니다.

- React Native 기반 모바일 애플리케이션 개발
- Android Native 기능과 React Native 연결
- AccessibilityService를 이용한 화면 및 메시지 정보 수집
- Foreground Service 기반 백그라운드 모니터링
- 이미지 및 메시지 분석 기능과 AI inference 연결
- Flask backend와 모바일 애플리케이션 통신
- ONNX Runtime을 이용한 모바일 추론 실험
- 실제 Android 기기에서 기능 테스트

## Android Integration

메신저와 SNS 환경에서 필요한 정보를 감지하기 위해
Android의 native 기능을 활용했습니다.

`AccessibilityService`를 이용해
화면 변경 이벤트에서 메시지와 텍스트 정보를 가져오고,
Foreground Service를 사용해 애플리케이션이 백그라운드에서도
필요한 모니터링 작업을 수행할 수 있도록 구성했습니다.

React Native에서 직접 사용하기 어려운 Android 기능은
Native Module을 통해 연결했습니다.

## AI Integration

초기에는 모바일 애플리케이션에서 수집한 데이터를
서버로 전송한 뒤 AI 모델을 실행하는 구조를 사용했습니다.

이후 개인정보 보호와 서버 의존성을 줄이는 방향을 고려하면서
일부 모델을 ONNX 형식으로 변환하고
Android 환경에서 직접 inference하는 방식을 실험했습니다.

이 과정에서 TinyBERT 계열 텍스트 모델과
CNN / MobileViT 계열 이미지 모델을
모바일 환경에서 활용하는 방법을 검토했습니다.

## System Flow

애플리케이션은 대략 다음과 같은 흐름으로 동작하도록 구성했습니다.

1. SNS 또는 메신저에서 텍스트 및 관련 이벤트 감지
2. 필요한 데이터를 애플리케이션으로 전달
3. 서버 또는 on-device AI inference 수행
4. 위험 여부 판단
5. 위험 가능성이 있는 경우 사용자에게 경고

이 과정에서 AI 모델의 출력 자체뿐 아니라,
모델을 언제 실행하고 결과를 애플리케이션 기능으로 어떻게 연결할지를 함께 다뤘습니다.

## What I Learned

이 프로젝트를 통해 AI 모델을 만드는 것과
AI 기능을 실제 애플리케이션에서 동작하게 만드는 것은
서로 다른 문제라는 점을 배웠습니다.

특히 Android lifecycle, background service,
native API와 React Native 간 통신,
서버와 모바일 환경의 차이 등을 다루면서
AI 기능을 실제 소프트웨어 시스템에 통합하는 경험을 할 수 있었습니다.

또한 서버 기반 inference와 on-device inference의 차이를 비교하면서
개인정보 보호, 네트워크 의존성, 처리 비용과 같은
애플리케이션 관점의 요구사항도 함께 고려하게 되었습니다.