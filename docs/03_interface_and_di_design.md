# 📄 Abstract Interface & Dependency Injection Specification

## 1. 의존성 역전 원칙 (DIP, Dependency Inversion Principle) 적용 정책
본 시스템은 비즈니스 로직(Domain Layer)이 데이터베이스(PostgreSQL)나 AI 라이브러리(LangChain/Ollama) 같은 구체적인 인프라 기술에 종속되지 않도록 **추상 인터페이스를 선언하고 이를 상속받아 구현하는 구조**를 취합니다.

* **Java 매칭:** Java의 `interface`를 선언하고 하위에 `Impl` 클래스를 구현하여 `Spring Container`가 의존성을 주입(DI)해 주는 메커니즘과 완전히 동일합니다.
* **Python 구현 방식:** `abc` (Abstract Base Classes) 라이브러리를 활용해 추상 메서드를 정의하고, 의존성 주입은 FastAPI의 내장 시스템인 `Depends` (디펜즈)를 활용합니다.

---

## 2. 핵심 추상 인터페이스 정의 (Abstract Interfaces)

### ① `MealPlanRepository` (식단 영속성 추상 인터페이스)
* **역할:** 식단 플랜 마스터 및 일별 플랜의 조회/저장을 추상화합니다.
* **Java 매칭:** `public interface MealPlanRepository extends JpaRepository<MealPlan, UUID>`

### ② `StockRepository` (재고 관리 추상 인터페이스)
* **역할:** 냉장고 실재고 상태 조회, 벌크 업데이트 및 낙관적 락 버전을 검증합니다.
* **Java 매칭:** `public interface StockRepository`

### ③ `AiServingInterface` (AI 에이전트 서빙 인터페이스)
* **역할:** 어린이집 식단 파싱 및 시맨틱 유사도 추출을 위한 LLM/Vector DB 연동을 추상화합니다. 이 격리 레이어 덕분에 2027년 AI 팀 합류 시 백엔드 코드 수정 없이 엔진 이식이 가능합니다.

---

## 3. 의존성 주입(DI) 컨테이너 및 런타임 흐름

FastAPI의 `Depends` 구문을 이용하여 런타임 시점에 실제 구체 클래스 인스턴스(SQLAlchemy 비동기 세션, 로컬 에이전트 인스턴스)를 서비스 레이어에 주입합니다.

```text
[HTTP Request]
      │
      ▼
[FastAPI Router (Presentation)]
      │
      │ ──► Inject: Depends(get_meal_plan_service)
      ▼
[MealPlanService (Domain)]
      │
      │ ──► Inject: Depends(get_postgres_stock_repo)
      │ ──► Inject: Depends(get_langchain_ai_agent)
      ▼
[Infrastructure Implementations]
  - SqlAlchemyStockRepositoryImpl
  - LangChainOllamaAgentImpl
