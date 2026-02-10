# SQL 성능 & 실행 계획 정리

이 문서는 SQL 성능을 분석하고 튜닝할 때 꼭 보셔야 할 개념과 실무 패턴을 정리한 것입니다.

구성은 다음과 같습니다.

1. EXPLAIN / EXPLAIN ANALYZE 읽는 법
2. Seq Scan vs Index Scan
3. 조인 방식과 조인 순서
4. JOIN 순서 최적화 예시
5. LIMIT + OFFSET 비효율
6. COUNT(*) 성능 이슈
7. WHERE 절에서 INT vs VARCHAR 비교 비용
8. 중복된 쿼리를 하나의 쿼리로 줄이는 예시
9. 인덱스를 점검해서 병목을 해결한 예시
10. 조건부 서브쿼리를 JOIN으로 변경한 성능 개선 예시
11. 마무리 키워드 정리

DB는 PostgreSQL을 기준으로 설명하지만, MySQL/Oracle에서도 개념은 거의 동일합니다.

---

## 1. EXPLAIN / EXPLAIN ANALYZE 읽는 법

EXPLAIN

* 쿼리를 실제로 실행하지 않고, 옵티마이저가 어떻게 실행할지 계획만 보여줍니다.
* 비용(cost)과 예상 row 수, 어떤 인덱스를 쓸지 등을 확인할 수 있습니다.

EXPLAIN ANALYZE

* 실제로 쿼리를 실행하면서, 계획 + 실제 실행 시간, 실제 row 수를 함께 보여줍니다.
* 성능 튜닝 시에는 EXPLAIN ANALYZE를 기준으로 보는 경우가 많습니다.

예시 (PostgreSQL)

```sql
EXPLAIN ANALYZE
SELECT  o.order_id,
        o.customer_id,
        o.amount
FROM    orders o
WHERE   o.customer_id = 123
  AND   o.created_at >= '2024-01-01'
  AND   o.created_at <  '2024-02-01';
```

대표적인 출력 구조 개념

* Node 타입: Seq Scan, Index Scan, Index Only Scan, Nested Loop, Hash Join 등
* cost: (startup_cost .. total_cost)

  * 단위는 상대적인 비용. 서로 비교용이지 절대 시간은 아닙니다.
* rows: 예상 row 개수
* width: 평균 row 크기(바이트)
* 실제 실행 시에는 actual time, actual rows 등이 함께 나옵니다.

실무에서 볼 때 포인트

* 최상단 노드부터 아래로 내려가면서, 어떤 노드가 시간이 많이 걸리는지 봅니다.
* Seq Scan이 문제인지, Hash Join이 문제인지, Sort가 문제인지 파악합니다.
* 예상 rows와 실제 rows가 크게 다르면 통계가 틀어져 있고, 옵티마이저가 잘못된 계획을 선택했을 가능성이 큽니다.

---

## 2. Seq Scan vs Index Scan

Seq Scan

* 테이블 전체를 처음부터 끝까지 읽으면서 조건을 체크하는 방식입니다.
* 인덱스가 없거나, 있어도 사용하지 않는 경우 나옵니다.
* 데이터가 매우 적은 테이블에서는 Seq Scan이 더 빠를 수 있습니다.

Index Scan

* 인덱스를 통해 필요한 row만 찾아오는 방식입니다.
* 조건이 인덱스 컬럼에 잘 맞게 설계되어 있고, 선택도가 괜찮으면 Index Scan이 나옵니다.

예시

```sql
-- name 인덱스 없음: Seq Scan 가능성 높음
SELECT *
FROM customer
WHERE name = '홍길동';

-- name 인덱스 생성 후: Index Scan 가능
CREATE INDEX idx_customer_name ON customer(name);
```

Seq Scan이 나오는 주요 이유

* 해당 컬럼에 맞는 인덱스가 없다.
* 인덱스가 있어도, 조건이 인덱스를 못 타는 패턴이다.
* 테이블이 너무 작아서 인덱스 타는 것보다 Full Scan이 싸다고 판단한다.
* 통계 상으로 대부분 row가 걸리는 저선택도 조건이라 인덱스 효용이 낮다.

