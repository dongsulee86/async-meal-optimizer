---

### 2. `docs/02_db_schema_specification.md`

```markdown
# 📄 Database Schema Specification

* **Database Engine:** PostgreSQL 15+ (with `pgvector` Extension)
* **Naming Convention:** Tables/Columns - Snake Case (스네이크 케이스)
* **Key Strategy:** All Primary Keys use `UUIDv4`

---

## 1. Master Data Layer (기준 데이터 레이어)

### ① `ingredients` (식재료 마스터)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 식재료 고유 식별자 |
| `name` | `VARCHAR(100)` | NOT NULL, UNIQUE | 식재료명 (ex: 소고기, 당근) |
| `category` | `VARCHAR(50)` | NOT NULL | 분류 (ex: 육류, 채소, 수산물) |
| `default_expiry_days` | `INTEGER` | NOT NULL | 표준 보관 가능 일수 (기본 유통기한) |

### ② `menus` (메뉴 마스터)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 메뉴 고유 식별자 |
| `name` | `VARCHAR(150)` | NOT NULL, UNIQUE | 메뉴명 (ex: 시금치계란말이) |
| `is_snack` | `BOOLEAN` | DEFAULT FALSE | 간식 여부 (True: 간식, False: 식사) |

### ③ `menu_ingredients` (메뉴-식재료 레시피 매핑)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `menu_id` | `UUID` | FK (`menus.id`), PK | 해당 메뉴 식별자 |
| `ingredient_id` | `UUID` | FK (`ingredients.id`), PK | 소요되는 식재료 식별자 |
| `required_amount` | `NUMERIC(6,2)` | NOT NULL | 필요 수량 (g 또는 ml 단위) |

---

## 2. User & Baby Layer (사용자 및 아기 프로필 레이어)

### ④ `users` (부모/사용자 계정)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 사용자 고유 식별자 |
| `email` | `VARCHAR(255)` | NOT NULL, UNIQUE | 로그인 이메일 |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | 계정 생성일시 |

### ⑤ `babies` (아기 프로필)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 아기 고유 식별자 |
| `user_id` | `UUID` | FK (`users.id`) | 부모(사용자) 식별자 |
| `name` | `VARCHAR(50)` | NOT NULL | 아기 이름 |
| `birth_date` | `DATE` | NOT NULL | 생년월일 (개월수 계산용) |

### ⑥ `baby_conditions` (아기 특이사항 및 알레르기)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 특이사항 고유 식별자 |
| `baby_id` | `UUID` | FK (`babies.id`) | 대상 아기 식별자 |
| `ingredient_id` | `UUID` | FK (`ingredients.id`) | 대상 식재료 식별자 |
| `condition_type` | `VARCHAR(20)` | NOT NULL | 상태 분류 (`ALLERGY`, `LIKE`, `DISLIKE`) |

---

## 3. Operation & Tracking Layer (오퍼레이션 및 이력 레이어)

### ⑦ `stock_items` (냉장고 실재고 관리)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 재고 개별 식별자 |
| `ingredient_id` | `UUID` | FK (`ingredients.id`) | 식재료 마스터 식별자 |
| `purchased_at` | `DATE` | NOT NULL | 마트 구매/입고일 |
| `expired_at` | `DATE` | NOT NULL | 유통기한 만료일 (알고리즘 1순위 조건) |
| `total_amount` | `NUMERIC(8,2)` | NOT NULL | 최초 구매 용량 (g, ml 등) |
| `remaining_amount` | `NUMERIC(8,2)` | NOT NULL | 현재 냉장고에 남은 용량 |
| `status` | `VARCHAR(20)` | DEFAULT 'AVAILABLE' | 상태 (`AVAILABLE`, `CONSUMED`, `WASTED`) |
| `version` | `INTEGER` | DEFAULT 1, NOT NULL | 낙관적 락(Optimistic Lock) 버전 관리용 |

### ⑧ `meal_plans` (식단 스케줄 마스터)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 식단 마스터 식별자 |
| `baby_id` | `UUID` | FK (`babies.id`) | 대상 아기 식별자 |
| `start_date` | `DATE` | NOT NULL | 스케줄 시작일 |
| `end_date` | `DATE` | NOT NULL | 스케줄 종료일 (7일 혹은 30일 단위) |
| `status` | `VARCHAR(20)` | DEFAULT 'DRAFT' | 플랜 상태 (`DRAFT`, `APPROVED`) |

### ⑨ `daily_meals` (일별 세부 식단 및 벡터 연동)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 일별 식단 고유 식별자 |
| `meal_plan_id` | `UUID` | FK (`meal_plans.id`) | 부모 식단 마스터 식별자 |
| `date` | `DATE` | NOT NULL | 식단 해당 날짜 |
| `daycare_lunch_raw` | `TEXT` | NULL | 어린이집 점심 원본 텍스트 (**AI 시맨틱 검색 대상**) |
| `home_breakfast_id` | `UUID` | FK (`menus.id`) | 추천된 아침 메뉴 |
| `home_dinner_id` | `UUID` | FK (`menus.id`) | 추천된 저녁 메뉴 |

### ⑩ `meal_histories` (식사 수행 이력 및 피드백)
| 컬럼명 | 데이터 타입 | 제약 조건 | 설명 |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | PRIMARY KEY | 이력 고유 식별자 |
| `baby_id` | `UUID` | FK (`babies.id`) | 대상 아기 식별자 |
| `daily_meal_id` | `UUID` | FK (`daily_meals.id`) | 대상 일별 식단 식별자 |
| `eat_status` | `VARCHAR(20)` | NOT NULL | 섭취 결과 (`COMPLETED`, `PARTIAL`, `SKIP`) |
| `leftover_ratio` | `NUMERIC(3,2)` | DEFAULT 0.00 | 남긴 비율 (0.00 ~ 1.00 / 재고 보정 연동) |
| `reaction_note` | `TEXT` | NULL | 특이사항 및 선호도 피드백 메모 |
