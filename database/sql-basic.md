# SQL 기본 개념 정리

---

## 1. SQL과 RDB 기본 이해

* SQL은 관계형 데이터베이스(RDB)에서 데이터를 정의(DDL), 조작(DML), 조회(DQL)하는 언어입니다.
* 테이블은 행(row, record)과 열(column, field)로 구성되며, 각 행은 하나의 레코드, 각 열은 하나의 속성을 의미합니다.
* 실무에서 가장 자주 사용하는 명령은 SELECT, INSERT, UPDATE, DELETE, JOIN, GROUP BY, ORDER BY 입니다.

---

## 2. JOIN 기본 (INNER / LEFT / RIGHT)

### 2.1 개념

* INNER JOIN
  양쪽 테이블 모두에 존재하는 행만 조회합니다. (교집합)
* LEFT JOIN
  왼쪽 테이블은 모두 살리고, 오른쪽 테이블이 없으면 NULL로 채웁니다.
* RIGHT JOIN
  오른쪽 테이블은 모두 살리고, 왼쪽 테이블이 없으면 NULL로 채웁니다.

### 2.2 예시

```sql
-- 주문이 있는 고객만
SELECT  c.customer_id,
        c.name,
        o.order_id
FROM    customer c
INNER JOIN orders o
        ON c.customer_id = o.customer_id;
```

```sql
-- 주문 여부와 상관없이 모든 고객
SELECT  c.customer_id,
        c.name,
        o.order_id
FROM    customer c
LEFT JOIN orders o
        ON c.customer_id = o.customer_id;
```

### 2.3 실무 포인트

* 대부분의 실무 조회는 INNER JOIN 또는 LEFT JOIN으로 해결하는 경우가 많습니다.
* RIGHT JOIN은 가독성이 떨어져, LEFT JOIN 기준으로 테이블 순서를 바꿔서 쓰는 편입니다.
* LEFT JOIN 후 WHERE에 오른쪽 테이블 컬럼 조건을 잘못 걸면 사실상 INNER JOIN처럼 동작하니 주의하셔야 합니다.

```sql
-- 사실상 INNER JOIN처럼 되는 패턴 (주의)
SELECT ...
FROM   customer c
LEFT JOIN orders o ON ...
WHERE  o.order_id IS NOT NULL;
```

---

## 3. WHERE vs HAVING

### 3.1 개념

* WHERE
  그룹핑과 집계 전에 행을 필터링하는 구문입니다.
* HAVING
  GROUP BY 이후, 집계 결과에 조건을 거는 구문입니다.

### 3.2 예시

```sql
-- 2024년 주문 중, 2건 이상 주문한 고객
SELECT  customer_id,
        COUNT(*) AS order_count
FROM    orders
WHERE   order_date >= '2024-01-01'
  AND   order_date <  '2025-01-01'
GROUP BY customer_id
HAVING  COUNT(*) >= 2;
```

### 3.3 실무 포인트

* 가능한 조건은 WHERE에 먼저 걸어서 대상 데이터를 줄이는 것이 성능상 유리합니다.
* HAVING에는 꼭 집계 결과에 대한 조건만 두는 습관을 들이시는 것이 좋습니다.

---

## 4. GROUP BY 규칙

### 4.1 개념

* GROUP BY는 특정 컬럼 기준으로 행을 묶고 집계 함수(SUM, COUNT, AVG 등)를 적용하기 위한 구문입니다.
* SELECT에 나오는 컬럼은 집계 함수이거나 GROUP BY에 포함된 컬럼만 허용됩니다.

### 4.2 예시

```sql
-- 고객별 주문 건수, 주문 금액 합계
SELECT  customer_id,
        COUNT(*)    AS order_count,
        SUM(amount) AS total_amount
FROM    orders
GROUP BY customer_id;
```

### 4.3 실무 포인트

* 일부 DB(MySQL 구버전)는 GROUP BY에 없는 컬럼도 SELECT에 허용하지만, 표준이 아니고 버그를 만들기 쉬우므로 피하시는 것이 좋습니다.
* 일자별·고객별·상품별 원장, 정산·매출·재고 집계 쿼리에서 GROUP BY 컬럼 설계가 매우 중요합니다.

---

