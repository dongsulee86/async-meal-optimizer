1. 시스템 아키텍처 개요 (Architecture Overview)
본 시스템은 외부 프레임워크와 인프라의 변경이 비즈니스 코어 로직에 영향을 주지 않도록 Clean Architecture (클린 아키텍처) 패턴을 따르며, 의존성 역전 원칙(DIP, Dependency Inversion Principle)을 철저히 준수합니다.

🏗️ 레이어별 정의 및 Java Spring 매칭
Presentation Layer (src/app)

역할: 외부의 HTTP(에이치티티피) 요청을 받고 응답을 반환하는 진입점입니다.

기술 스택: FastAPI (패스트에이피아이), Uvicorn (유비콘)

Java 매칭: Spring Boot Controller (스프링 부트 컨트롤러)

Domain Layer (src/domain)

역할: 어떤 인프라(DB, AI 프레임워크)에도 의존하지 않는 순수한 비즈니스 규칙과 엔티티 상태, 최적화 알고리즘의 핵심 코어입니다.

기술 스택: Pure Python (순수 파이썬)

Java 매칭: Domain Entity & Core Service (순수 자바 비즈니스 로직 및 엔티티)

Infrastructure Layer (src/repository, src/ai_interface)

역할: 데이터베이스 영속화, 외부 API 통신, Vector DB 검색 등 물리적인 I/O 작업을 수행하는 최외곽 레이어입니다.

기술 스택: PostgreSQL (포스트그레스큐엘), SQLAlchemy Async (에스큐엘알케미 어싱크), httpx (에이치티티피엑스), ChromaDB (크로마디비)

Java 매칭: Spring Data JPA Repository (스프링 데이터 제이피에이 리포지토리) 및 외부 Client (클라이언트) 구현체

2. 런타임 환경 (Runtime Environment)
Concurrency Model (동시성 모델): 단일 스레드 이벤트 루프(Single Thread Event Loop) 기반 비동기 논블로킹(Non-blocking) 아키텍처를 채택하여, 무거운 최적화 시뮬레이션 연산 및 외부 API 통신 시 스레드 차단(Blocking) 없이 자원을 최적화합니다.

Database Isolation (데이터 격리): 동시성 제어가 필요한 냉장고 재고(stock_items) 자산에 대해 Optimistic Lock (낙관적 락) 메커니즘(version 컬럼 체크)을 적용하여, 분산 환경에서의 데이터 무결성을 애플리케이션 레벨에서 가볍고 안전하게 보장합니다.

3. 핵심 데이터 파이프라인 및 연산 흐름 (Data Flow)
한 달 치 식단 최적화 및 생성 요청이 들어왔을 때, 시스템의 I/O 비용을 최소화하고 성능을 극대화하기 위한 In-Memory Batch Simulation (인메모리 배치 시뮬레이션) 흐름도입니다.

Plaintext
[Client / UI]
      │
      │ (1) POST /api/v1/meal-plans (한 달 치 식단 생성 요청)
      ▼
[Presentation Layer: FastAPI Router]
      │
      │ (2) 비동기 호출 (async/await 이벤트 루프 진입)
      ▼
[Domain Layer: Optimization Service]
      │
      ├─(3) 단 한 번의 비동기 쿼리로 대량 인출 (Bulk Fetch) ──────┐
      │                                                           ▼
      │                                            [Infrastructure Layer]
      ├─(4) 외부 어린이집 영양성분 비동기 조회 (httpx) ────────►  - PostgreSQL (재고 원장)
      │                                                           - 식약처 Open API
      ▼                                                           - Vector DB (유사도 검색)
[In-Memory Snapshot (Python Objects)]                             ▲
      │                                                           │
      ├─(5) 가상 시뮬레이션 루프 구동 (I/O 트래픽 0%)                │
      │     - 알레르기/선호도 하드 필터링                         │
      │     - 유통기한 임박 재료 우선순위 배치                    │
      │     - 재고 가상 차감 및 장보기 리스트 추출                 │
      │                                                           │
      ▼                                                           │
[Final Meal Plan & Stock Adjustments]                             │
      │                                                           │
      └─(6) 최종 결과 데이터 트랜잭션 묶음으로 한 번에 영속화 ────┘
            (Bulk Insert / Bulk Update)
