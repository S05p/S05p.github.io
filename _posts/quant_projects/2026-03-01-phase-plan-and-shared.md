---
layout: default
title: "퀀트 트레이딩 시스템: Phase 계획 + Phase 1 shared 코드 분리 (2편)"
date: 2026-03-01
categories: [quant]
description: "전체 개발 Phase 로드맵과, Monorepo에서 공통 코드를 서비스 간에 공유하는 방법을 설명합니다."
---

1편에서 아키텍처와 각 결정의 이유를 다뤘다. 이번 편에서는 실제로 어떤 순서로 개발할 것인지, 그리고 Monorepo에서 공통 코드를 어떻게 분리해서 쓰는지를 기록한다.

---

## 전체 Phase 로드맵

| Phase | 내용 | 핵심 결정 |
|---|---|---|
| **1** | Monorepo 뼈대 + 인프라 | Docker Compose, shared editable install |
| **2** | (아키텍처 RFC 작성) | 이 글이 그 결과물 |
| **3** | Data Ingestion | WebSocket 추상화, Kafka fan-out, ON CONFLICT DO NOTHING |
| **4** | Strategy Service | 코드 레지스트리 패턴, 파라미터 Pydantic 검증 |
| **5** | Backtest Service | SAGA Orchestration, Redis Lock, run_in_executor |
| **6** | Trading Service | FSM 주문 상태, 4단계 리스크 체크, Redis 직렬화 락 |
| **7** | Insight Service | LangChain LCEL, Redis 6h TTL, 백테스트 손실 구간 분석 |
| **8** | Auth Service | JWT Rotation, X-User-Id → JWT 교체 |
| **9** | Monorepo 분리 | git subtree split, 내부 PyPI |

Phase 순서에는 이유가 있다. **데이터가 없으면 백테스트도, 트레이딩도 없다.** 그래서 Data Ingestion이 먼저다. 인증은 마지막이다 — `X-User-Id` 헤더로 Phase 8 전까지 임시 처리하면, 나머지 개발 기간 동안 인증 로직 없이 비즈니스 로직에만 집중할 수 있다.

---

## Phase별 포스트 예고

각 Phase가 완료될 때마다 포스트를 업데이트할 예정이다.

- **Phase 3**: WebSocket 연결 관리, 갭 탐지 로직, Kafka producer/consumer 설계
- **Phase 4**: Backtrader 전략 코드 레지스트리 패턴, 파라미터 검증 구조
- **Phase 5**: SAGA Orchestration 구현 코드, Redis 분산 락 TTL 계산법
- **Phase 6**: 주문 FSM, 리스크 체크 레이어 설계
- **Phase 7**: LangChain LCEL 체인 구성, 뉴스 컨텍스트 포맷팅
- **Phase 8**: JWT Refresh Token Rotation 구현, 기존 의존성 함수 교체 전략
- **Phase 9**: git subtree split 실행 과정, shared 라이브러리 버전 관리

---

## Phase 1: Monorepo에서 shared 코드를 어떻게 나눠 쓰는가

### 왜 Monorepo인가

처음부터 5개 서비스를 별도 레포로 분리하면 어떤 일이 생기는지부터 생각해봤다.

공통 스키마인 `BacktestResult`에 필드 하나 추가하면 어떻게 되는가?

```
① shared-lib 레포에서 변경 후 버전 태그 (v0.1.4)
② backtest 레포에서 shared-lib 버전 업 후 PR
③ trading 레포에서 shared-lib 버전 업 후 PR
④ insight 레포에서 shared-lib 버전 업 후 PR
```

MVP 단계에서 API 경계가 아직 확정되지 않은 상태에서 이 오버헤드를 감당할 이유가 없다. Monorepo에서 자유롭게 변경하며 경계를 찾고, 그 경계가 안정되면 분리한다.

---

### shared 디렉토리 구조

```
quant-platform/
├── shared/
│   ├── pyproject.toml
│   └── quant_shared/
│       ├── __init__.py
│       ├── schemas/
│       │   ├── __init__.py
│       │   ├── responses.py      # ApiResponse, PaginatedResponse
│       │   └── backtest.py       # BacktestResult, BacktestStatus
│       ├── exceptions/
│       │   ├── __init__.py
│       │   └── base.py           # NotFoundError, ForbiddenError, ...
│       └── utils/
│           ├── __init__.py
│           └── redis_lock.py     # 분산 락 유틸리티
│
└── services/
    ├── backtest/
    │   └── pyproject.toml
    └── trading/
        └── pyproject.toml
```

shared에 들어가는 것과 들어가지 않는 것의 기준은 명확하다.

| shared에 넣는 것 | 서비스 내부에 두는 것 |
|---|---|
| 서비스 간 공유 Pydantic 스키마 | 서비스별 DB 모델 (SQLAlchemy) |
| 공통 예외 클래스 | 서비스별 비즈니스 로직 |
| Redis 락 유틸리티 | 서비스별 설정 (config.py) |
| API 응답 래퍼 타입 | 서비스별 의존성 주입 |

