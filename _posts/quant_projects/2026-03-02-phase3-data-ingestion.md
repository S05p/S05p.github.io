---
layout: default
title: "퀀트 트레이딩 시스템: Phase 3 — Data Ingestion (3편)"
date: 2026-03-02
categories: [quant]
description: "실시간 시세 데이터를 수집해 Kafka와 TimescaleDB로 흘려보내는 파이프라인 설계와 구현을 기록합니다."
---

Data Ingestion이 왜 첫 번째 서비스인지는 2편에서 이야기했다. 데이터가 없으면 백테스트도, 트레이딩도, 인사이트도 없다. 나머지 모든 서비스가 TimescaleDB에 쌓인 시세 데이터를 전제로 동작한다.

이번 편에서는 실제 구현 내용을 기록한다.

---

## 서비스 구조

Data Ingestion은 REST API가 없다. 사용자 요청을 받는 서비스가 아니라 **상시 동작하는 백그라운드 프로세스**다.

```
data-ingestion/
├── app/
│   ├── broker/
│   │   ├── base.py      # TickData + BrokerDataSource 추상 인터페이스
│   │   ├── mock.py      # 랜덤 데이터 생성 (외부 API 없음)
│   │   ├── polling.py   # REST 폴링 → WebSocket처럼 동작
│   │   └── factory.py   # 환경 변수로 구현체 선택
│   ├── producer.py      # broker.stream() → Kafka 발행
│   ├── consumer.py      # Kafka → TimescaleDB 저장
│   ├── gap_detector.py  # 갭 탐지 + yfinance 백필
│   └── repository.py    # DB 접근 레이어
```

세 개의 독립적인 루프가 동시에 돌아간다.

```python
# app/main.py
async def main() -> None:
    broker = get_broker()  # 환경 변수로 구현체 결정

    await asyncio.gather(
        run_producer(broker),  # 루프 1: 브로커 시세 → Kafka 발행
        run_consumer(),        # 루프 2: Kafka 소비 → TimescaleDB 저장
        run_gap_scheduler(),   # 루프 3: 6시간마다 갭 탐지 + 백필
    )
```

`asyncio.gather()`는 세 코루틴을 단일 이벤트 루프에서 동시에 실행한다. 스레드가 아니라 협력적 멀티태스킹이다. 각 루프가 `await`를 만날 때마다 이벤트 루프가 다른 루프를 실행할 기회를 얻는다.

---

## 루프 1: BrokerDataSource — 폴링을 WebSocket처럼

처음에는 `mock_producer.py`에서 랜덤 데이터를 생성해 Kafka에 직접 발행했다. 이 구조의 문제는 나중에 실제 WebSocket으로 교체할 때 코드를 통째로 바꿔야 한다는 것이다.

대신 **브로커 데이터 소스를 추상 인터페이스로 분리**했다.

### 추상 인터페이스

```python
# app/broker/base.py
class TickData(BaseModel):
    model_config = ConfigDict(frozen=True)  # 불변 처리

    ticker: str
    timestamp: datetime
    price: float
    volume: int
    bid: float
    ask: float

    @field_validator("price", "bid", "ask")
    @classmethod
    def must_be_positive(cls, v: float) -> float:
        if v <= 0:
            raise ValueError("price/bid/ask must be positive")
        return v

    @field_validator("volume")
    @classmethod
    def must_be_non_negative(cls, v: int) -> int:
        if v < 0:
            raise ValueError("volume must be non-negative")
        return v


class BrokerDataSource(ABC):
    @abstractmethod
    def stream(self, tickers: list[str]) -> AsyncIterator[TickData]:
        """tickers를 구독하고 TickData를 지속적으로 yield."""
        ...
```

`TickData`를 `@dataclass` 대신 `BaseModel`로 구현한 이유가 있다. `@dataclass`는 조금 더 가볍지만, 외부 입력(브로커 API 응답)을 처리하는 경계 객체에서는 `@field_validator`로 잘못된 값(음수 가격, 음수 거래량)을 즉시 차단할 수 있는 BaseModel이 맞다. `ConfigDict(frozen=True)`로 불변성은 동일하게 보장한다.

`BrokerDataSource`를 사용하는 쪽은 `async for tick in broker.stream(tickers):`로만 쓴다. 데이터가 어디서 오는지 알 필요가 없다.

### Mock 구현체 — 랜덤 데이터

