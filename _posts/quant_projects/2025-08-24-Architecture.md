---
layout: default
title: "백엔드 구성 및 설계"
date: 2025-08-24
categories: [quant]
# 또는 categories: quant
---

![Devops 이미지](/public/images/Architecture.jpg)

이번 시간엔 퀀트의 방법론에 따라,\
백엔드 서버를 어떤 식으로 구성할 지에 대해 기록하겠습니다.

## Tools

1. Fast-api
2. Uvicorn
3. Nginx
4. Docker (+ Docker network)
5. Mysql
6. Redis
7. WireGuard VPN

## Architecture

![Architeture 이미지](/public/images/Architecture_2.png)

매우 소액으로 진행하지만 실제 투자가 진행되므로, 내 증권 API키가 사용된다.\
때문에, Auth가 완성되기 전까지는 VPN 없이 접근 불가능하도록 강력한 보안으로 진행한다.

### GateWay Server

1. 백테스트 결과
2. 백테스트 즉시 실행
3. 전략 서버에 데이터 분석 후, 증권에 매도 매수 진행
  * Redis로 중복 주문 방지
4. 해당 결과 매도, 매수가 진행되었는지 사용자에게 Alarm 전달

### Market Data (비동기)

1. 증권에 데이터 조회 요청 (약 1시간 주기)
2. 데이터를 정리하여 DB에 저장

### BackTest (비동기)

1. Market Data로 분석한 데이터를 호출 (사용자가 호출하지 않는 이상 새벽에만)
2. 해당 데이터로 3일, 7일, 30일, 3개월, 1년 단위로 BackTest 진행
3. 해당 전략의 유효성 분석
4. 해당 결과를 사용자 Alarm으로 전달
5. 해당 결과를 DB에 저장

### Strategy (Quant 핵심)

1. BackTest 결과로 유효율 평가(3일, 7일, 30일만 사용)
2. 해당 결과에 적합한 증권이 있는지 확인 후 매수, 매도

여기서 핵심은 BackTest와 Strategy는 모두 같은 전략을 사용한다는 점이다.\
유지보수 및 사고 방지를 위해 해당 전략을 라이브러리화 하여 사용하는 것이 중요하다.\
모든 에러코드 및 orm 또한 라이브러리화가 필요하다.

다음 시간엔 GateWay Was 개발을 진행하겠습니다.