DB 모델은 shared에 넣지 않는다. 서비스별로 DB 스키마가 달라질 수 있고, 모델을 공유하면 서비스 간 결합도가 높아진다. 서비스 간 데이터 교환은 Pydantic 스키마(DTO)로만 한다.

---

### pip install -e: 로컬 editable 설치

**shared/pyproject.toml**

```toml
[project]
name = "quant-shared"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.0",
    "redis[asyncio]>=5.0",
]
```

**services/backtest/pyproject.toml**

```toml
[project]
name = "backtest-service"
dependencies = [
    "quant-shared",
    "fastapi>=0.110",
    "sqlalchemy[asyncio]>=2.0",
    # ...
]

[tool.uv.sources]
quant-shared = { path = "../../shared", editable = true }
```

`editable = true`가 핵심이다. 설치된 패키지가 실제 소스 경로를 가리킨다. `quant_shared/schemas/responses.py`를 수정하면 재설치 없이 즉시 반영된다.

로컬 개발 환경 설정:

```bash
# 가상환경 생성 후
pip install -e "./shared"
pip install -e "./services/backtest[dev]"

# 이제 어디서든 동작
python -c "from quant_shared.schemas.responses import ApiResponse; print('ok')"
```

---

### Docker 빌드: shared를 컨테이너 안으로

컨테이너는 로컬 파일 시스템이 없으니 shared를 직접 복사해서 설치해야 한다.

**services/backtest/Dockerfile**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# shared 먼저 복사 (레이어 캐시 활용)
COPY shared/ ./shared/
COPY services/backtest/pyproject.toml ./service/pyproject.toml
COPY services/backtest/ ./service/

# shared → editable install, service → editable install
RUN pip install --no-cache-dir -e ./shared && \
    pip install --no-cache-dir -e ./service

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

shared를 먼저 COPY하는 이유가 있다. Docker 레이어 캐시는 COPY 대상이 변경되면 이후 레이어를 무효화한다. shared가 자주 바뀌면 매번 전체 재빌드가 일어난다. 반면 service 코드는 자주 바뀌는데 shared는 상대적으로 안정적이다. 안정적인 것을 앞에 두면 캐시를 더 많이 활용할 수 있다.

---

### mypy와 IDE가 shared를 인식하는 방법

editable install을 쓰면 mypy와 IDE(PyCharm, VSCode)가 패키지 이름으로 추적한다. 상대 경로가 아니라 패키지 이름으로 import하기 때문에 타입 체크가 완전히 동작한다.

```python
# services/backtest/app/schemas.py
from quant_shared.schemas.responses import ApiResponse  # 패키지 이름으로 import
from quant_shared.exceptions.base import NotFoundError
```

루트 `pyrightconfig.json`에서 경로를 명시해 두면 VSCode에서도 자동완성이 정상 동작한다.

```json
{
  "venvPath": ".",
  "venv": "venv",
  "pythonVersion": "3.11"
}
```

---

### 언제 shared를 독립 라이브러리로 분리하는가

세 조건이 동시에 충족될 때다.

```
□ 서비스별 배포 주기가 달라짐
  → auth는 매주 배포, backtest는 월 1회

□ 팀이 서비스별로 분리됨
  → auth 팀과 trading 팀이 각각 별도로 운영

□ shared 변경이 전체 서비스 테스트를 강제하는 비용이 너무 커짐
  → shared 수정 한 번에 CI가 5개 서비스를 다 돌림
```

이 기준이 오기 전에 분리하면 버전 관리 비용만 늘어난다. "언젠가는 해야 하니까"라는 이유로 미리 분리하지 않는다.

분리할 때는 `git subtree split`으로 shared의 커밋 히스토리를 보존하면서 별도 레포로 추출한다.

```bash
# shared를 별도 브랜치로 추출 (커밋 히스토리 보존)
git subtree split --prefix=shared -b shared-standalone

# 새 레포에 push
git remote add shared-remote https://github.com/org/quant-shared.git
git push shared-remote shared-standalone:main
```

각 서비스의 `pyproject.toml`에서 로컬 경로를 패키지 버전으로 교체한다.

```toml
# 분리 전
[tool.uv.sources]
quant-shared = { path = "../../shared", editable = true }

# 분리 후 (내부 PyPI 또는 GitHub Packages)
[project]
dependencies = ["quant-shared>=1.0.0,<2.0.0"]
```

---

## 마치며

Phase 1은 코드보다 결정이 많은 단계다. 무엇을 어떻게 나눌 것인가, 그 경계가 나중에 얼마나 많은 일을 줄여주는지를 미리 생각해두는 것이 핵심이다.

다음 포스트는 Phase 3 — Data Ingestion 구현이다. WebSocket 연결 관리, Kafka 프로듀서 설계, 갭 탐지 로직을 다룰 예정이다.