---

## 3. 조인 방식과 조인 순서

대표적인 조인 방식

* Nested Loop Join

  * 바깥 테이블 하나씩 읽으면서, 안쪽 테이블에서 매번 조건에 맞는 row를 찾습니다.
  * 안쪽 테이블 쪽에 인덱스가 있으면 효율적입니다.
  * 작은 테이블과 큰 테이블 조합에서 많이 사용됩니다.
* Hash Join

  * 한쪽 테이블(주로 작은 쪽)을 메모리 해시 테이블로 만들고, 다른 쪽을 스캔하면서 해시로 매칭합니다.
  * 인덱스 없이도 대량 조인을 빠르게 처리할 수 있습니다.
  * 메모리 사용량이 많을 수 있고, 해시 빌드 비용이 있습니다.
* Merge Join

  * 두 테이블이 조인 키 기준으로 이미 정렬되어 있을 때 사용됩니다.
  * 정렬된 상태라면 매우 효율적이지만, 정렬을 따로 해야 한다면 Sort 비용이 들어갑니다.

조인 순서의 의미

* 세 개 이상의 테이블을 조인할 때, 어떤 순서로 묶어서 조인할지가 성능에 큰 영향을 줍니다.
* 필터링 효과가 큰 테이블(조건으로 많이 줄어드는 테이블)을 먼저 조인하는 것이 일반적으로 유리합니다.
* 옵티마이저가 통계에 따라 조인 순서를 자동으로 정하지만, 잘못 선택하는 경우도 있으므로 실행 계획을 보고 판단해야 합니다.

---

## 4. JOIN 순서 최적화 예시

상황

* orders: 1,000,000건
* company: 3,000건
* customer: 500,000건

요구사항

* 특정 회사(company_id)와 특정 기간(created_at) 동안의 주문과 고객 정보를 조회하고 싶습니다.

좋지 않은 패턴

```sql
SELECT  o.order_id, c.name, cp.company_name
FROM    orders o
JOIN    customer c ON o.customer_id = c.customer_id
JOIN    company  cp ON o.company_id  = cp.company_id
WHERE   o.created_at >= '2024-01-01'
  AND   o.created_at <  '2024-02-01'
  AND   cp.company_id = 100;
```

문제

* company 조건이 마지막에 적용되어, 필요 이상으로 많은 주문과 고객을 먼저 붙일 수 있습니다.
* 실행 계획에서 큰 Hash Join 두 번 + 마지막에 Filter가 걸릴 수 있습니다.

더 나은 패턴

```sql
SELECT  o.order_id, c.name, cp.company_name
FROM    company  cp
JOIN    orders   o ON o.company_id  = cp.company_id
JOIN    customer c ON o.customer_id = c.customer_id
WHERE   cp.company_id = 100
  AND   o.created_at >= '2024-01-01'
  AND   o.created_at <  '2024-02-01';
```

또는 company_id와 created_at에 인덱스 추가

```sql
CREATE INDEX idx_orders_company_created
ON orders (company_id, created_at);
```

효과

* cp.company_id = 100 조건을 먼저 적용하고, 바로 해당 회사 orders만 인덱스로 스캔할 수 있습니다.
* customer는 필요한 주문 row 수만큼만 조인하게 되어 전체 I/O가 줄어듭니다.

실무 포인트

* 항상 FROM, JOIN 순서를 바꾼다고 계획이 반드시 바뀌는 것은 아니지만,
  실행 계획을 보면서 작은 테이블, 강한 필터 조건이 있는 테이블을 먼저 조인하는지 확인하시면 좋습니다.
* 인덱스 설계와 조인 순서가 함께 맞아 떨어져야 효과가 큽니다.

---

## 5. LIMIT + OFFSET 비효율 문제

다음과 같은 페이징 쿼리가 있다고 가정합니다.

```sql
SELECT  o.order_id, o.customer_id, o.created_at, o.amount
FROM    orders o
WHERE   o.company_id = 100
ORDER BY o.created_at DESC, o.order_id DESC
LIMIT   20
OFFSET  100000;
```

문제