```python
# app/broker/mock.py
class MockBrokerDataSource(BrokerDataSource):
    async def stream(self, tickers: list[str]) -> AsyncIterator[TickData]:
        while True:
            for ticker in tickers:
                base = _BASE_PRICES.get(ticker, 100.0)
                price = round(base * (1 + random.uniform(-0.005, 0.005)), 2)

                yield TickData(
                    ticker=ticker,
                    timestamp=datetime.now(tz=timezone.utc),
                    price=price,
                    volume=random.randint(1_000, 50_000),
                    bid=round(price - 0.02, 2),
                    ask=round(price + 0.02, 2),
                )

            await asyncio.sleep(self._interval)
```

외부 API가 전혀 없다. 개발 환경에서 Kafka, TimescaleDB, 전체 파이프라인을 테스트할 때 쓴다.

### Polling 구현체 — REST를 WebSocket처럼

핵심 아이디어다. **N초마다 REST API를 호출하되, 결과를 WebSocket push처럼 `yield`한다.**

```python
# app/broker/polling.py
class PollingBrokerDataSource(BrokerDataSource):
    async def stream(self, tickers: list[str]) -> AsyncIterator[TickData]:
        while True:
            for ticker in tickers:
                tick = await self._fetch_tick(ticker)  # REST 호출
                if tick is not None:
                    yield tick  # ← WebSocket push처럼 yield

            await asyncio.sleep(self._interval)

    async def _fetch_tick(self, ticker: str) -> TickData | None:
        """현재는 yfinance. 실제 브로커 REST API로 교체 시 이 메서드만 바꾼다."""
        try:
            loop = asyncio.get_event_loop()
            fast_info = await loop.run_in_executor(
                None,
                lambda: yf.Ticker(ticker).fast_info,  # 동기 → 스레드 풀
            )
            price = float(fast_info.last_price)
            spread = round(price * 0.0001, 2)

            return TickData(
                ticker=ticker,
                timestamp=datetime.now(tz=timezone.utc),
                price=price,
                volume=int(fast_info.three_month_average_volume or 0),
                bid=round(price - spread, 2),
                ask=round(price + spread, 2),
            )
        except Exception:
            return None  # 개별 종목 실패는 전체 스트림을 멈추지 않는다
```

`yfinance`는 동기 라이브러리다. `run_in_executor`로 스레드 풀에 넘겨야 이벤트 루프가 블락되지 않는다.

실제 증권사 REST API(`Alpaca`, `Polygon.io`, `IBKR`)로 교체할 때는 `_fetch_tick()` 하나만 바꾸면 된다. 나머지 코드는 전혀 건드리지 않아도 된다.

### 구현체 선택 — Factory

```python
# app/broker/factory.py
def get_broker() -> BrokerDataSource:
    provider = settings.BROKER_PROVIDER  # 환경 변수

    if provider == "mock":
        return MockBrokerDataSource(interval_seconds=settings.POLL_INTERVAL_SECONDS)
    if provider == "polling":
        return PollingBrokerDataSource(interval_seconds=settings.POLL_INTERVAL_SECONDS)

    raise ValueError(f"Unknown BROKER_PROVIDER: '{provider}'")
```

```bash
# .env
BROKER_PROVIDER=mock     # 개발: 외부 API 없음
BROKER_PROVIDER=polling  # 스테이징: yfinance REST
BROKER_PROVIDER=websocket  # 프로덕션: 실제 WebSocket (추후)
```

### Producer — 구현체를 모른다

```python
# app/producer.py
async def run_producer(broker: BrokerDataSource) -> None:
    producer = AIOKafkaProducer(bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS)
    await producer.start()

    try:
        async for tick in broker.stream(settings.NASDAQ_100_TICKERS):
            topic = f"{settings.KAFKA_TOPIC_PREFIX}-{tick.ticker}"
            await producer.send_and_wait(topic, _serialize(tick))
    finally:
        await producer.stop()
```

`broker`가 Mock인지 Polling인지 Producer는 알 필요 없다. `broker.stream()`에서 `TickData`가 나오면 Kafka에 발행하면 된다.

나중에 WebSocket 구현체가 생겨도 Producer 코드는 한 줄도 바뀌지 않는다.

### Kafka Topic 설계: 종목별 분리

토픽을 `market-data` 하나로 쓰지 않고 `market-data-AAPL`, `market-data-MSFT`처럼 **종목별로 분리**했다.

이유는 두 가지다.

