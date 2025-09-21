---
layout: default
title: "GateWay Was"
date: 2025-08-31
categories: [quant]
---

![Fastapi 이미지](/public/images/Gateway_was.png)

## 왜 Fast-api인가

 내가 기존에 사용했던 Python 프레임워크는 2가지이다.
Django과 Flask인데, 각 프레임워크의 내가 느낀 장단점은 다음과 같다.

* Django
  * 장점
    * 풀스택 프레임워크
    * 오랜 시간 Python 웹 프레임워크의 대표 주자, 풍부한 생태계
    * 빠른 설계 가능. Admin + Auth 기능 기본 제공
  * 단점
    * 작은 서비스에 쓰기에는 불필요한 기능이 많음
    * 내부 규칙이 강해서 구조를 자유롭게 커스터마이즈하기 어려움
    * 비동기 친화적 구조가 아님

  * <span style="color: gray;"><em>대표적인 사용 기업 - Instagram, Pinterest</em></span>

* Flask
  * 장점
    * 마이크로 프레임워크
    * 자유도 높음
  * 단점
    * 모든 기능을 직접 구현
    * 비동기 친화적 구조가 아님

  * <span style="color: gray;"><em>대표적인 사용 기업 - Netflix, Reddit, LinkedIn, Airbnb</em></span>

Fast-api의 장단점은 찾아본 결과 다음과 같다.

* Fast-api
  * 장점
    * 자동 문서화 (flask, django도 라이브러리 결합 시 가능하나 추가적인 품이 들어간다.)
    * 비동기 지원
    * 요즘 대세가 되고 있는 Python 프레임워크 
  * 단점
    * 신생 프레임워크여서 레퍼런스가 적음

  * <span style="color: gray;"><em>대표적인 사용 기업 - Uber, Netflix</em></span>

## MSA 구조에서 Gateway 서버의 역할

 우선, MSA 구조는 서비스의 역할이 명확히 분배되고, 서버의 부하를 줄일 수 있다.
단, Gateway 서비스의 부하는 이전과 동일하다.
Gateway 서비스의 부하가 이전과 동일하다면, MSA로 각 서비스의 부하를 줄인 것도 의미가 없어진다.
어떻게 해결해야 될까?

> 예를 들어, 사용자가 post 생성 요청을 하게 된다면,
> 요청 전달: 사용자 > gateway 서비스 > post 서비스 > 서비스 로직 동작 > gateway 서비스 > 사용자
> 와 같은 전달 순서를 거치게된다. (글 작성이 성공했는지, 실패했는지는 알아야하니까)
> 따라서 Gateway 서비스는 응답이 올 때까지 긴 시간을 대기하게 된다.

gateway에 프로세스를 많이 할당한다 하여도, 감당 불가한 부하가 발생할 것이다.

### 해결 방안

> 오랜 시간이 소요되는 생성, 수정 요청은 어차피 비동기로 구성해야한다. (e.g. 비디오 업로드)
> 따라서 get 요청에 대해서만 해당 로직이 필요하다.

1. 사용자가 조회 요청
2. 해당 요청을 (path, query, body, form)를 md5로 hashing (expired time을 readDB에 저장되는 시간보다 짧게) 
  > 202 accept으로 return하면서 해당 hashing 값을 같이 사용자에게 전달. hashing 값을 같이 서비스 로직에 전달
3. 서비스 로직에서 결과를 readDB(e.g. elastic)에 hashing을 키값으로 저장 (expired time 10초)
4. 클라이언트에서는 해당 hashing 값으로 조회 요청
5. Gateway 서비스에서 접근 가능 여부만 확인 후 readDB에 해당 값이 있으면 전달. 없으면 다시 1~4번 과정


해당 포스트는 다음 글을 참조하였습니다.
https://wikidocs.net/175092 