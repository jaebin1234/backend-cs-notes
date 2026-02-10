# SQL 예제 모음 

이 문서는 실제 업무(주문·재고·포인트·결제)에서 자주 나오는 SQL 문제들을 예시와 함께 정리한 것입니다.
각 예제는 다음 구성을 가집니다.

* 문제 설명
* 주요 테이블 구조(간단 버전)
* 예시 쿼리
* 왜 이렇게 썼는지, 유의할 점

---

## 1. 공통 도메인 테이블 개요

예제마다 약간씩 다르게 쓰지만, 기본적으로 아래 도메인들을 가정합니다.

* 고객: customer
* 주문: orders, order_item
* 상품: product
* 재고/원장: stock_ledger 또는 inbound / outbound
* 포인트: point_balance, point_history
* 결제: payment, payment_history

실제 DDL은 회사마다 다르기 때문에 예제에서는 필요한 컬럼만 최소한으로 사용합니다.

---

## 2. (A) 기본 JOIN 문제 – 2024년 1월에 2건 이상 주문한 고객

문제
고객(customer), 주문(orders), 주문상품(order_item) 테이블에서
2024년 1월에 2건 이상 주문한 고객 목록을 조회하라.

핵심 포인트

* JOIN
* GROUP BY
* HAVING

### 2.1 테이블 구조 (예시)

```text
customer
---------
customer_id   PK
name
grade

orders
---------
order_id      PK
customer_id   FK -> customer
order_date
status        (CREATED, PAID, CANCELED 등)

order_item
---------
order_item_id PK
order_id      FK -> orders
product_id    FK -> product
quantity
unit_price
```

### 2.2 예시 쿼리

```sql
SELECT  c.customer_id,
        c.name,
        COUNT(DISTINCT o.order_id) AS order_count
FROM    customer c
JOIN    orders o
          ON c.customer_id = o.customer_id
WHERE   o.order_date >= '2024-01-01'
  AND   o.order_date <  '2024-02-01'
  AND   o.status = 'PAID'
GROUP BY c.customer_id,
         c.name
HAVING  COUNT(DISTINCT o.order_id) >= 2;
```

### 2.3 설명과 유의할 점

* 기간 필터와 상태 조건은 WHERE에서 먼저 걸어서 대상 데이터를 줄입니다.
* 주문 건수는 order_item과의 JOIN 때문에 중복될 수 있으므로 COUNT(DISTINCT o.order_id)를 사용하는 것이 안전합니다.
* GROUP BY에는 고객 기준 컬럼을 넣고, HAVING에서 건수 조건을 체크합니다.

---

## 3. (B) 반품·취소 집계 – 상품별 주문/출고/취소/반품

문제
상품별로 다음 컬럼을 한 줄에 조회하라.

* 주문수량
* 출고수량
* 취소수량
* 반품수량

핵심 포인트

* 여러 테이블 JOIN
* SUM(CASE WHEN…) 조건부 집계
* GROUP BY

### 3.1 테이블 구조 (예시)

```text
product
---------
product_id    PK
name
category_id

orders
---------
order_id      PK
customer_id
order_date
status

order_item
---------
order_item_id PK
order_id      FK -> orders
product_id    FK -> product
quantity

shipment_item         출고 라인
---------
shipment_item_id PK
order_item_id    FK -> order_item
shipped_qty
shipped_at

cancel_item           취소 라인
---------
cancel_item_id  PK
order_item_id   FK -> order_item
cancel_qty
cancel_at

return_item           반품 라인
---------
return_item_id  PK
order_item_id   FK -> order_item
return_qty
return_at
```

### 3.2 예시 쿼리

```sql
SELECT  p.product_id,
        p.name,
        SUM(oi.quantity)                AS ordered_qty,
        COALESCE(SUM(si.shipped_qty),0) AS shipped_qty,
        COALESCE(SUM(ci.cancel_qty),0)  AS canceled_qty,
        COALESCE(SUM(ri.return_qty),0)  AS returned_qty
FROM    product p
JOIN    order_item oi
          ON p.product_id = oi.product_id
JOIN    orders o
          ON oi.order_id = o.order_id
LEFT JOIN shipment_item si
          ON oi.order_item_id = si.order_item_id
LEFT JOIN cancel_item ci
          ON oi.order_item_id = ci.order_item_id
LEFT JOIN return_item ri
          ON oi.order_item_id = ri.order_item_id
WHERE   o.order_date >= '2024-01-01'
  AND   o.order_date <  '2024-02-01'
  AND   o.status IN ('PAID', 'SHIPPED', 'COMPLETED')
GROUP BY p.product_id,
         p.name;
```