첫째, **소비자가 필요한 종목만 구독**할 수 있다. Trading Service가 AAPL 전략만 운영한다면 `market-data-AAPL`만 구독하면 된다. 100개 종목을 전부 받아서 필터링할 필요가 없다.

둘째, **종목별로 처리량을 독립적으로 관리**할 수 있다. 거래량이 많은 AAPL과 적은 종목을 같은 토픽에 넣으면 한쪽이 다른 쪽을 지연시킬 수 있다.

---

## 루프 2: Consumer — Kafka에서 읽어 TimescaleDB에 쓰기

### 배치 플러시: 100개 OR 5초

처음에는 100개가 쌓이면 플러시하는 단순한 구조였다. 문제가 있었다.

종목이 10개이고 interval이 5초면, 10개 메시지/5초 = 2개/초. 100개가 쌓이려면 **50초**가 걸린다. 50초 동안 DB에 아무것도 반영되지 않는다.

**"100개 OR 5초, 둘 중 먼저 도달하는 쪽"** 으로 수정했다.

```python
# app/consumer.py
_BATCH_SIZE = 100
_FLUSH_INTERVAL_SEC = 5.0

async def run_consumer() -> None:
    ...
    batch: list[dict] = []

    async def flush() -> None:
        if not batch:
            return
        async with TimescaleSession() as session:
            inserted = await insert_market_data_batch(session, batch)
        batch.clear()

    consumer_iter = consumer.__aiter__()

    while True:
        try:
            # 최대 5초 대기. 그 안에 메시지가 오면 배치에 추가
            msg = await asyncio.wait_for(
                consumer_iter.__anext__(),
                timeout=_FLUSH_INTERVAL_SEC,
            )
            batch.append(_parse(msg.value))

            # 조건 1: 100개 도달 → 즉시 플러시
            if len(batch) >= _BATCH_SIZE:
                await flush()

        except asyncio.TimeoutError:
            # 조건 2: 5초 동안 메시지 없음 → 남은 배치 강제 플러시
            await flush()

        except StopAsyncIteration:
            break
```

`asyncio.wait_for(timeout=5.0)`이 핵심이다. 5초 안에 메시지가 오면 정상 처리, 5초가 지나면 `TimeoutError`를 발생시켜 강제 플러시 경로로 들어간다.

### ON CONFLICT DO NOTHING: 재시도를 안전하게

```python
# app/repository.py
stmt = text("""
    INSERT INTO market_data (time, ticker, price, volume, bid, ask)
    VALUES (:time, :ticker, :price, :volume, :bid, :ask)
    ON CONFLICT (time, ticker) DO NOTHING
""")
```

Consumer가 재시작되면 Kafka에서 마지막 커밋된 offset부터 다시 읽는다. 이미 DB에 저장된 데이터를 다시 INSERT 시도하게 된다.

`ON CONFLICT DO NOTHING`이 없으면 unique constraint 에러로 Consumer가 멈춘다. 이 한 줄이 재시도를 에러 없이 안전하게 만든다. **몇 번을 실행해도 결과가 동일**한 것을 멱등성(idempotency)이라 한다.

---

## 루프 3: 갭 탐지와 백필

WebSocket 연결이 끊어지거나 서비스가 재시작되면 그 사이의 데이터가 누락된다. 이걸 "갭(gap)"이라고 부른다. 갭을 그냥 두면 백테스트가 불완전한 데이터로 실행된다.

### 탐지: "있어야 할 날"과 "실제 DB"를 비교

```python
async def detect_and_record_gaps(lookback_days: int = 7) -> None:
    end   = datetime.utcnow().date()
    start = end - timedelta(days=lookback_days)

    # NYSE 공식 거래일 목록 (공휴일 포함)
    expected_days = set(get_expected_trading_days(start, end))

    for ticker in settings.NASDAQ_100_TICKERS:
        latest = await get_latest_time(session, ticker)  # DB 최신 시각

        for d in sorted(expected_days):
            if d > latest.date():
                # 거래일인데 데이터가 없음 → data_gaps 테이블에 기록
                await record_gap(session, ticker, ...)
```

NYSE 거래일 계산은 `pandas_market_calendars`를 쓰고, 실패 시 평일(월~금)로 폴백한다.

### 백필: yfinance로 빈 구간 채우기