## 5. DISTINCT 동작 방식

### 5.1 개념

* DISTINCT는 결과 집합에서 중복된 행을 제거합니다.
* DISTINCT는 SELECT 뒤에 나열된 전체 컬럼 조합을 기준으로 중복을 판단합니다.

### 5.2 예시

```sql
-- 고객 수
SELECT DISTINCT customer_id
FROM orders;

-- 고객 + 주문일 기준 조합의 개수
SELECT DISTINCT customer_id, order_date
FROM orders;
```

### 5.3 실무 포인트

* DISTINCT는 내부적으로 정렬 또는 해시 작업이 들어가서 비용이 큽니다.
* DISTINCT로 중복 제거하기 전에, JOIN 구조나 WHERE 조건이 잘못되어 중복이 생긴 것은 아닌지 먼저 확인하시는 것이 좋습니다.

---

## 6. COUNT(*) vs COUNT(col) 차이

### 6.1 개념

* COUNT(*)
  NULL 여부 상관없이 행 수를 셉니다.
* COUNT(col)
  해당 컬럼이 NULL이 아닌 행만 셉니다.

### 6.2 예시

```sql
SELECT COUNT(*)      AS total_users,
       COUNT(email)  AS users_with_email
FROM   customer;
```

### 6.3 실무 포인트

* 단순 행 개수 조회에는 COUNT(*)를 사용하시는 것이 명확합니다.
* 특정 컬럼 채워짐 여부를 세고 싶을 때 COUNT(col)를 사용합니다.
* PostgreSQL, MySQL, Oracle 모두 COUNT(*)는 최적화가 잘 되어 있는 기본 패턴입니다.

---

## 7. NULL 비교 방법

### 7.1 개념

* NULL은 “값이 없음 / 알 수 없음”을 의미하며, 일반 비교 연산자(=, <>)로 비교할 수 없습니다.
* 전용 연산자인 IS NULL, IS NOT NULL을 사용해야 합니다.

### 7.2 예시

```sql
SELECT *
FROM   customer
WHERE  email IS NULL;

SELECT *
FROM   customer
WHERE  email IS NOT NULL;
```

### 7.3 실무 포인트

* NULL 비교를 잘못하면 조건이 전혀 적용되지 않거나, 일부 데이터가 빠져 정합성 문제가 발생할 수 있습니다.
* NULL을 다른 값으로 대체할 때 사용하는 함수는 DB마다 조금 다릅니다.

  * PostgreSQL: COALESCE
  * MySQL: IFNULL, COALESCE
  * Oracle: NVL, COALESCE
* 집계 함수에서 NULL은 보통 “무시”되는 특성이 있습니다.

---

## 8. 계층형 쿼리와 셀프 조인

### 8.1 셀프 조인(Self Join) 개념

* 같은 테이블을 두 번 이상 조인하여 상위–하위 관계를 표현할 때 사용합니다.

```sql
-- 직원과 그 직원의 관리자 이름 조회
SELECT  e.employee_id,
        e.name        AS employee_name,
        m.name        AS manager_name
FROM    employee e
LEFT JOIN employee m
       ON e.manager_id = m.employee_id;
```

### 8.2 계층형 쿼리 개념

* 조직도, 카테고리 트리, 메뉴 트리 등 계층 구조 데이터를 조회할 때 사용합니다.
* DB별 문법

  * PostgreSQL / MySQL 8+: WITH RECURSIVE
  * Oracle: CONNECT BY

```sql
-- PostgreSQL / MySQL 8+ 기준 카테고리 트리 조회
WITH RECURSIVE category_tree AS (
    SELECT id, parent_id, name, 1 AS depth
    FROM   category
    WHERE  parent_id IS NULL

    UNION ALL

    SELECT c.id, c.parent_id, c.name, ct.depth + 1
    FROM   category c
    JOIN   category_tree ct
      ON   c.parent_id = ct.id
)
SELECT *
FROM   category_tree;
```

### 8.3 실무 포인트

* 메뉴·조직·상품 카테고리, 권한 트리 구조에서 자주 사용됩니다.
* 계층형 쿼리가 느릴 경우, depth 컬럼·path 문자열·materialized path·nested set 구조 등을 도입하여 튜닝하기도 합니다.

