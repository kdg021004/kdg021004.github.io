---
layout: post
title: "RomanceSlaughter에서 서버 추론과 On-device AI를 연결하며 배운 것"
date: 2026-09-19
categories:
  - AI
  - Development
tags:
  - Android
  - React Native
  - ONNX Runtime
  - On-device AI
  - Mobile AI
---

RomanceSlaughter는 SNS와 메신저 환경에서
로맨스 스캠 및 웹캠 피싱 위험 신호를 탐지하고
사용자에게 경고하는 모바일 애플리케이션으로 개발했다.

2인 팀 프로젝트였고,
내가 특히 많이 다룬 부분은 AI 모델 자체보다
모델의 입력이 되는 정보를 Android 환경에서 수집하고,
분석 결과를 실제 애플리케이션 기능으로 연결하는 과정이었다.

프로젝트를 진행하면서
서버에서 모델을 실행하는 구조와
모바일 기기에서 직접 inference하는 구조를 모두 경험했다.

## 모바일 환경에서는 모델만 있으면 끝나지 않았다

처음에는 AI 모델이 준비되면
앱에서 데이터를 서버에 보내고
결과만 받아오면 된다고 생각하기 쉬웠다.

하지만 실제 모바일 애플리케이션에서는
모델 실행 이전에 해결해야 하는 문제가 많았다.

예를 들어 메신저나 SNS에서 발생하는 정보를 분석하려면
다음과 같은 과정이 필요했다.

1. Android에서 필요한 이벤트를 감지한다.
2. 메시지나 이미지와 같은 입력 데이터를 가져온다.
3. React Native 애플리케이션으로 데이터를 전달한다.
4. 서버 또는 on-device 환경에서 inference를 수행한다.
5. 결과를 다시 애플리케이션 동작과 연결한다.
6. 위험 가능성이 있는 경우 사용자에게 경고한다.

결국 모델은 전체 시스템의 한 부분이었다.

## React Native만으로 해결하기 어려웠던 기능

애플리케이션 UI는 React Native를 기반으로 개발했지만,
메신저 화면의 변화나 백그라운드 동작처럼
Android OS와 가까운 기능은
Java 기반 Native 기능을 함께 사용해야 했다.

특히 다음 기능을 활용했다.

- `AccessibilityService`
- Foreground Service
- Android Native Module
- ContentObserver
- Java 기반 Android API

`AccessibilityService`를 이용하면
화면에서 발생하는 accessibility event를 받을 수 있고,
이를 통해 필요한 텍스트 정보를 감지하는 구조를 만들 수 있었다.

하지만 React Native 코드에서
Android Native API를 직접 사용할 수는 없기 때문에
Native Module을 통해 Java와 JavaScript 사이를 연결해야 했다.

이 과정에서
AI application을 만든다는 것이
단순히 Python 모델 코드를 앱으로 옮기는 작업은 아니라는 점을 체감했다.

## 서버 기반 inference

초기 구조에서는
모바일 애플리케이션에서 필요한 데이터를 수집한 뒤
Flask 서버로 전달하고,
서버에서 AI inference를 수행하는 방식을 사용했다.

이 구조의 장점은 분명했다.

- 모바일 기기의 성능 제약을 덜 받는다.
- 모델을 서버에서 쉽게 변경할 수 있다.
- Python 기반 모델 코드를 그대로 활용하기 쉽다.

반면 모바일 환경에서는 몇 가지 문제가 있었다.

데이터를 서버로 전송해야 하기 때문에
네트워크 연결에 의존하게 되고,
메시지나 이미지처럼 민감할 수 있는 정보를 다룰 경우
개인정보 보호도 함께 고려해야 했다.

또한 사용자가 늘어나면
서버 inference 비용과 운영 문제도 생길 수 있다.

## On-device inference를 고려한 이유

이런 문제를 줄이기 위해
일부 AI 기능을 모바일 기기에서 직접 실행하는 방향을 실험했다.

ONNX Runtime을 이용해
모델을 Android 환경에서 실행하는 방식을 적용했고,
서버 요청 없이 기기 내부에서 inference할 수 있는 구조를 검토했다.

당시 다룬 모델에는
TinyBERT 계열의 텍스트 모델과
CNN / MobileViT 계열의 이미지 모델이 포함되어 있었다.

On-device inference의 장점은 다음과 같았다.

- 네트워크 연결에 대한 의존도를 줄일 수 있다.
- 민감한 데이터를 서버로 보내지 않을 수 있다.
- 반복적인 서버 inference 비용을 줄일 수 있다.
- 앱 내부에서 보다 직접적으로 AI 기능을 실행할 수 있다.

물론 반대로
모바일 기기의 CPU, 메모리, 모델 크기,
inference 속도와 같은 새로운 제약도 생겼다.

그래서 모든 모델을 무조건 on-device로 옮기는 것보다
어떤 기능을 서버에서 처리하고
어떤 기능을 기기 내부에서 처리할지 판단하는 것이 중요했다.

## Android Service와 AI 기능을 연결하기

모델을 실행하는 것만큼 중요한 문제는
"언제 inference를 실행할 것인가"였다.

사용자가 앱 화면을 직접 보고 있지 않을 때도
필요한 이벤트를 감지해야 했기 때문에
Foreground Service를 이용한 백그라운드 동작을 다뤘다.

전체적인 흐름은 다음과 같이 구성했다.

1. Android Service에서 이벤트 감지
2. 필요한 텍스트 또는 이미지 정보 수집
3. Native Module을 통해 React Native와 데이터 전달
4. 서버 또는 ONNX Runtime 기반 inference
5. 결과에 따라 사용자에게 위험 경고

이 구조를 구현하면서
Android lifecycle과 background execution,
React Native와 Native 코드 간 통신을 함께 이해해야 했다.

## 실제 기기에서 테스트하며 알게 된 점

모바일 애플리케이션은
개발 환경에서 코드가 실행되는 것과
실제 스마트폰에서 안정적으로 동작하는 것이 달랐다.

Service의 lifecycle,
권한,
백그라운드 실행,
Accessibility 설정,
Native Module 연결 등은
실제 Android 기기에서 확인해야 하는 부분이 많았다.

그래서 기능을 구현한 뒤
실제 기기를 이용해 반복적으로 테스트하면서
앱과 AI inference 흐름이 함께 동작하는지 확인했다.

## 프로젝트를 통해 배운 점

이 프로젝트에서 가장 크게 배운 것은
AI 모델을 만드는 것과
AI 기능을 제품 안에서 동작하게 만드는 것은
서로 다른 문제라는 점이었다.

모바일 AI application에서는
다음 요소를 함께 고려해야 했다.

- 데이터가 어디에서 들어오는가
- Android에서 데이터를 어떻게 수집하는가
- Native 코드와 application 코드를 어떻게 연결하는가
- inference를 서버에서 할 것인가 기기에서 할 것인가
- 개인정보와 네트워크 의존성은 어떻게 줄일 것인가
- 모델 결과를 실제 사용자 기능으로 어떻게 연결할 것인가

RomanceSlaughter를 진행하면서
AI 모델을 하나의 독립된 프로그램으로 보는 것이 아니라,
실제 software system 안의 component로 바라보는 경험을 할 수 있었다.

이후 다른 AI 프로젝트에서도
모델 성능뿐 아니라
모델이 어떤 환경에서 실행되고
어떤 흐름으로 실제 기능과 연결되는지를
함께 고려하게 된 계기가 되었다.