### 3.3 설명과 유의할 점

* 주문수량은 order_item 기준으로 SUM(oi.quantity)로 계산합니다.
* 출고·취소·반품은 없을 수도 있으므로 LEFT JOIN을 사용합니다.
* 여러 번 출고·취소·반품될 수 있다는 가정하에 SUM으로 합산합니다.
* COALESCE로 NULL을 0으로 바꿔서 계산과 표시를 편하게 만듭니다.

---

## 4. (C) 재고 원장 – 상품별 누적 재고 계산

문제
입고(inbound), 출고(outbound) 테이블을 이용해
상품별 누적 재고(총 입고 - 총 출고)를 조회하라.

핵심 포인트

* SUM 집계
* OUTER JOIN 또는 단일 원장 구조
* NULL → 0 처리

### 4.1 테이블 구조 (분리 패턴 예시)

```text
inbound
---------
inbound_id   PK
product_id   FK -> product
qty
inbound_at

outbound
---------
outbound_id  PK
product_id   FK -> product
qty
outbound_at
```

### 4.2 예시 쿼리

```sql
SELECT  p.product_id,
        p.name,
        COALESCE(i.total_in_qty, 0)  AS total_in_qty,
        COALESCE(o.total_out_qty, 0) AS total_out_qty,
        COALESCE(i.total_in_qty, 0)
          - COALESCE(o.total_out_qty, 0) AS current_stock
FROM    product p
LEFT JOIN (
    SELECT  product_id,
            SUM(qty) AS total_in_qty
    FROM    inbound
    GROUP BY product_id
) i ON p.product_id = i.product_id
LEFT JOIN (
    SELECT  product_id,
            SUM(qty) AS total_out_qty
    FROM    outbound
    GROUP BY product_id
) o ON p.product_id = o.product_id;
```

### 4.3 설명과 유의할 점

* 입고·출고를 각각 서브쿼리에서 합산한 뒤 product와 LEFT JOIN합니다.
* product 기준으로 LEFT JOIN하므로, 입출고 이력이 없는 상품도 모두 조회할 수 있습니다.
* 단일 원장(stock_ledger) 구조라면 SUM(CASE WHEN tx_type = 'IN' …) 패턴으로 더 간단하게 만들 수 있습니다.

---

## 5. (D) 최근 상태만 조회 – 결제 히스토리

문제
결제 히스토리(payment_history)에서
각 payment_id의 가장 최근 상태만 조회하라.

핵심 포인트

* 윈도우 함수
* ROW_NUMBER
* PARTITION BY

### 5.1 테이블 구조 (예시)

```text
payment
---------
payment_id      PK
user_id
amount
created_at

payment_history
---------
history_id      PK
payment_id      FK -> payment
status          (REQUESTED, PENDING, SUCCESS, FAILED 등)
changed_at
reason
```

### 5.2 예시 쿼리

```sql
WITH last_status AS (
    SELECT  ph.payment_id,
            ph.status,
            ph.changed_at,
            ROW_NUMBER() OVER (
                PARTITION BY ph.payment_id
                ORDER BY     ph.changed_at DESC, ph.history_id DESC
            ) AS rn
    FROM    payment_history ph
)
SELECT  ls.payment_id,
        ls.status,
        ls.changed_at
FROM    last_status ls
WHERE   ls.rn = 1;
```

### 5.3 설명과 유의할 점

* 결제 건(payment_id)별로 히스토리를 그룹핑하고, 시간 역순으로 정렬한 뒤 ROW_NUMBER로 순번을 매깁니다.
* rn = 1만 선택하면 각 결제 건의 최신 상태만 얻을 수 있습니다.
* 주문 상태, 포인트 상태, 정산 상태 등 여러 도메인에서 공통으로 사용되는 패턴입니다.

