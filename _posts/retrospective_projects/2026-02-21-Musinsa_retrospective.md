---
layout: post
title: "무신사 과제 회고"
date: 2026-02-21
categories: [retrospective]
description: "무신사 루키 과제 회고"
---

![무신사 이미지](/public/images/musinsa.png)

## 개요

무신사 루키 2차 과제의 회고. 수강신청 시스템을 FastAPI + Redis + MariaDB 기반으로 구현했다.\
핵심 목표는 동시 다중 요청 환경에서의 **정합성 보장**이었다.

---

## 설계 의도

### 아키텍처: 헥사고날

테스트 용이성을 위해 헥사고날 아키텍처를 선택했다. 외부 의존성(Redis, DB)을 인터페이스로 추상화하여 테스트 시 SQLite in-memory + FakeRedis로 교체하면 인프라 없이도 전체 비즈니스 로직을 검증할 수 있다.

### 동시성 제어: Redis 이중 분산 락

수강신청 시 두 가지 정합성을 동시에 보장해야 했다.

| 락 종류 | 키 | 목적 |
|---|---|---|
| Student Lock | `enrollment:student:{id}` | 동일 학생의 동시 요청 직렬화 (학점 합계, 시간표 충돌) |
| Course Lock | `enrollment:course:{id}` | 동일 강좌의 정원 정합성 보호 |

락 획득 순서는 항상 **student → course** 로 고정하여 데드락의 조건 중 하나인 순환참조를 방지했다. contextmanager 패턴으로 락 해제를 명시적으로 관리했다.

### FIFO 순서 보장: BLPOP

기존 `redis-py`의 `Lock`은 polling 방식 (0.1초 간격 `SET NX` 재시도) 이라 락이 풀리는 순간 경쟁이 발생한다. 도착 순서와 처리 순서가 일치하지 않는다.

BLPOP은 Redis 서버가 대기 클라이언트를 **FIFO 큐로 직접 관리**하므로 순서가 보장된다. polling이 아닌 이벤트 기반이라 CPU 낭비도 없다.

```
[토큰 1개 Redis List]
  → BLPOP (FIFO 대기)
  → 토큰 획득 → owner 키 SET (TTL, 비정상 종료 대비)
  → 비즈니스 로직
  → Lua 스크립트로 owner DEL + 토큰 RPUSH (원자적 해제)
```

### DB 세션 지연 획득 (db_factory 패턴)

18,000 RPS 상황에서 락 대기 시간이 최대 30초라면 DB 커넥션을 30초간 점유하는 문제가 생긴다.

```
변경 전: 요청 → DB 획득 → 락 대기(~30초) → 비즈니스 → DB 반환  # DB 점유 ~30초
변경 후: 요청 → 락 대기(~30초) → DB 획득 → 비즈니스(~50ms) → DB 반환  # DB 점유 ~50ms
```

SessionLocal 팩토리를 의존성으로 주입하고, 락 내부에서만 세션을 생성하도록 변경했다.

### 예외 처리 전략

HTTP 상태코드와 도메인 에러코드를 분리했다. 같은 400이라도 원인이 다르면 클라이언트가 구분할 수 있어야 한다.

```
CustomException (base)
├── BadRequestException (400)  → ENROLLMENT_NOT_STARTED, CREDIT_LIMIT_EXCEEDED, ...
├── ConflictException (409)    → COURSE_FULL, SCHEDULE_CONFLICT, REENROLL_NOT_ALLOWED
└── NotFoundException (404)    → STUDENT_NOT_FOUND, COURSE_NOT_FOUND
```

전역 핸들러 하나가 모든 예외를 `{ success: false, error: { code, message } }` 형태로 통일한다. 처리되지 않은 예외는 `INTERNAL_ERROR`로 떨어진다.

### 테스트 전략: 인프라 없이 전체 비즈니스 로직 검증

외부 의존성 없이 테스트 가능하도록 두 가지 핵심 교체를 했다.

**SQLite in-memory + StaticPool**

```python
engine = create_engine(
    "sqlite:///:memory:",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,
)
```

SQLite in-memory는 커넥션마다 별도 DB를 생성한다. `Base.metadata.create_all()`로 테이블을 만든 커넥션과 테스트가 실제로 쓰는 커넥션이 다르면, \
테스트 세션에는 테이블이 없다. `StaticPool`은 모든 세션이 커넥션 1개를 공유하도록 강제하여 이 문제를 해결한다.

**FakeRedisService: queue.Queue로 BLPOP 시맨틱 재현**

