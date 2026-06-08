# 📄 RESTful API Endpoints Specification

## 1. API 디자인 원칙
* **Base URL:** `/api/v1`
* **Format:** Request & Response 모두 `JSON` (제이슨) 포맷을 사용합니다.
* **Status Code (상태 코드):**
  * `200 OK`: 요청 성공
  * `201 Created`: 생성 성공
  * `400 Bad Request`: 비즈니스 로직 검증 실패 (ex: 알레르기 유발 물질 포함 메뉴 강제 주입 시)
  * `409 Conflict`: 동시성 제어 실패 (낙관적 락 버전 충돌)

---

## 2. 엔드포인트 명세 (Endpoints List)

### ① 식단 스케줄 (Meal Plans)

#### [POST] /api/v1/meal-plans
* **설명:** 특정 아기의 개월수 및 냉장고 재고 유통기한을 기반으로 주간/월간 배치 식단을 인메모리로 최적화하여 대량 생성합니다.
* **Request Body (요청 본문):**
```json
{
  "baby_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "start_date": "2026-06-01",
  "end_date": "2026-06-30"
}
```
* **Response (201 Created - 응답 성공):**
```json
{
  "meal_plan_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "status": "DRAFT",
  "total_generated_days": 30
}
```

#### [GET] /api/v1/meal-plans/{meal_plan_id}/daily
* **설명:** 생성된 마스터 플랜 하위의 일자별 상세 식단 리스트(어린이집 메뉴, 아침/저녁 추천 메뉴)를 조회합니다.
* **Response (200 OK):**
```json
[
  {
    "date": "2026-06-08",
    "daycare_lunch": "쇠고기버섯불고기, 백김치",
    "recommended_breakfast": "계란채소죽",
    "recommended_dinner": "갈치구이, 시금치나물"
  }
]
```

---

### ② 냉장고 재고 관리 (Inventory)

#### [POST] /api/v1/inventory/stock-items
* **설명:** 마트에서 장을 본 식재료 재고를 냉장고 원장에 등록합니다. (유통기한 자동 계산용)
* **Request Body:**
```json
{
  "ingredient_id": "ca73b32c-3f2d-4581-9fbd-c619cbdf5f12",
  "purchased_at": "2026-06-08",
  "total_amount": 500.00
}
```

#### [PUT] /api/v1/inventory/stock-items/{id}
* **설명:** 수동 보정 또는 아기의 식사 섭취 피드백에 의해 재고 수량을 수정합니다. (낙관적 락 `version` 검증 적용)
* **Request Body:**
```json
{
  "remaining_amount": 350.00,
  "version": 1
}
```
* **Response Error (409 Conflict - 충돌 에러):** 타 사용자가 동시에 수정하여 버전이 맞지 않는 경우 에러를 반환합니다.

---

### ③ 취식 피드백 및 이력 (Meal History)

#### [POST] /api/v1/meal-histories
* **설명:** 아기의 실제 취식 상태(완식 여부, 남긴 비율)를 기록하고, 이에 연동되어 냉장고 실재고 잔여 용량이 자동으로 비동기 보정 처리됩니다.
* **Request Body:**
```json
{
  "baby_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "daily_meal_id": "8c2da53b-2e3d-4cda-8bfe-1c9c3e21ba5f",
  "eat_status": "PARTIAL",
  "leftover_ratio": 0.50,
  "reaction_note": "오전 어린이집 메뉴와 겹쳐서 저녁 고기반찬을 반쯤 남김"
}
```