```python
async def run_backfill() -> None:
    gaps = await get_pending_gaps(session, limit=50)

    for gap in gaps:
        df = yf.download(gap["ticker"], start=..., end=..., auto_adjust=True)

        rows = [{
            "time":   ts.to_pydatetime(),
            "ticker": gap["ticker"],
            "price":  float(row["Close"]),
            "volume": int(row["Volume"]),
            "bid":    float(row["Close"]) - 0.01,
            "ask":    float(row["Close"]) + 0.01,
        } for ts, row in df.iterrows()]

        await insert_market_data_batch(session, rows)  # ON CONFLICT DO NOTHING
        await mark_gap_filled(session, gap["id"], "yfinance")
```

6시간마다 `detect_and_record_gaps()` → `run_backfill()` 순서로 실행된다.

---

## 전체 흐름 정리

```
환경 변수 BROKER_PROVIDER
    ↓ get_broker()
    ├── "mock"    → MockBrokerDataSource
    └── "polling" → PollingBrokerDataSource

[Producer: run_producer(broker)]
  async for tick in broker.stream(tickers):
    → Kafka "market-data-{ticker}" 발행

[Consumer: run_consumer()]
  Kafka "market-data-*" 구독
  → wait_for(timeout=5초)로 메시지 대기
  → 100개 OR 5초 → TimescaleDB INSERT (ON CONFLICT DO NOTHING)

[Gap Scheduler: run_gap_scheduler()]
  6시간마다
  → NYSE 거래일 vs DB 데이터 비교
  → 갭 발견 시 data_gaps 기록
  → yfinance로 백필
```

---

## 구현체 교체 시나리오

이 구조의 가치는 교체 비용이 거의 없다는 것이다.

**시나리오 A: Alpaca REST API로 전환 (브로커 교체)**

```python
# app/broker/polling.py의 _fetch_tick()만 수정
async def _fetch_tick(self, ticker: str) -> TickData | None:
    async with httpx.AsyncClient() as client:
        res = await client.get(
            f"https://data.alpaca.markets/v2/stocks/{ticker}/quotes/latest",
            headers={"APCA-API-KEY-ID": settings.ALPACA_API_KEY, ...},
        )
    data = res.json()
    return TickData(ticker=ticker, price=float(data["quote"]["ap"]), ...)
```

`producer.py`, `consumer.py`, `main.py` — 한 줄도 안 바뀐다.

**시나리오 B: 실제 WebSocket으로 전환 (전송 방식 교체)**

```python
# app/broker/websocket.py 신규 작성
class WebSocketBrokerDataSource(BrokerDataSource):
    async def stream(self, tickers: list[str]) -> AsyncIterator[TickData]:
        async with alpaca_websocket_connect() as ws:
            await ws.subscribe(tickers)
            async for message in ws:
                yield TickData.from_ws_message(message)

# app/broker/factory.py에 분기 추가
if provider == "websocket":
    return WebSocketBrokerDataSource()
```

환경 변수 `BROKER_PROVIDER=websocket`으로 바꾸면 끝이다.

---

## 고도화 숙제

Claude에게 해당 서비스를 고도화한다면 어떤게 추가될까 물어봤다.
Phase 3를 완성한 뒤 더 탄탄하게 만들 수 있는 작업들이다. 지금 당장 구현하면 과설계지만, 시스템이 커지면 반드시 마주치게 되는 문제들이다.

---

### 숙제 1: WebSocket 구현체 추가

`WebSocketBrokerDataSource`를 구현하는 것이 첫 번째 실전 과제다. 인터페이스는 이미 설계되어 있다.

핵심 구현 포인트:

- **reconnect with exponential backoff**: 연결이 끊어졌을 때 즉시 재연결하면 서버에 부하가 몰린다. 1초 → 2초 → 4초 → 8초처럼 대기 시간을 지수적으로 늘려야 한다.

```python
async def _connect_with_retry(self) -> None:
    delay = 1.0
    for _ in range(self._max_retries):
        try:
            await self._ws.connect()
            return
        except ConnectionError:
            await asyncio.sleep(delay)
            delay = min(delay * 2, 60)  # 최대 60초
```

- **heartbeat 감지**: 브로커가 보내는 ping에 응답하지 않으면 연결이 끊어진다.
- **재연결 중 갭 기록**: 재연결 시작 시각과 완료 시각을 `data_gaps`에 즉시 기록해야 백필이 가능하다.

---

### 숙제 2: Kafka Dead Letter Queue (DLQ)

Consumer가 특정 메시지를 처리하다가 실패하면 어떻게 되는가?