프로덕션의 BLPOP은 Redis 서버가 대기 클라이언트를 FIFO 큐로 관리한다. 이걸 Python으로 재현하면 `queue.Queue(maxsize=1)`이 된다.

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

`queue.Queue.get()`은 Python 표준 라이브러리로 thread-safe + FIFO가 보장된다. 프로덕션 Redis의 BLPOP과 동일한 시맨틱을 외부 의존성 없이 구현한 것이다.

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

`autouse=True`로 모든 테스트에 자동 적용된다. 테스트 실행일에 관계없이 수강신청 기간이 항상 유효한 상태로 유지된다. 이슈 1번(날짜 하드코딩 문제)을 겪고 나서 추가한 fixture다.

### Monit 모니터링

컨테이너 내부에서 MariaDB, Redis, FastAPI 프로세스를 감시하고 장애 시 자동 재시작한다. 포트는 외부에 노출하지 않고 컨테이너 내부에서만 사용한다.

---

## 이슈

### 1. 수강신청 기간 하드코딩

**상황**

동시성 테스트 스크립트를 실행했는데 10명 동시 요청에 1명도 성공하지 못했다.

**원인**

`config.py`의 `ENROLLMENT_END_DATE`가 `"2025-02-28"`로 하드코딩되어 있었다. 실행일(2026년)을 기준으로 수강신청 기간이 이미 종료된 상태라 모든 요청이 400으로 거부되었다.

단위 테스트에서는 conftest.py fixture가 날짜를 동적으로 오버라이드해서 이 문제가 잡히지 않았다.

**해결**

앱 시작 시점 기준으로 동적 계산하도록 변경했다.
- `ENROLLMENT_START_DATE`: 오늘 - 14일
- `ENROLLMENT_END_DATE`: 오늘 + 14일
- `SEMESTER_START_DATE`: 오늘 + 30일

---

### 2. blocking_timeout 부족으로 인한 요청 reject

**상황**

테스트 스크립트에서 1, 4, 7번만 성공하고 나머지는 실패했다. Redis lock이 끝날 때까지 대기해야 하는데, 일부 요청이 그냥 reject되는 상황이었다.

**원인**

`blocking_timeout=3초`로 설정되어 있었다. 요청 1건 처리에 약 0.3초 걸리면, 10번째 요청은 3초 내에 락을 획득하지 못하고 타임아웃으로 실패한다.

`timeout`(락 TTL)과 `blocking_timeout`(락 대기 시간)의 역할을 혼동했다.

| 파라미터 | 역할 |
|---|---|
| `timeout` | 프로세스 비정상 종료 시 락 자동 해제 (안전장치) |
| `blocking_timeout` | 락이 풀릴 때까지 최대 대기 시간 |

**해결**

`blocking_timeout`을 30초로 증가했다. 100명 동시 요청(×0.3초 = 30초)까지 대응 가능하다.

---

### 3. FIFO Lock 해제의 Race Condition → 500 에러

**상황**

테스트 스크립트를 돌리면 간헐적으로 500 에러가 발생했다. 순차성도 여전히 보장되지 않았다.

**원인**

락 해제 시 `DELETE`와 `RPUSH`가 비원자적으로 수행되어 토큰이 중복 생성되었다.

```python
# 문제의 코드
self.client.delete(owner_key)     # 1) owner 삭제
# ← 이 사이에 다른 스레드가 init 호출 → 토큰 중복 생성
self.client.rpush(queue_key, "1") # 2) 토큰 반환
```

토큰이 2개가 되면 2개 스레드가 동시에 락을 보유하게 되고, DB 동시 쓰기로 IntegrityError → 500이 발생했다.

또한 매 요청마다 `_ensure_fifo_lock`을 호출하는 구조가 BLPOP → SET owner 사이에 init이 끼어들 race window를 만들었다.

**해결**

1. **원자적 해제**: DELETE + RPUSH를 Lua 스크립트로 묶어 원자적으로 수행
2. **init 1회화**: 프로세스당 키별로 한 번만 초기화 (로컬 캐시로 중복 방지)
3. **토큰 유실 복구**: BLPOP 타임아웃 시 Blocking 대기시간 만료 후 토큰 재생성

---

## 대기열 시스템 도입 점검 — AS-IS / TO-BE

과제가 종료되고 나서, 병목지점을 검토해보았다.\
사용자 -\> nginx -\> WAS -\> redis | DB 흐름에서 보통 병목이 발생하는 지점은 2가지이다.
1. `nginx -> WAS`
2. `WAS -> redis | DB`