* OFFSET 100000은 앞의 100000행을 읽고 버린 뒤, 그 다음 20행만 결과로 돌려줍니다.
* 실행 계획에서 Sort 후 많은 rows를 스캔한 뒤, 마지막에 LIMIT을 적용하는 형태가 나올 수 있습니다.
* 페이지가 깊어질수록 성능이 급격히 떨어집니다.

개선 아이디어

* keyset(커서) 페이지네이션으로 변경
  (이전 응답에서 설명한 방식: 마지막 created_at, order_id를 커서로 받아 이후 데이터를 조회)
* 자주 보는 페이지 범위(최근 N페이지)에만 OFFSET을 허용하고,
  더 이전 데이터는 다른 조회 방식을 제공하는 것도 하나의 전략입니다.

실무 포인트

* 실행 계획에서 Rows Removed by Limit 같은 항목이 크면,
  LIMIT 적용 전까지 불필요하게 많이 읽고 버린다는 의미입니다.
* 대규모 데이터에서는 반드시 인덱스를 타는 ORDER BY + LIMIT 패턴을 설계하는 것이 중요합니다.

---

## 6. COUNT(*) 성능 이슈

대형 테이블에서 다음 쿼리는 비용이 큽니다.

```sql
SELECT COUNT(*) FROM orders;
```

문제

* MVCC 기반 DB(PostgreSQL 등)는 삭제/갱신된 튜플 처리 때문에, 단순히 “행 개수만” 따로 저장해 두기 어렵습니다.
* 결국 전체 페이지를 스캔해야 하는 경우가 많습니다.
* WHERE 조건까지 붙으면 거의 Full Scan에 가깝습니다.

개선 아이디어

* 인덱스만 스캔하는 Index Only Scan을 유도
  예: 적당한 인덱스가 있고 visibility map이 잘 유지되면 테이블 페이지를 최소화할 수 있습니다.
* 정산/통계 용도면, 별도의 카운트 집계 테이블을 두고 배치로 갱신하는 전략 사용
  예: 일별/월별 order_count_summary 테이블에 집계값 저장
* 정확한 값이 아니라 대략적인 값이면 통계 보거나, approximate count 기능을 사용하는 방법도 있습니다(DB별 지원 여부 다름).

실무 포인트

* “사용자 수” “전체 주문 수” 같은 숫자를 화면에서 자주 보여줘야 한다면,
  실시간 COUNT(*)가 아니라 사전 집계 값을 사용하는 구조를 고려하셔야 합니다.

---

## 7. WHERE 절에서 INT vs VARCHAR 비교 비용

타입이 다를 때

```sql
-- id 컬럼은 INT, 파라미터는 문자열
SELECT *
FROM   orders
WHERE  id = '123';
```

문제

* DB가 내부적으로 캐스팅을 수행합니다.
* 경우에 따라 인덱스가 있는 id 컬럼에 함수/캐스팅이 들어가는 형태가 되어, 인덱스를 제대로 활용하지 못할 수 있습니다.
* 특히 VARCHAR 컬럼에 숫자 형식이 섞여 있는 상태에서 타입 섞인 비교를 자주 하면, 불필요한 연산 비용이 늘어납니다.

개선

* 컬럼 타입과 파라미터 타입을 맞추는 것이 가장 좋습니다.
* 애플리케이션에서 PreparedStatement의 파라미터 타입을 명확히 지정합니다.

실무 포인트

* 실행 계획에서 Filter 부분에 함수 호출, cast 표현이 보이면 의심해 볼 필요가 있습니다.
* id, amount, created_at 같은 컬럼은 타입을 명확하게 두고, 쿼리도 그에 맞게 작성하는 습관이 중요합니다.

---

## 8. 중복된 쿼리를 하나의 쿼리로 줄이는 예시

문제 상황

* 동일한 조건으로 orders 테이블을 여러 번 조회하는 코드가 있습니다.

예시

```sql
-- 1) 주문 목록 조회
SELECT *
FROM orders
WHERE company_id = 100
  AND created_at >= '2024-01-01'
  AND created_at <  '2024-02-01';

-- 2) 같은 조건으로 총 금액 합계 조회
SELECT SUM(amount)
FROM orders
WHERE company_id = 100
  AND created_at >= '2024-01-01'
  AND created_at <  '2024-02-01';
```

