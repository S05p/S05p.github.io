---
layout: post
title: "무신사 과제 회고"
date: 2026-02-21
categories: [retrospective]
description: "무신사 루키 과제 회고"
---

![무신사 이미지](/public/images/musinsa.png)

## 개요

> 무신사 루키 2차 과제의 회고. 수강신청 시스템을 FastAPI + Redis + MariaDB 기반으로 구현했다.
>
> 핵심 목표는 동시 다중 요청 환경에서의 **정합성 보장**이었다.

---

## 설계 의도

### 아키텍처: 헥사고날

> 테스트 용이성을 위해 헥사고날 아키텍처를 선택했다. 외부 의존성(Redis, DB)을 인터페이스로 추상화하여 테스트 시 SQLite in-memory + FakeRedis로 교체하면 인프라 없이도 전체 비즈니스 로직을 검증할 수 있다.

### 동시성 제어: Redis 이중 분산 락

> 수강신청 시 두 가지 정합성을 동시에 보장해야 했다.
>
> | 락 종류 | 키 | 목적 |
> |---|---|---|
> | Student Lock | `enrollment:student:{id}` | 동일 학생의 동시 요청 직렬화 (학점 합계, 시간표 충돌) |
> | Course Lock | `enrollment:course:{id}` | 동일 강좌의 정원 정합성 보호 |
>
> 락 획득 순서는 항상 **student → course** 로 고정하여 데드락의 조건 중 하나인 순환참조를 방지했다. contextmanager 패턴으로 락 해제를 명시적으로 관리했다.

### FIFO 순서 보장: BLPOP

> 기존 `redis-py`의 `Lock`은 polling 방식 (0.1초 간격 `SET NX` 재시도) 이라 락이 풀리는 순간 경쟁이 발생한다. 도착 순서와 처리 순서가 일치하지 않는다.
>
> BLPOP은 Redis 서버가 대기 클라이언트를 **FIFO 큐로 직접 관리**하므로 순서가 보장된다. polling이 아닌 이벤트 기반이라 CPU 낭비도 없다.

```
[토큰 1개 Redis List]
  → BLPOP (FIFO 대기)
  → 토큰 획득 → owner 키 SET (TTL, 비정상 종료 대비)
  → 비즈니스 로직
  → Lua 스크립트로 owner DEL + 토큰 RPUSH (원자적 해제)
```

### DB 세션 지연 획득 (db_factory 패턴)

> 18,000 RPS 상황에서 락 대기 시간이 최대 30초라면 DB 커넥션을 30초간 점유하는 문제가 생긴다.

```
변경 전: 요청 → DB 획득 → 락 대기(~30초) → 비즈니스 → DB 반환  # DB 점유 ~30초
변경 후: 요청 → 락 대기(~30초) → DB 획득 → 비즈니스(~50ms) → DB 반환  # DB 점유 ~50ms
```

> SessionLocal 팩토리를 의존성으로 주입하고, 락 내부에서만 세션을 생성하도록 변경했다.

### DB 커넥션 풀 수치 근거

> `pool_size=10, max_overflow=20`으로 총 30개 커넥션을 설정했다. db_factory 패턴 덕분에 DB 세션은 락 내부에서만 열리고, 비즈니스 로직 처리 후 즉시 닫힌다. 실측 기준 DB 작업 시간은 약 50ms다.
>
> 워커 4개 기준 초당 ~80건 처리 → 80 × 0.05s = **동시 4개 점유**. 평상시에는 4개면 충분하다. `pool_size=10`은 2.5배 여유, `max_overflow=20`은 순간 부하를 대비한 임시 확장이다.

### 시간표 충돌 감지

> 구간 겹침 알고리즘으로 판별한다.

```python
new.start_time < exist.end_time and new.end_time > exist.start_time
```

> 두 구간이 겹치지 않는 경우는 딱 두 가지다 — 새 강좌가 완전히 앞에 끝나거나(`new.end <= exist.start`), 완전히 뒤에 시작하거나(`new.start >= exist.end`). 이 두 경우의 여집합이 위 조건이다. 요일(`day_of_week`)이 같을 때만 검사한다.

### 수강신청 비즈니스 정책

> 세 가지 정책을 서비스 레이어에서 검증한다.
>
> - **재수강**: 이전 학기 동일 강좌에서 A 이상 받으면 재수강 불가 (`REENROLL_NOT_ALLOWED`)
> - **취소 마감**: 개강일 3일 전까지만 취소 가능 (`CANCELLATION_DEADLINE_PASSED`)
> - **최대 학점**: 학기당 18학점 초과 불가 (`CREDIT_LIMIT_EXCEEDED`)

### 예외 처리 전략

> HTTP 상태코드와 도메인 에러코드를 분리했다. 같은 400이라도 원인이 다르면 클라이언트가 구분할 수 있어야 한다.

```
CustomException (base)
├── BadRequestException (400)  → ENROLLMENT_NOT_STARTED, CREDIT_LIMIT_EXCEEDED, ...
├── ConflictException (409)    → COURSE_FULL, SCHEDULE_CONFLICT, REENROLL_NOT_ALLOWED
└── NotFoundException (404)    → STUDENT_NOT_FOUND, COURSE_NOT_FOUND
```

> 전역 핸들러 하나가 모든 예외를 `{ success: false, error: { code, message } }` 형태로 통일한다. 처리되지 않은 예외는 `INTERNAL_ERROR`로 떨어진다.

### 테스트 전략: 인프라 없이 전체 비즈니스 로직 검증

> 외부 의존성 없이 테스트 가능하도록 두 가지 핵심 교체를 했다.

**SQLite in-memory + StaticPool**

```python
engine = create_engine(
    "sqlite:///:memory:",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,
)
```