1번 지점에서 병목이 발생하는 이유는 worker가 바쁘기 때문에 발생한다.\
1개의 worker가 최대 몇 가지의 요청을 처리할 수 있을까? 하고 AS-IS 상태에서 요청을 날린 결과,\
1개의 worker가 초당 약 280건의 요청을 처리하고 있었다 (wrk 4t/100c/30s 실측). 이 때 redis의 연결을 async로 비동기 redis를 사용한다면 더 많은 양을 처리할 수 있겠는데? 라는 생각이 들었고,\
이를 검토하다 보니 동기/비동기 Redis와 BLPOP의 조합에서 각각 다른 구조적 한계가 드러났다.

> 여기서 말하는 BLPOP은 **락 메커니즘**으로 사용되는 BLPOP이다 (아래 AS-IS 흐름의 그것). 이후 등장하는 대기열 시스템의 BLPOP(큐 소비자 패턴)과는 다른 맥락이다.

**동기 Redis + BLPOP**

워커 N개 = 최대 커넥션 N개. BLPOP이 스레드를 블로킹하므로 커넥션이 터질 구조 자체가 없다.\
단, BLPOP 대기 중에는 스레드 전체가 멈추므로 FastAPI의 async 장점을 완전히 포기하게 된다. 사실상 동기 서버로 동작한다.

**비동기 Redis + BLPOP**

코루틴은 수천 개 동시 실행이 가능하다. 각 코루틴이 BLPOP 대기 중 커넥션을 점유하므로, 18,000개 요청이 동시에 대기에 들어가면 18,000개의 커넥션이 필요해진다.\
커넥션 풀 기본 크기(10~50)를 즉시 고갈시키고, 풀 크기를 늘리면 이번엔 Redis 서버의 `maxclients`(기본 10,000, 실무 상한 ~20,000)에 걸린다.\
FastAPI의 비동기 처리량을 살리려다가 오히려 Redis 커넥션 풀 고갈이라는 더 큰 문제가 생긴다.

어느 방향이든 **BLPOP을 잡고 대기하는 구조 자체가 문제**다. 어떻게 해결해야될까? 하고 찾은게 **대기열 시스템**이다.

### AS-IS : Redis 분산 락

현재 수강신청 API는 Redis 분산 락(FIFO Lock)으로 동시성을 제어한다.

**흐름**

![sync 시퀸스 다이어그램](/public/images/sync.png)

```
사용자 요청 → BLPOP(락 대기) → SET owner → 비즈니스 로직 → Lua 스크립트 해제
```

요청 1건당 Redis 연산이 4~6회 발생한다 (BLPOP, SET, Lua, DEL 등).

**문제점**

- 18,000 RPS 기준 약 90,000 ops/sec로 단일 Redis 한계(~100,000 ops/sec)에 근접
- 락 경쟁이 심해질수록 타임아웃으로 인한 실패율이 급등 (워커를 늘려도 95% 이상 실패)
- 동기 Redis이므로 BLPOP이 스레드를 블로킹 → FastAPI async 장점 미활용, 처리량이 워커 수에 직접 비례

---

### TO-BE : 대기열 시스템

대기열로 바꾸면 API 서버는 `ZADD` 1번만 하고 즉시 반환한다. 커넥션을 잡고 대기하는 일이 없으므로, 비동기 Redis를 사용해도 풀 고갈 문제가 없고 FastAPI async 장점도 온전히 활용할 수 있다.

```
AS-IS: 요청 → BLPOP 대기(커넥션 점유 최대 30초) → 처리
TO-BE: 요청 → ZADD(즉시 반환, 커넥션 반납)    → 202 응답
```

**흐름**

![async 시퀸스 다이어그램](/public/images/async.png)

```
사용자 요청 → ZADD(큐 삽입, 1회) → 즉시 202 + ticket_id 반환

[별도 Worker 프로세스]
ZPOPMIN → DB 쓰기 (단일 프로세스 직렬 처리)

클라이언트 폴링:
  GET /enrollments/queue/{ticket_id}/status
  ← { status: "pending" } → "처리 중"
  ← { status: "success" } → "신청 완료"
```

**구조상 주의할 점 — 다중 인스턴스에서의 Worker 분리**

RPS 18,000 수준에서는 인스턴스를 추가할 필요가 없어보이지만, 추후 확장되면 Worker가 여러개인 문제가 발생한다.\
```
# 잘못된 구조 (Worker가 인스턴스마다 존재)
API Server A (Worker 포함) → ZPOPMIN → DB 쓰기
API Server B (Worker 포함) → ZPOPMIN → DB 쓰기
                                  ↑
                           두 Worker가 서로 다른 요청을 병렬로 DB 쓰기
                           → 직렬화 보장 상실 → 정원 초과 등 정합성 문제 재발
```

