---
layout: default
title: "아키텍처 다시 생각하기"
date: 2025-12-28
categories: [quant]
---

![Fastapi 이미지](/public/images/Architecture(2))

다시 생각한 아키텍처 그림이다.

## 변경 사유
1. Gateway서버에서 인증/인가 부하를 전부 감당하기 어려울 것 같다.
2. 실제 해당 전략으로 주문을 하게 된다면, 1시간 주기로 증권 시장 데이터를 가져오는 건 너무 느리다. 

추후 코드 작업물과 같이 돌아오겠다.