---

## 6. (E) 월별 가장 많이 팔린 상품 Top-1

문제
2024년 월별로 가장 많이 팔린 상품 1개를 조회하라.

핵심 포인트

* GROUP BY로 월·상품별 집계
* 윈도우 함수로 월별 순위 계산
* PARTITION BY month, ORDER BY qty DESC

### 6.1 테이블 구조

A에서 사용한 orders, order_item, product를 그대로 사용합니다.

### 6.2 예시 쿼리 (PostgreSQL 기준)

```sql
WITH monthly_product_sales AS (
    SELECT  DATE_TRUNC('month', o.order_date) AS month,
            oi.product_id,
            SUM(oi.quantity)                  AS total_qty
    FROM    orders o
    JOIN    order_item oi
              ON o.order_id = oi.order_id
    WHERE   o.order_date >= '2024-01-01'
      AND   o.order_date <  '2025-01-01'
      AND   o.status = 'PAID'
    GROUP BY DATE_TRUNC('month', o.order_date),
             oi.product_id
),
ranked AS (
    SELECT  month,
            product_id,
            total_qty,
            ROW_NUMBER() OVER (
                PARTITION BY month
                ORDER BY     total_qty DESC, product_id
            ) AS rn
    FROM    monthly_product_sales
)
SELECT  r.month,
        r.product_id,
        p.name,
        r.total_qty
FROM    ranked r
JOIN    product p
          ON r.product_id = p.product_id
WHERE   r.rn = 1
ORDER BY r.month;
```

### 6.3 설명과 유의할 점

* 첫 번째 CTE에서 월·상품별 판매 수량을 먼저 집계합니다.
* 두 번째 CTE에서 월별로 ROW_NUMBER를 매기고, rn = 1인 상품만 선택합니다.
* 동률 처리 정책에 따라 정렬 기준을 더 넣거나 RANK/DENSE_RANK를 사용할 수 있습니다.

---

## 7. (F) 정합성 확인 – 포인트 잔여 vs 히스토리 합계

문제
포인트 히스토리에서
잔여포인트보다 history 합계가 더 큰 user_id 목록을 찾아라.

핵심 포인트

* SUM 집계
* JOIN
* WHERE 비교

### 7.1 테이블 구조 (예시)

```text
point_balance
---------
user_id        PK
balance_point  현재 잔여 포인트

point_history
---------
history_id     PK
user_id        FK -> point_balance
tx_type        (CHARGE, USE, CANCEL, EXPIRE 등)
amount         (+ 또는 - 값)
created_at
```

### 7.2 예시 쿼리

```sql
WITH history_sum AS (
    SELECT  user_id,
            SUM(amount) AS history_total
    FROM    point_history
    GROUP BY user_id
)
SELECT  pb.user_id,
        pb.balance_point,
        hs.history_total
FROM    point_balance pb
JOIN    history_sum hs
          ON pb.user_id = hs.user_id
WHERE   hs.history_total <> pb.balance_point;
```

또는 문제에서 요구한 것처럼 history 합계가 잔여포인트보다 큰 경우만 보고 싶다면:

```sql
WHERE hs.history_total > pb.balance_point;
```

### 7.3 설명과 유의할 점

* amount 설계를 일관되게 유지해야 합니다. (충전 +, 사용 -, 소멸 - 등)
* 이 쿼리는 배치나 모니터링 작업에 넣어 데이터 불일치를 조기 발견하는 용도로 사용하기 좋습니다.
* 실제 실무에서는 확정 상태만 집계하기 위해 tx_status 같은 추가 조건이 들어가는 경우가 많습니다.

---

## 8. (G) 성능 최적화 – 함수, 서브쿼리 때문에 인덱스를 못 타는 경우

이 섹션에서는 두 가지 유형을 다룹니다.

* 컬럼에 함수를 씌워서 인덱스를 타지 못하는 경우
* 조건부 서브쿼리로 복잡하게 작성한 포인트 내역 조회를 JOIN/CTE로 풀어서 최적화하는 경우

### 8.1 날짜 함수 때문에 인덱스를 못 타는 경우