올바른 구조는 Worker를 단일 독립 서비스로 분리하는 것이다.

```
[LB]
 ├── API Server A → ZADD만 (큐에 넣기)
 ├── API Server B → ZADD만 (큐에 넣기)
 │
 [공유 Redis ZSET]
 │
 └── Worker Server (단일 프로세스) → ZPOPMIN → DB 쓰기
```

API 서버는 큐 삽입 역할만, Worker는 단일 프로세스로 분리해야 직렬화가 보장된다.

**부하 테스트 측정 결과 (wrk 4t/100c/30s, Docker standalone)**

| | 1 worker + 락 | 4 workers + 락 | 1 worker + 대기열(sync) | 4 workers + 대기열(sync) | 1 worker + 대기열(async) | 4 workers + 대기열(async) |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| RPS | 300 | 840 | 1,484 | 2,893 | 2,807 | **6,896** |
| Avg Latency | 339ms | 140ms | 68ms | 35ms | 36ms | **15ms** |
| Max Latency | 1,450ms | 826ms | 299ms | 164ms | 169ms | 84ms |

> 맥북 air M2의 cpu코어는 8코어로, wrk 테스트 + docker 등 타 프로세스까지 실행 중이여서 4코어까지 선형적인 성능 향상이 이뤄지고, 이후로는 성능이 비슷하거나, 더 떨어졌다. 따라서 4코어를 기준으로 테스트를 진행한다.

async Redis + `async def` 엔드포인트로 전환하자 thread pool 한계가 사라지면서, 1 worker만으로 sync 4 worker 수준(2,807 vs 2,893)이 됐다. 4 worker 기준으로는 sync 대비 약 2.4배(6,896 vs 2,893) 향상됐다.

**트레이드오프 요약**

| 관점 | AS-IS (락) | TO-BE (대기열) |
|---|---|---|
| 처리량 | 낮음 (락 경쟁 → 타임아웃) | 높음 (직렬화 → 실패 없음) |
| 응답 방식 | 동기 (즉시 성공/실패) | 비동기 (폴링 필요) |
| 실패 모드 | 단순 | 복잡 (Worker 장애, Redis OOM 대응 필요) |
| 인프라 복잡도 | 낮음 | 높음 (Worker 서비스 별도 운영) |
| UX | 즉시 결과 확인 가능 | 수 초간 "처리 중" 화면 노출 |

---

## 결론

이번 과제를 통해 동시성 제어를 단계적으로 고도화하는 과정을 직접 경험했다.

처음에는 단순한 Redis Lock으로 시작했다. 이내 polling 방식이 FIFO를 보장하지 않는다는 걸 알게 됐고, BLPOP 기반 FIFO Lock으로 전환했다. 하지만 FIFO Lock을 구현하면서 원자성, Race Condition, 토큰 유실이라는 예상치 못한 문제들을 연달아 만났다. 각 문제를 하나씩 해결하는 과정에서 Redis의 동작 원리를 깊게 이해하게 됐다.

과제가 끝나고 나서 성능을 분석하다 보니, 동기 Redis + BLPOP 구조가 FastAPI의 async 장점을 전혀 활용하지 못한다는 사실을 발견했다. 이게 대기열 시스템을 검토하게 된 출발점이었다. 결론적으로 BLPOP으로 커넥션을 잡고 대기하는 구조 자체가 한계였고, ZADD로 즉시 반환하는 대기열 방식이 처리량과 안정성 모두에서 우위였다.

**배운 것:**

> 설계가 엔지니어의 핵심 덕목이 될 것이라 생각한다.\
> 병목 지점을 점검하고, 발생 가능한 시나리오를 구상한다. 해당 지점에서 발생할 문제를 미리 발견하고 예방한다.\
> 이번 과제는 그 사고방식을 실제로 적용해본 경험이었다.

이 과제는 Claude Code를 활용해 개발했다. 직접 코드를 치는 시간보다 "왜 이렇게 동작하는가"를 이해하고 검증하는 데 더 많은 시간을 썼다. AI와 함께 개발할수록 설계 판단력이 더 중요해진다는 걸 체감했다.

**다음에 해보고 싶은 것:**

비지니스 로직이 단순하다보니 싱글 인스턴스로도 가능해보이지만, 실제 비지니스 로직에서는 요청 처리량이 더 줄어들 것이다.\
18,000개의 요청을 정말 다 소화해야되는 것이라면 스케일업을 하거나, N개의 인스턴스로도 스케일 아웃이 필요해보인다.\
N개의 인스턴스와 1개의 Worker 구성을 구현해보고 싶다. 