> SQLite in-memory는 커넥션마다 별도 DB를 생성한다. `Base.metadata.create_all()`로 테이블을 만든 커넥션과 테스트가 실제로 쓰는 커넥션이 다르면, 테스트 세션에는 테이블이 없다. `StaticPool`은 모든 세션이 커넥션 1개를 공유하도록 강제하여 이 문제를 해결한다.

**FakeRedisService: queue.Queue로 BLPOP 시맨틱 재현**

> 프로덕션의 BLPOP은 Redis 서버가 대기 클라이언트를 FIFO 큐로 관리한다. 이걸 Python으로 재현하면 `queue.Queue(maxsize=1)`이 된다.

```python
@contextmanager
def distributed_lock(self, key, timeout=10, blocking_timeout=30):
    with self._queue_mutex:
        if key not in self._fifo_queues:
            q = queue.Queue(maxsize=1)
            q.put(True)          # 토큰 1개 초기화 (Redis List의 초기 토큰과 동일)
            self._fifo_queues[key] = q
        q = self._fifo_queues[key]

    q.get(timeout=blocking_timeout)  # BLPOP 대기 (thread-safe, FIFO 보장)
    try:
        yield
    finally:
        q.put(True)              # 토큰 반환 (RPUSH와 동일)
```

> `queue.Queue.get()`은 thread-safe + FIFO가 보장된다. 프로덕션 Redis의 BLPOP과 동일한 시맨틱을 외부 의존성 없이 구현한 것이다.

**autouse fixture로 날짜 오버라이드**

```python
@pytest.fixture(autouse=True)
def _override_enrollment_settings():
    today = date.today()
    settings.ENROLLMENT_START_DATE = (today - timedelta(days=7)).isoformat()
    settings.ENROLLMENT_END_DATE   = (today + timedelta(days=7)).isoformat()
    settings.SEMESTER_START_DATE   = (today + timedelta(days=30)).isoformat()
    yield
    # 원복
```

> `autouse=True`로 모든 테스트에 자동 적용된다. 테스트 실행일에 관계없이 수강신청 기간이 항상 유효한 상태로 유지된다. 이슈 1번(날짜 하드코딩 문제)을 겪고 나서 추가한 fixture다.

### Monit 모니터링

> 컨테이너 내부에서 MariaDB, Redis, FastAPI 프로세스를 감시하고 장애 시 자동 재시작한다. 포트는 외부에 노출하지 않고 컨테이너 내부에서만 사용한다.

---

## 이슈

### 1. 수강신청 기간 하드코딩

**상황**

> 동시성 테스트 스크립트를 실행했는데 10명 동시 요청에 1명도 성공하지 못했다.

**원인**

> `config.py`의 `ENROLLMENT_END_DATE`가 `"2025-02-28"`로 하드코딩되어 있었다. 실행일(2026년)을 기준으로 수강신청 기간이 이미 종료된 상태라 모든 요청이 400으로 거부되었다.
>
> 단위 테스트에서는 conftest.py fixture가 날짜를 동적으로 오버라이드해서 이 문제가 잡히지 않았다.

**해결**

> 앱 시작 시점 기준으로 동적 계산하도록 변경했다.
>
> - `ENROLLMENT_START_DATE`: 오늘 - 14일
> - `ENROLLMENT_END_DATE`: 오늘 + 14일
> - `SEMESTER_START_DATE`: 오늘 + 30일

---

### 2. blocking_timeout 부족으로 인한 요청 reject

**상황**

> 테스트 스크립트에서 1, 4, 7번만 성공하고 나머지는 실패했다. Redis lock이 끝날 때까지 대기해야 하는데, 일부 요청이 그냥 reject되는 상황이었다.

**원인**

> `blocking_timeout=3초`로 설정되어 있었다. 요청 1건 처리에 약 0.3초 걸리면, 10번째 요청은 3초 내에 락을 획득하지 못하고 타임아웃으로 실패한다.
>
> `timeout`(락 TTL)과 `blocking_timeout`(락 대기 시간)의 역할을 혼동했다.
>
> | 파라미터 | 역할 |
> |---|---|
> | `timeout` | 프로세스 비정상 종료 시 락 자동 해제 (안전장치) |
> | `blocking_timeout` | 락이 풀릴 때까지 최대 대기 시간 |

**해결**

> `blocking_timeout`을 30초로 증가했다. 100명 동시 요청(×0.3초 = 30초)까지 대응 가능하다.

---

### 3. FIFO Lock 해제의 Race Condition → 500 에러

**상황**

> 테스트 스크립트를 돌리면 간헐적으로 500 에러가 발생했다. 순차성도 여전히 보장되지 않았다.

**원인**

> 락 해제 시 `DELETE`와 `RPUSH`가 비원자적으로 수행되어 토큰이 중복 생성되었다.

```python
# 문제의 코드
self.client.delete(owner_key)     # 1) owner 삭제
# ← 이 사이에 다른 스레드가 init 호출 → 토큰 중복 생성
self.client.rpush(queue_key, "1") # 2) 토큰 반환
```

> 토큰이 2개가 되면 2개 스레드가 동시에 락을 보유하게 되고, DB 동시 쓰기로 IntegrityError → 500이 발생했다.
>
> 또한 매 요청마다 `_ensure_fifo_lock`을 호출하는 구조가 BLPOP → SET owner 사이에 init이 끼어들 race window를 만들었다.

**해결**

> 1. **원자적 해제**: DELETE + RPUSH를 Lua 스크립트로 묶어 원자적으로 수행
> 2. **init 1회화**: 프로세스당 키별로 한 번만 초기화 (로컬 캐시로 중복 방지)
> 3. **토큰 유실 복구**: BLPOP 타임아웃 시 owner TTL 만료 후 토큰 재생성