문제

```sql
SELECT *
FROM orders
WHERE DATE(created_at) = '2024-01-01';
```

개선안

```sql
SELECT *
FROM   orders
WHERE  created_at >= '2024-01-01 00:00:00'
  AND  created_at <  '2024-01-02 00:00:00';
```

설명

* DATE(created_at)처럼 컬럼에 함수를 씌우면, created_at에 걸려 있는 인덱스를 그대로 사용하지 못하는 경우가 많습니다.
* 인덱스는 보통 원본 컬럼(created_at)에 걸려 있으므로, 컬럼에 함수를 적용하지 않고 범위 조건(>=, <)으로 표현하는 것이 좋습니다.
* 날짜 범위를 잡을 때는 항상 시작은 이상(>=), 끝은 미만(<)으로 잡는 습관을 들이면, 타임존·마이크로초 이슈에 덜 민감해집니다.

### 8.2 조건부 서브쿼리를 JOIN/CTE로 바꾸는 예시 (포인트 내역)

문제
2024년에 포인트를 많이 사용한 유저의 포인트 내역만 보고 싶다고 가정해 봅니다.
조건: 2024년 동안 USE 타입 합계가 100000 이상인 유저의 point_history 전체를 조회.

서브쿼리 버전

```sql
SELECT  ph.*
FROM    point_history ph
WHERE   ph.tx_date >= '2024-01-01'
  AND   ph.tx_date <  '2025-01-01'
  AND   ph.user_id IN (
        SELECT  user_id
        FROM    point_history
        WHERE   tx_date >= '2024-01-01'
          AND   tx_date <  '2025-01-01'
          AND   tx_type = 'USE'
        GROUP BY user_id
        HAVING  SUM(amount) >= 100000
  );
```

JOIN + CTE 버전

```sql
WITH heavy_user AS (
    SELECT  user_id
    FROM    point_history
    WHERE   tx_date >= '2024-01-01'
      AND   tx_date <  '2025-01-01'
      AND   tx_type = 'USE'
    GROUP BY user_id
    HAVING  SUM(amount) >= 100000
)
SELECT  ph.*
FROM    point_history ph
JOIN    heavy_user hu
          ON ph.user_id = hu.user_id
WHERE   ph.tx_date >= '2024-01-01'
  AND   ph.tx_date <  '2025-01-01';
```

설명과 유의할 점

* IN 서브쿼리도 요즘 DB는 내부적으로 JOIN 형태로 재작성하기도 하지만,
  CTE + JOIN 구조로 바꿔 주면 로직이 더 눈에 잘 들어오고, 추가 조건을 붙이거나 디버깅하기가 편해집니다.
* heavy_user CTE에서 인덱스(user_id, tx_date, tx_type)를 잘 타게 만들면, 전체 쿼리 성능에 큰 영향을 줍니다.
* 복잡한 상관 서브쿼리(SELECT 안에서 매 행마다 SUM을 다시 계산하는 패턴)도, 대부분 이렇게 CTE/파생 테이블 + JOIN/윈도우 함수로 바꾸는 것이 좋습니다.
* 포인트·정산 도메인에서는
  “조건을 만족하는 유저 집합을 먼저 구한다 → 그 유저의 전체 히스토리를 가져온다”
  같은 패턴이 자주 등장하므로, 이런 식으로 두 단계로 나누는 구조를 익혀두시면 좋습니다.

---

## 9. 페이징과 정렬 – OFFSET vs 커서(Keyset) 페이지네이션

리스트 화면에서 페이징을 구현할 때 대표적인 방식 두 가지입니다.

* OFFSET 페이지네이션
* 커서(Keyset) 페이지네이션

### 9.1 OFFSET 페이지네이션

문제
주문 리스트를 생성일 내림차순으로 정렬하고,
페이지 단위로 20개씩 보여주되 n번째 페이지를 조회하라.

예시 쿼리

```sql
-- page 파라미터: 1, 2, 3 ...
SELECT  o.order_id,
        o.customer_id,
        o.order_date,
        o.status,
        o.amount
FROM    orders o
WHERE   o.order_date >= '2024-01-01'
ORDER BY o.created_at DESC, o.order_id DESC
LIMIT   20
OFFSET  20 * (:page - 1);
```