---

## 9. 윈도우 함수 기본

### 9.1 개념

* 윈도우 함수(Window Function)는 PARTITION BY로 그룹을 나누고, 각 행에 대해 순번, 누적합, 이전/다음 값 등을 계산하는 기능입니다.
* 집계 함수와 달리 행 수를 줄이지 않고, 결과 컬럼을 추가하는 것이 특징입니다.

### 9.2 자주 사용하는 윈도우 함수

* ROW_NUMBER()
  그룹 내에서 1부터 시작하는 순번 부여
* RANK(), DENSE_RANK()
  순위 계산
* SUM() OVER, AVG() OVER
  누적합, 이동 평균
* LAG(), LEAD()
  이전 행, 다음 행의 값을 가져옴
* COUNT() OVER
  파티션별 개수

### 9.3 예시: 결제 히스토리에서 최신 상태만 조회

```sql
WITH last_status AS (
    SELECT  payment_id,
            status,
            created_at,
            ROW_NUMBER() OVER (
                PARTITION BY payment_id
                ORDER BY     created_at DESC
            ) AS rn
    FROM    payment_history
)
SELECT  payment_id,
        status,
        created_at
FROM    last_status
WHERE   rn = 1;
```

### 9.4 실무 포인트

* 각 엔티티의 최신 상태만 조회할 때 굉장히 많이 사용합니다.
* 정산·매출·재고·포인트에서 전일 대비 증감, 누적 합계, 순번 기반 로직 등에 활용합니다.
* PostgreSQL, Oracle은 윈도우 함수 지원이 매우 강력하고, MySQL도 8.0 이상에서 충분히 실무에 활용 가능합니다.

---

## 10. 기본 개념과 함께 알아두면 좋은 항목

아래 개념들은 이 문서에서 깊게 다루지는 않지만, 기본 개념과 함께 정리해 두시면 좋습니다.

* ORDER BY, LIMIT / OFFSET, 페이징 전략
* 서브쿼리 종류 (IN 서브쿼리, 스칼라 서브쿼리, 상관 서브쿼리)
* DDL / DML / DQL / DCL / TCL 구분
* 트랜잭션과 COMMIT / ROLLBACK 동작
* VIEW, MATERIALIZED VIEW 기본 개념

이 부분은 별도 파일(sql-advanced-basic.md 등)로 확장하셔도 좋습니다.

---

## 11. PostgreSQL / MySQL / Oracle 비교

### 11.1 공통점

* 세 DB 모두 표준 SQL을 기반으로 JOIN, GROUP BY, 윈도우 함수(버전 차이는 있음), 인덱스, 트랜잭션 등을 지원합니다.
* 기본 인덱스 구조는 B-Tree이며 MVCC를 기반으로 동시성을 제어합니다.

### 11.2 PostgreSQL

* 오픈소스이며 표준 SQL 지원이 매우 강합니다.
* 윈도우 함수, CTE(WITH, WITH RECURSIVE), JSONB, 배열 타입 등 고급 기능을 잘 지원합니다.
* B-Tree 외에도 GiST, GIN, BRIN 등 다양한 인덱스 타입을 지원합니다.
* SERIAL, BIGSERIAL, SEQUENCE, IDENTITY 컬럼 등 다양한 자동 증가 키 방식을 제공합니다.
* READ COMMITTED, REPEATABLE READ, SERIALIZABLE 격리 수준을 지원하며, 스냅샷 기반 MVCC를 사용합니다.

### 11.3 MySQL (InnoDB 기준)

* 웹 서비스에서 가장 많이 사용하는 오픈소스 DB 중 하나입니다.
* InnoDB 스토리지 엔진이 기본이며 트랜잭션·외래키·MVCC를 지원합니다.
* AUTO_INCREMENT 컬럼으로 자동 증가 키를 관리합니다.
* MySQL 8.0부터 CTE, 윈도우 함수 등 현대적인 SQL 기능을 넓게 지원합니다.
* 테이블당 하나의 클러스터드 인덱스를 가지며, 보통 PK가 클러스터드 인덱스로 사용됩니다.

### 11.4 Oracle

