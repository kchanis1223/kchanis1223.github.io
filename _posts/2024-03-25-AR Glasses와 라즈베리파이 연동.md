---
title: AR Glasses와 Rasberry Pie 연동
date: 2024-07-01 4:41:34 +0800
categories: [Project, 게임엔진/Unity]
tags: [writing]
description: 스마트 헬멧 제작을 위해 AR Glasses와 Rasberry Pi 연동하기
render_with_liquid: false
---

# 목표
스마트 헬멧의 기능을 위해, 라즈베리파이에 부착된 센서&장치들과 AR Glasses의 연동.

## AR Glasses

- 사용 장비 : Nreal Light
- 개발 환경 : Unity3D 22.03.16

먼저 Nreal Light 제품을 사용하므로, 해당 기기를 목적으로 만들어진 NRSDK를 이용하면 쉽게 작업할 수 있다.
<br>
NRSDK 링크 : `https://xreal.gitbook.io/nrsdk/nrsdk-fundamentals/quickstart-for-android`
<br>
<br>

Documents를 참고하여 진행하면 Unity에 설치하고 빌드하는 것은 어렵지 않다. (Unity 버전과 NRSDK 버전, target Android만 잘 확인하자)
<br>
<br>
NRSDK에 함께 제공되는 HelloMR 씬을 빌드한 후, 스마트폰에 Nreal Light를 연결하고 실행하여 정상 작동 하는지 테스트 해볼 수 있다.
<br>
<br>
 

 ## Raspberry Pi

 - 사용 장비 : 라즈베리파이 제로 WH, 2W
 - OS : Raspberry Pi OS 64/32bit 

 라즈베리파이에 Raspberry pi OS를 올려 python과 C를 사용하여 센서를 제어하고 통신을 위한 코드를 구성하였다. 통신을 위해 MQTT 프로토콜을 사용하였는데, MQTT는 IOT에 많이 사용되는 프로토콜이다.
<br>

 MQTT 설명 링크 : `https://aws.amazon.com/ko/what-is/mqtt/`

<br>
<br>
라즈베리 파이에서는 MQTT를 위해 mosquitto를 설치하여 broker로 사용하였고, 센서↔AR 글래스 간 pub/sub을 구성하였다. 같은 네트워크 하에 존재할 때, 연결된 기기들에 대해 메시지를 보낼 수 있도록 하였으며, 하나의 Topic만을 만들어, 자원 사용을 최소화 했다.

## 통신

- 