문제

* 동일한 범위를 두 번 스캔합니다.
* 네트워크 왕복도 두 번입니다.

개선 아이디어 1: 한 쿼리에서 함께 가져오기

```sql
SELECT  o.*,
        SUM(o.amount) OVER () AS total_amount
FROM    orders o
WHERE   o.company_id = 100
  AND   o.created_at >= '2024-01-01'
  AND   o.created_at <  '2024-02-01';
```

* 윈도우 함수 OVER()를 활용해 전체 합계를 각 row에 같이 붙여서 가져올 수 있습니다.

개선 아이디어 2: CTE로 한 번만 스캔

```sql
WITH filtered_orders AS (
    SELECT *
    FROM   orders
    WHERE  company_id = 100
      AND  created_at >= '2024-01-01'
      AND  created_at <  '2024-02-01'
)
SELECT
    (SELECT COUNT(*) FROM filtered_orders)       AS order_count,
    (SELECT SUM(amount) FROM filtered_orders)    AS total_amount;
```

* 일부 DB에서는 CTE를 물리적으로 두 번 평가할 수 있지만,
  그래도 애플리케이션에서 두 번 왕복하는 것보다는 한 번에 가져오는 전략이 유리할 때가 많습니다.

실무 포인트

* 동일 조건으로 비슷한 쿼리를 여러 번 날리고 있지 않은지 확인해 보시면 좋습니다.
* 가능하면 한 번의 스캔으로 여러 정보를 같이 가져오도록 합치는 패턴을 고려해 보시는 것이 좋습니다.

---

## 9. 인덱스를 점검해서 병목을 해결한 예시

상황

* 주문 검색 화면에서 다음 쿼리가 느립니다.

```sql
SELECT  o.order_id, o.company_id, o.status, o.created_at, o.amount
FROM    orders o
WHERE   o.company_id = 100
  AND   o.status     = 'PAID'
  AND   o.created_at >= '2024-01-01'
  AND   o.created_at <  '2024-02-01'
ORDER BY o.created_at DESC
LIMIT  50;
```

EXPLAIN ANALYZE 결과

* Seq Scan on orders
* Filter: company_id = 100, status = 'PAID', created_at between ...
* 실행 시간: 수백 ms 이상, 데이터가 많아질수록 더 느려짐

원인

* company_id, status, created_at 조합에 맞는 인덱스가 없습니다.
* created_at 단일 인덱스만 있는 상태거나, 아예 인덱스가 없는 상태일 수 있습니다.

해결

```sql
CREATE INDEX idx_orders_company_status_created_at
ON orders (company_id, status, created_at DESC);
```

다시 EXPLAIN ANALYZE

* Index Scan using idx_orders_company_status_created_at on orders
* 조건에서 company_id, status, created_at 범위를 모두 인덱스로 필터링
* ORDER BY도 인덱스 순서를 그대로 사용
* 실행 시간: 수 ms 수준으로 감소

실무 포인트

* 실행 계획에서 Seq Scan이 보이면, 정말로 어쩔 수 없는 Full Scan인지,
  아니면 필요한 인덱스를 안 만들어서 그런 것인지 구분하시는 게 중요합니다.
* WHERE + ORDER BY 패턴에 맞는 복합 인덱스를 추가하면 체감 성능이 크게 좋아집니다.

---

## 10. 조건부 서브쿼리를 JOIN으로 변경하여 성능 개선 예시

상황

* 포인트 시스템에서, 2024년에 포인트를 많이 사용한 user의 모든 포인트 내역을 보고 싶다고 가정합니다.
* 조건: 2024년 USE 타입 합계가 100000 이상인 user의 전체 point_history.

기존 쿼리 (IN 서브쿼리)

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

문제

* 서브쿼리가 복잡하고, 옵티마이저가 효율적으로 풀지 못하면 불필요한 중복 스캔이 발생할 수 있습니다.
* 실행 계획이 복잡해지고, 어떤 부분이 병목인지 파악하기 어려울 수 있습니다.

