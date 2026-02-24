---
layout: post
title: "Redis 분산락 최적화 — 블로킹 2회에서 1회로"
date: 2026-02-24
categories: [retrospective]
description: "Lua 스크립트 기반 DECR로 course lock을 제거하고 학생 lock만 남겨 블로킹을 1회로 줄인 설계"
---

![티켓팅 이미지](/public/images/ticketing.jpg)

## 개요

> 수강신청 시스템의 분산락 구조를 다시 들여다보다가, 블로킹이 2회 발생하는 구조이다.
>
> 티켓팅 시스템에서 사용하는 Redis decrease를 사용하면 블로킹을 1회 줄일 수 있어서 해당 방법을 소개하고자 한다.
>
> 학생 단위 락과 강좌 단위 락을 각각 BLPOP으로 획득하는데, 강좌 정원 차감은 BLPOP 없이 Lua 스크립트로 원자적으로 처리할 수 있다. 이 아이디어를 정리해봤다.

---

## AS-IS: 블로킹 2회 구조

**흐름**

```
사용자 요청
→ [Pre-check]      스케줄 + 학점 체크 (non-blocking)
→ [1차 블로킹]     BLPOP student_lock → re-query 검증 → 락 해제
→ [2차 블로킹]     BLPOP course_lock  → 정원 확인 → 차감 → 락 해제
→ DB INSERT
```

**문제점**

> 학생 락과 강좌 락을 각각 BLPOP으로 획득한다. 두 구간이 순차적으로 블로킹되므로 단일 요청의 대기 시간이 길어지고, 동시 요청이 폭주할수록 타임아웃 실패율이 급등한다.
>
> 강좌 락은 사실 정원 카운트 조회 + 차감이라는 단순 연산이다. 이 연산은 별도의 락 없이 Redis의 원자성만으로 안전하게 처리할 수 있다.

---

## TO-BE: 블로킹 1회 구조

**핵심 아이디어**

> 강좌 잔여 정원을 `course:{course_id}:cnt` 키로 Redis에 관리하고, Lua 스크립트로 원자적 DECR을 수행한다.
>
> Lua 스크립트는 Redis 내부에서 단일 명령처럼 실행되므로 별도의 강좌 락이 필요 없다. 학생 락(BLPOP) 하나만 남기고 강좌 락을 제거할 수 있다.

![sync 시퀸스 다이어그램](/public/images/lock_with_decr.png)

---

## 상세 흐름

### 1. Pre-check

> 락 획득 전 빠른 조기 탈출을 위한 사전 검증이다. 시간표 충돌과 학점 초과 여부를 DB에서 조회해 하나라도 조건에 맞지 않으면 reject한다.
>
> 이 시점의 검증 결과는 최신 상태를 보장하지 않는다. 정합성은 이후 락 획득 후 re-query에서 재확인한다.

---

### 2. 학생 락 획득 + re-query (블로킹 1회)

> `BLPOP student:{student_id}:lock`으로 학생 단위 락을 획득한다. 락 획득 이후 트랜잭션 내에서 학점과 시간표를 다시 조회한다. Pre-check 이후 상태가 바뀌었을 수 있으므로 re-query가 필수다.
>
> 하나라도 조건이 맞지 않으면 즉시 reject하고 락을 해제한다.

---

### 3. Lua 스크립트로 강좌 정원 차감

> 학생 락 내에서 Lua 스크립트로 `course:{course_id}:cnt`(잔여 정원)를 원자적으로 처리한다.

```lua
local key      = KEYS[1]  -- course:{course_id}:cnt
local init_key = KEYS[2]  -- course:{course_id}:init

local val = redis.call('GET', key)

if val == false then
    -- 키 없음 → init 로직 필요
    local acquired = redis.call('SET', init_key, '1', 'NX', 'EX', 30)
    if acquired then
        return -1  -- init key 획득 성공
    else
        return -2  -- init key 획득 실패 (다른 스레드가 init 진행 중)
    end
elseif tonumber(val) <= 0 then
    return 0       -- 정원 초과
else
    return redis.call('DECR', key)  -- 잔여 정원 차감, 남은 값 반환
end
```

**반환값 해석**

| 반환값 | 의미 | 처리 |
|:---:|:---:|:---:|
| `-1` | init key 획득 성공 | init 로직 수행 후 DECR 재시도 |
| `-2` | init key 획득 실패 | 짧은 sleep 후 DECR 재시도 |
| `0` | 정원 초과 | reject |
| `> 0` | 차감 성공 | 남은 잔여 정원, 진행 |

---

### 4. Init 로직 — 키가 없는 경우

> `course:{course_id}:cnt` 키가 없는 상황은 두 가지다.
>
> 1. 수강신청 시작 시점 (최초 접근)
> 2. Redis 재시작 등으로 키가 증발한 경우
>
> 여러 스레드가 동시에 "키 없음"을 감지할 수 있으므로, `course:{course_id}:init`을 추가 mutex로 사용해 단 하나의 스레드만 초기화를 수행하도록 한다.