현재 코드는 에러가 나면 Consumer 루프 자체가 멈춘다. 배치 100개 중 1개가 문제가 있으면 99개가 같이 멈춘다.

**DLQ 패턴**: 처리 실패한 메시지를 별도 토픽(`market-data-dlq`)으로 보내고, 원본 Consumer는 계속 진행한다.

```
정상 메시지 → TimescaleDB 저장 → offset commit
실패 메시지 → DLQ 토픽 발행  → offset commit (멈추지 않음)

DLQ Consumer (별도)
→ 실패 메시지를 나중에 검토하거나 수동 재처리
```

---

### 숙제 3: Kafka Consumer Lag 모니터링

**Consumer Lag**는 "Producer가 발행한 최신 offset"과 "Consumer가 처리한 offset"의 차이다.

```
Producer: offset 10,000번까지 발행
Consumer: offset 9,500번까지 처리
→ Lag = 500 (파이프라인이 500개 밀려있음)
```

Lag가 계속 증가하면 Consumer가 처리 속도를 따라가지 못하고 있다는 신호다. Prometheus + Grafana로 대시보드에 표시하면 실시간으로 파악할 수 있다.

---

### 숙제 4: TimescaleDB Continuous Aggregates

현재는 raw tick 단위로 저장한다. 백테스트에서 "1분봉"이 필요하면 그 분의 tick들을 모두 읽어 직접 집계해야 한다.

**Continuous Aggregates**는 TimescaleDB가 집계를 미리 계산해서 별도 뷰에 저장하는 기능이다.

```sql
CREATE MATERIALIZED VIEW market_data_1m
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    ticker,
    first(price, time) AS open,
    max(price)         AS high,
    min(price)         AS low,
    last(price, time)  AS close,
    sum(volume)        AS volume
FROM market_data
GROUP BY bucket, ticker;
```

백테스트가 이 뷰를 조회하면 집계 연산 없이 바로 OHLCV 데이터를 얻는다.

---

### 숙제 5: 장 운영시간 체크

현재 Mock/Polling은 24시간 계속 동작한다. 실제 WebSocket으로 전환하면 NYSE 운영시간(ET 09:30~16:00)에만 데이터가 들어온다.

`stream()` 내부에서 장 운영시간을 체크해 **장 외 시간에는 연결을 끊고 대기**하는 로직이 필요하다. 운영시간 외에 연결을 유지하면 불필요한 API 비용이 발생한다.

---

### 숙제 6: Kafka Producer Acknowledgment 튜닝

현재 기본값 `acks=1` — 리더 파티션에 기록되면 완료로 간주한다.

금융 데이터에서는 `acks=all`을 고려해야 한다. 리더 + 모든 팔로워 파티션에 복제가 완료되어야 완료로 간주한다. 리더가 죽어도 데이터가 유실되지 않는다.

```python
producer = AIOKafkaProducer(
    bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
    acks="all",
    enable_idempotence=True,  # 재시도 시 중복 발행 방지
)
```

---

### 숙제 7: NASDAQ 100 종목 목록 자동 갱신

현재 `settings.NASDAQ_100_TICKERS`는 하드코딩된 리스트다. NASDAQ 100은 분기마다 리밸런싱된다.

종목이 추가되면 `data_gaps`에 기록하고 백필을 실행해야 한다. 종목이 제거되면 해당 종목의 구독을 해제해야 한다. 분기마다 구성 종목을 자동으로 갱신하는 로직이 필요하다.

---

## 마치며

Phase 3의 핵심 설계 결정 두 가지를 정리하면:

**1. BrokerDataSource 추상화**
폴링이든 WebSocket이든 호출부는 `async for tick in broker.stream(tickers)`로만 쓴다. 구현체 교체 비용이 거의 없다. 환경 변수 하나로 개발(Mock) → 스테이징(Polling) → 프로덕션(WebSocket) 전환이 가능하다.

**2. 배치 플러시: 100개 OR 5초**
단순한 "N개 쌓이면 플러시"는 메시지가 적을 때 오랫동안 DB에 반영되지 않는 문제가 생긴다. `asyncio.wait_for(timeout=5초)`로 타임아웃 기반 강제 플러시를 추가해 실시간성을 확보했다.

다음 포스트는 Phase 4 — Strategy Service다. 전략 로직을 코드로 관리하고 파라미터는 DB에 저장하는 **코드 레지스트리 패턴**, Backtrader 전략 3개 구현을 다룰 예정이다.