개선 쿼리 (CTE + JOIN)

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

설명

* heavy_user CTE에서 조건에 맞는 user 집합을 먼저 계산합니다.

  * 이때 user_id, tx_date, tx_type 인덱스를 잘 타게 설계하면 좋습니다.
* 그다음 전체 point_history에서 heavy_user에 해당하는 user만 JOIN해서 조회합니다.
* 실행 계획도 “heavy_user 생성”과 “heavy_user와 point_history 조인”으로 나뉘어 보여서 이해하기 쉽습니다.

실무 포인트

* 상관 서브쿼리처럼 매 row마다 다시 SELECT를 수행하는 패턴은 성능에 치명적일 수 있습니다.
* 대부분의 조건부 서브쿼리는 CTE/파생 테이블 + JOIN + 윈도우 함수 형태로 재구성할 수 있습니다.
* 실행 계획을 비교할 때, 전체 비용과 실제 실행 시간, 각각의 노드가 소비하는 시간을 함께 보는 게 중요합니다.

---

### CTE 없이 서브쿼리를 JOIN으로 변경한 예시

기존 IN 서브쿼리

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

CTE 없이 JOIN으로 변경

```sql
SELECT  ph.*
FROM    point_history ph
JOIN   (
        SELECT  user_id
        FROM    point_history
        WHERE   tx_date >= '2024-01-01'
          AND   tx_date <  '2025-01-01'
          AND   tx_type = 'USE'
        GROUP BY user_id
        HAVING  SUM(amount) >= 100000
) AS heavy_user
   ON ph.user_id = heavy_user.user_id
WHERE ph.tx_date >= '2024-01-01'
  AND ph.tx_date <  '2025-01-01';
```

설명 포인트 정리용 메모

* `heavy_user` 서브쿼리에서 “조건을 만족하는 user_id 집합”을 먼저 만든 뒤,
  `point_history ph`와 `JOIN`으로 붙여 전체 히스토리를 조회하는 패턴입니다.
* CTE(`WITH heavy_user AS (...)`) 대신, FROM 절에 파생 테이블 `(...) AS heavy_user`를 두어 같은 효과를 냅니다.
* 인덱스는 `point_history(user_id, tx_date, tx_type)` 조합을 고려하면 좋습니다.


---

## 11. 성능·실행 계획 키워드 정리

| 구분         | 키워드                           | 한 줄 요약                                       |
| ---------- | ----------------------------- | -------------------------------------------- |
| 실행 계획 기본   | EXPLAIN / EXPLAIN ANALYZE     | 계획만 vs 실제 실행 포함, 상단 노드와 큰 코스트 위주로 확인         |
| 스캔 방식      | Seq Scan vs Index Scan        | 인덱스 유무, 조건 패턴, 테이블 크기에 따라 선택                 |
| 조인         | Nested Loop, Hash Join, Merge | 작은 쪽 먼저, 해시 메모리, 정렬 여부에 따라 방식 결정             |
| 조인 순서      | 필터링 강한 테이블 우선                 | company, 날짜, 상태 등으로 row를 먼저 줄이는 순서가 유리       |
| 페이징        | LIMIT + OFFSET, keyset        | OFFSET 큰 경우 비효율, 커서 기반 페이징 고려                |
| 카운트        | COUNT(*)                      | 대형 테이블에서는 Full Scan 비용, 집계 테이블·Index Only 활용 |
| 타입 비교      | INT vs VARCHAR                | 타입 불일치로 캐스팅 발생 시 인덱스 사용 저하 위험                |
| 중복 쿼리      | 윈도우 함수, CTE                   | 같은 조건 두 번 스캔하지 말고 한 번에 가져오도록 통합              |
| 인덱스 병목 해결  | Seq Scan → Index Scan         | WHERE + ORDER BY 패턴에 맞춘 복합 인덱스로 큰 개선 가능      |
| 서브쿼리 성능 개선 | 서브쿼리 → CTE + JOIN             | 조건부 서브쿼리는 대부분 CTE/조인으로 재구성 가능                |