**init key 획득 성공 (반환값 `-1`)**

```
1. GET course:{course_id}:cnt
   → 이미 존재하면 탈출 (다른 스레드가 먼저 초기화 완료)

2. DB 조회 → 현재 수강인원(count) 확인

3. SETNX course:{course_id}:cnt {max_capacity - count}
   → 잔여 정원 세팅

4. step 3 (Lua DECR) 재시도
```

> 1번 double-check는 init key 획득과 DB 조회 사이의 짧은 gap에서 다른 스레드가 먼저 초기화를 완료했을 경우를 방어한다. 이미 키가 생겼다면 내가 DB를 다시 볼 필요 없이 Lua DECR만 재시도하면 된다.

**init key 획득 실패 (반환값 `-2`)**

```
다른 스레드가 init 진행 중 → 짧은 sleep 후 step 3 재시도
```

> init이 완료되면 `cnt` 키가 생성되고, 다음 Lua 실행에서 정상적으로 DECR 경로를 탄다.

---

### 5. DB INSERT + 실패 시 복구

> Lua DECR 성공 이후 DB에 수강신청 레코드를 INSERT한다. DB 쓰기가 실패하면 Redis에서 차감한 잔여 정원을 반드시 복구해야 한다.

```
DB INSERT 성공 → unlock → 완료
DB INSERT 실패 → INCR course:{course_id}:cnt (잔여 정원 복구) → unlock → reject
```

> Redis 차감이 DB 반영보다 먼저 이루어지므로, DB 실패 시 보상 개념으로 INCR을 반드시 수행해야 한다. 이 부분이 누락되면 실제 수강인원보다 잔여 정원이 적게 집계돼 정원 카운트 불일치가 발생한다.

---

## 전체 시퀀스

```
[Pre-check]
  스케줄 + 학점 DB 조회
  → 실패: reject

[Student Lock — BLPOP]   ← 블로킹 1회
  re-query (학점, 시간표)
  → 실패: reject + unlock

[Lua DECR]               ← atomic, non-blocking
  course:{course_id}:cnt 처리
  ├─ not exist + init 획득(-1)
  │    double-check → DB 조회 → SETNX cnt → Lua DECR 재시도
  ├─ not exist + init 실패(-2)
  │    sleep → Lua DECR 재시도
  ├─ exist & 0
  │    정원 초과 → reject + unlock
  └─ exist & > 0
       DECR 성공 → 진행

[DB INSERT]
  성공 → unlock → 완료
  실패 → INCR 복구 → unlock → reject
```

---

## 트레이드오프

| 관점 | AS-IS (블로킹 2회) | TO-BE (블로킹 1회) |
|---|---|---|
| 블로킹 횟수 | 2회 (student + course BLPOP) | 1회 (student BLPOP) |
| 강좌 정원 처리 | course lock 내 DB 조회 + 차감 | Lua 원자적 DECR |
| 요청 순서 보장 | course BLPOP 큐 순서에 준함 | **보장 안 됨** |
| Init 복잡도 | 단순 | init key + double-check 필요 |
| Redis 키 증발 | 락 기반이라 영향 없음 | cnt 키 소실 시 init 재진입 (자동 복구) |
| DB 실패 보상 | 락만 해제 | INCR 복구 필수 |
| 타임아웃 실패 위험 | 2구간 누적 | 1구간으로 감소 |

> 가장 큰 트레이드오프는 **요청 순서 보장 포기**다.
>
> 학생 락은 `student:{student_id}:lock`으로 학생별로 분리되어 있다. 즉 여러 학생의 요청이 사실상 동시에 Lua DECR 단계까지 도달하고, 남은 정원 1석에 여러 명이 동시에 경쟁한다. Redis가 먼저 처리한 요청이 정원을 가져가는 구조이므로, "먼저 요청한 학생이 반드시 우선"은 보장되지 않는다.
>
> 수강신청 맥락에서는 정원 초과 방지(정합성)가 절대 요건이고, 동일 밀리초 내 동시 요청의 순서는 어차피 네트워크 지터 수준이라 허용 범위로 볼 수 있다. 반면 "줄 선 순서대로"가 사용자와의 명시적 약속인 콘서트 티켓팅 같은 시스템이라면 이 구조는 적합하지 않다.
>
> init 로직의 복잡도 증가와 DB INSERT 실패 시 INCR 보상도 빠뜨리면 안 된다. Redis 키 증발 상황에서 DB를 통해 자동 복구되는 점은 오히려 장점이다.

---

## 결론

> 강좌 lock을 Lua 스크립트 기반 DECR로 대체하면 BLPOP 블로킹을 2회에서 1회로 줄일 수 있다.
>
> 동시 요청이 폭주하는 수강신청 특성상 블로킹 구간 단축은 워커 처리량 개선과 타임아웃 실패율 감소로 이어진다.
>
> 구현 시 init key double-check와 DB 실패 시 INCR 보상은 반드시 챙겨야 한다. 어느 한 곳이라도 빠지면 정원 초과 또는 잔여 정원 불일치가 발생한다.