* 상용 DB로, 금융·공공·대형 기간계 시스템에서 많이 사용됩니다.
* PL/SQL, 풍부한 분석 함수, CONNECT BY 기반 계층형 쿼리, 파티셔닝, 병렬 쿼리 등 대규모 데이터 처리 기능이 강력합니다.
* SEQUENCE로 자동 증가 키를 관리하며, 최근 버전에서는 IDENTITY도 지원합니다.
* 트랜잭션, 잠금, 힌트 기능이 매우 풍부하여 세밀한 튜닝이 가능합니다.

### 11.5 간단 차이 요약

* 자동 증가 키

  * PostgreSQL: SERIAL / SEQUENCE / IDENTITY
  * MySQL: AUTO_INCREMENT
  * Oracle: SEQUENCE (추가로 IDENTITY)
* 계층형 쿼리

  * PostgreSQL / MySQL 8+: WITH RECURSIVE
  * Oracle: CONNECT BY
* 윈도우 함수

  * PostgreSQL, Oracle: 오래전부터 지원
  * MySQL: 8.0 이후 본격 지원
* JSON

  * PostgreSQL: JSONB 매우 강력
  * MySQL: JSON 타입 지원
  * Oracle: 버전에 따라 JSON 기능 제공

---

## 12. 마무리 정리

* SQL 기본 개념에서 가장 중요한 것은 JOIN, WHERE vs HAVING, GROUP BY, DISTINCT, COUNT, NULL, 계층형/셀프 조인, 윈도우 함수입니다.
* 위 개념을 PostgreSQL / MySQL / Oracle 중 어떤 DB를 써도 설명할 수 있으면, 면접과 실무 둘 다 안정적으로 대응하실 수 있습니다.
* 이후에는 인덱스, 실행 계획, 트랜잭션, 정합성, 정산/원장 쿼리 등으로 범위를 넓혀가시면 좋습니다.

---

## 13. 주문·재고 시스템 예시 스키마와 자주 쓰는 조회 쿼리

실무에서 자주 나오는 주문·재고 도메인을 가볍게 예시로 정리합니다.
포인트·정산·결제와도 패턴이 비슷해서 연습용으로 좋습니다.

### 13.1 예시 테이블 구조 (간단 버전)

```text
customer         고객
-------------
customer_id      PK
name
grade

product          상품
-------------
product_id       PK
name
category_id
price

orders           주문
-------------
order_id         PK
customer_id      FK -> customer
order_date
status           (CREATED, PAID, CANCELED 등)

order_item       주문 상품
-------------
order_item_id    PK
order_id         FK -> orders
product_id       FK -> product
quantity
unit_price

stock_ledger     재고 원장
-------------
ledger_id        PK
product_id       FK -> product
tx_type          (IN, OUT)
quantity
tx_date
ref_order_item_id   주문 기반 출고인 경우 참조
```

관계 요약

* customer 1 : N orders
* orders 1 : N order_item
* product 1 : N order_item
* product 1 : N stock_ledger

### 13.2 자주 쓰는 조회 쿼리 예시

#### 13.2.1 기간별 고객 주문 요약

기간 내 고객별 주문 건수 및 주문 금액 합계 조회

```sql
SELECT  o.customer_id,
        COUNT(DISTINCT o.order_id)            AS order_count,
        SUM(oi.quantity * oi.unit_price)      AS total_amount
FROM    orders o
JOIN    order_item oi
           ON o.order_id = oi.order_id
WHERE   o.order_date >= '2024-01-01'
  AND   o.order_date <  '2024-02-01'
  AND   o.status = 'PAID'
GROUP BY o.customer_id;
```

포인트

* INNER JOIN, GROUP BY, SUM 집계, DISTINCT order_id 활용

#### 13.2.2 월별 상품별 판매 수량/금액

```sql
SELECT  DATE_TRUNC('month', o.order_date)     AS month,
        oi.product_id,
        SUM(oi.quantity)                      AS total_qty,
        SUM(oi.quantity * oi.unit_price)      AS total_amount
FROM    orders o
JOIN    order_item oi
           ON o.order_id = oi.order_id
WHERE   o.status = 'PAID'
GROUP BY DATE_TRUNC('month', o.order_date),
         oi.product_id;
```

MySQL에서는 DATE_TRUNC 대신 DATE_FORMAT을 사용할 수 있습니다.