설명과 유의할 점

* 구현이 단순하고, 특정 페이지 번호에 임의 접근하기 쉽습니다.
* OFFSET 값이 커질수록 앞의 많은 행을 스캔하고 버려야 해서, 페이지가 깊어질수록 성능이 급격히 나빠집니다.
* 데이터가 실시간으로 계속 추가·삭제되는 환경에서는 같은 page 번호라도 내용이 흔들릴 수 있습니다.

### 9.2 커서(Keyset) 페이지네이션

개념
마지막으로 본 행의 정렬 키(예: created_at, id)를 커서로 넘겨주고,
그 이후 데이터만 조회하는 방식입니다.

첫 페이지 쿼리 예시

```sql
SELECT  o.order_id,
        o.customer_id,
        o.order_date,
        o.status,
        o.amount
FROM    orders o
WHERE   o.order_date >= '2024-01-01'
ORDER BY o.created_at DESC, o.order_id DESC
LIMIT   20;
```

다음 페이지 쿼리 예시
(마지막으로 본 created_at, order_id를 커서로 전달받았다고 가정)

```sql
SELECT  o.order_id,
        o.customer_id,
        o.order_date,
        o.status,
        o.amount
FROM    orders o
WHERE   o.order_date >= '2024-01-01'
  AND  (
        o.created_at < :lastCreatedAt
     OR (o.created_at = :lastCreatedAt AND o.order_id < :lastOrderId)
  )
ORDER BY o.created_at DESC, o.order_id DESC
LIMIT   20;
```

설명과 유의할 점

* OFFSET 없이 “정렬 키 기준으로 이후 데이터만” 조회하기 때문에, 깊은 페이지에서도 성능이 일정하게 유지됩니다.
* created_at, order_id에 복합 인덱스가 있으면 매우 효율적인 스캔이 가능합니다.
* 특정 page 번호로 바로 점프하기는 어렵지만, 보통 모바일/웹 무한 스크롤 방식에는 커서 방식이 더 잘 맞습니다.
* 정렬 키는 유일하게 만들어 주는 것이 좋습니다. (created_at만으로는 동일 시각 데이터가 있을 수 있으니, created_at + id 조합 사용)

### 9.3 OFFSET vs 커서 비교 요약

* OFFSET

  * 장점: 구현이 단순, page=3 같이 바로 접근 가능
  * 단점: deep page에서 느림, 실시간 데이터 변화에 취약
* 커서

  * 장점: deep page에서도 빠름, 스크롤 UX에 적합
  * 단점: page 번호 접근이 어렵고, 커서 관리가 필요

---

## 10. 정리 – 예제에서 사용한 키워드 요약

| 예제 | 도메인         | 핵심 포인트                     | 사용 개념                                          |
| -- | ----------- | -------------------------- | ---------------------------------------------- |
| A  | 고객·주문       | 2건 이상 주문 고객 조회             | JOIN, WHERE, GROUP BY, HAVING                  |
| B  | 주문·출고·취소·반품 | 상품별 주문/출고/취소/반품 집계         | 여러 테이블 JOIN, SUM(CASE), GROUP BY               |
| C  | 재고 원장       | 상품별 누적 재고 계산               | SUM, 서브쿼리, OUTER JOIN, COALESCE                |
| D  | 결제 히스토리     | 결제 건별 최신 상태 조회             | 윈도우 함수, ROW_NUMBER, PARTITION BY               |
| E  | 매출 통계       | 월별 가장 많이 팔린 상품 Top-1       | GROUP BY, 윈도우 함수, 순위 계산                        |
| F  | 포인트 정합성     | 잔여 vs 히스토리 합계 비교           | SUM 집계, JOIN, WHERE 비교                         |
| G  | 성능 튜닝       | 날짜 함수 인덱스 문제, 포인트 서브쿼리 최적화 | 인덱스, 범위 조건, CTE, 조건부 서브쿼리 → JOIN               |
| 9  | 페이징         | OFFSET vs 커서 페이지네이션        | ORDER BY, LIMIT/OFFSET, Keyset Pagination, 인덱스 |

