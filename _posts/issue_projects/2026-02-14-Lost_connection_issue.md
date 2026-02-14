---
layout: post
title: "Lost Connection Issue"
date: 2026-02-14
categories: [issue]
description: "Lost connection issue 분석"
---

![트러블 슈팅 이미지](/public/images/Trouble_shooting.jpg)

## 이슈

0. 특정 서버가 정상적으로 기동 중임을 확인.
1. 특정 서버의 보안 이슈로 SSL 인증서 관련 프로그램을 업데이트 이후 재부팅을 진행.
2. 재부팅 이후, monit으로 기존 docker 컨테이너들이 정상적으로 실행됨을 확인.
3. 단 WAS 쪽에서 DB가 연결되지 않는 문제를 발견 (`Lost connection ...`)

## 원인

1. DBSafer에 WAS의 IP(Docker network IP)가 화이트리스트에 없음.
2. 서버를 재기동하는 과정에서 먼저 실행되는 컨테이너가 IP를 할당받음.
3. 이로 인해, 서버 재부팅 이전에 지정한 화이트리스트의 IP를 타 컨테이너가 점유하게 됨.

## 개선점

1. 각 컨테이너 별 Docker Network IP를 지정하는 게 좋아보임.
2. DBSafer에 화이트리스트로 도커 네트워크 대역대를 추가.

## 배운점 (디버깅)

DBSafer가 강제로 연결을 종료한다는건 생각치 못했다.\
요점은 DBSafer가 있을수도 있어요. 를 디버깅 시 고려하는게 아니라 로그를 보는 방식이다.

1. WAS로그를 확인 시 DB의 연결이 `Lost connection`이라고 나왔다.
2. DB로그를 확인 시 WAS의 연결이 `Handshack failed .. `라고 나왔다.

이 로그로 알아낼 수 있는 정보는 다음과 같다.
1. 일단 WAS, DB 연결은 됐다. > 소켓통신까지는 성공했다. - 연결이 되지 않았다면 다른 종류의 로그가 나와야한다. 

위 정보를 확인하고 1. 금일 업데이트한 SSL 2. Docker network를 의심했다.
1. WAS 컨테이너에서 mysql client로 DB 컨테이너 요청 (ssl 미사용 옵션) > `Handshack failed...`\
SSL 미사용 옵션을 적용했는데도 동일한 로그로 실패 > 그렇다면 DB의 SSL 관련된 문제는 아니다.
2. Docker network를 통해 WAS에서 요청 가능한 타 서비스로 요청 (e.g. WAS > WAS) > 정상적으로 요청 성공.\
Docker network를 통해 타 컨테이너로 ping 요청도 정상적으로 간다. > 그렇다면 서버 호스트의 SSL 관련된 문제는 아니다.

여기까지 오니 정신이 매우 혼미해졌다.\
AI에게 아무리 도움을 받아도 추정되는 내용이 다 max_connection, thread 폭주 같은 내용들..\
약 2시간 동안 끙끙거린 후, 이건 외부 프로세스가 연결을 끊는 것이다. 라는 결론에 도달했다. (거의 논리비약)

이후로 혹시나 싶어 DBSaferr같은게 있나 싶어서 확인해보니, 있었다. (!!!!)\
타 팀에서 해당 프로세스를 설치하였다고 했다.\
일단 화이트리스트에 WAS IP를 추가해달라하였고,\
추후 된다면, 도커 네트워크 대역대 추가를 요청하였다.

배운 점은,\
외운대로 디버깅 하지말자.\
막히면 막힐수록, 로그 내용에 집중하자 이다.