포인트

* 월 단위 그룹핑
* 주문 상태 조건 필터링

#### 13.2.3 현재 재고 조회 (입출고 원장 기준)

재고 원장 테이블에서 상품별 재고를 계산하는 예시입니다.

```sql
SELECT  sl.product_id,
        SUM(CASE WHEN sl.tx_type = 'IN'  THEN sl.quantity
                 WHEN sl.tx_type = 'OUT' THEN -sl.quantity
                 ELSE 0 END)              AS current_stock
FROM    stock_ledger sl
GROUP BY sl.product_id;
```

포인트

* SUM(CASE WHEN…) 형태의 조건부 집계
* 재고 원장은 정산·포인트 원장 패턴과 거의 동일합니다.

#### 13.2.4 각 주문의 최신 상태 조회 (윈도우 함수)

결제/주문 상태가 여러 번 바뀔 수 있는 히스토리에서 최신 상태만 조회하는 예시입니다.

```sql
WITH order_status AS (
    SELECT  order_id,
            status,
            changed_at,
            ROW_NUMBER() OVER (
                PARTITION BY order_id
                ORDER BY     changed_at DESC
            ) AS rn
    FROM    order_status_history
)
SELECT order_id,
       status,
       changed_at
FROM   order_status
WHERE  rn = 1;
```

포인트

* ROW_NUMBER() OVER (PARTITION BY) 패턴
* 최신 상태만 가져오는 정산·결제·포인트 공통 패턴

#### 13.2.5 주문이 한 번도 없는 고객 조회 (LEFT JOIN)

```sql
SELECT  c.customer_id,
        c.name
FROM    customer c
LEFT JOIN orders o
       ON c.customer_id = o.customer_id
WHERE   o.order_id IS NULL;
```

포인트

* LEFT JOIN + WHERE o.order_id IS NULL 패턴
* 비활성 고객, 주문 이력 없는 고객 찾기에 자주 사용합니다.

---

## 14. 키워드 요약 표

마지막으로, 이 문서에서 다룬 핵심 키워드를 표로 정리합니다.

| 구분       | 키워드                              | 한줄 요약                            | 관련 기능/예시                      |
| -------- | -------------------------------- | -------------------------------- | ----------------------------- |
| JOIN     | INNER / LEFT / RIGHT JOIN        | 테이블 간 관계를 연결하는 기본 조인 유형          | 고객·주문·주문상품 조회                 |
| 필터링      | WHERE vs HAVING                  | WHERE는 그룹 전 필터, HAVING은 집계 결과 필터 | 기간 필터는 WHERE, 건수 조건은 HAVING   |
| 집계       | GROUP BY 규칙                      | 그룹 기준 컬럼만 SELECT에 집계 없이 올릴 수 있음  | 고객별 매출, 일자별 주문 수              |
| 중복 제거    | DISTINCT                         | 결과 집합에서 중복 행 제거                  | 중복 고객, 중복 조합 제거               |
| 카운트      | COUNT(*) vs COUNT(col)           | 모든 행 vs NULL이 아닌 값만 카운트          | 전체 고객 수, 이메일 등록 고객 수          |
| NULL     | NULL 비교                          | =가 아닌 IS NULL / IS NOT NULL으로 비교 | 누락된 값 조회, COALESCE/IFNULL/NVL |
| 계층형      | 셀프 조인, WITH RECURSIVE            | 동일 테이블 또는 재귀 쿼리로 상·하위 구조 표현      | 조직도, 카테고리 트리                  |
| 윈도우 함수   | ROW_NUMBER, LAG, LEAD, SUM OVER  | 행을 줄이지 않고 순번·누적합·이전값 등을 계산       | 최신 상태 조회, 매출 누적, 전일 대비 증감     |
| DB 비교    | PostgreSQL / MySQL / Oracle      | 공통은 SQL 기본, 구현·기능·문법이 조금씩 다름     | 계층형 쿼리, 자동 증가 키, JSON, 윈도우 함수 |
| 주문·재고 예시 | orders, order_item, stock_ledger | 주문·주문상품·재고 원장 테이블로 도메인 패턴 이해     | 고객별 주문, 월별 매출, 재고 원장 집계       |
