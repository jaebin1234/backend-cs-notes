# SQL 인덱스 정리

이 문서는 인덱스의 기본 개념부터 B-Tree 구조, PK와 인덱스, 클러스터드/논클러스터드, 커버링 인덱스, 인덱스를 못 타는 패턴, ORDER BY + LIMIT, 복합 인덱스 컬럼 순서까지 실무 관점에서 정리한 내용입니다.

---

## 1. 인덱스 기본 개념

* 인덱스는 테이블의 특정 컬럼(또는 컬럼 조합)에 대해 정렬·검색이 빠르게 되도록 만드는 별도의 자료구조입니다.
* 책의 목차/색인처럼, 전체 테이블을 다 뒤지지 않고 필요한 행을 빠르게 찾기 위해 사용합니다.
* 장점: 조회 성능 향상 (특히 WHERE, JOIN, ORDER BY, GROUP BY에 자주 등장하는 컬럼)
* 단점: 인덱스를 유지해야 하기 때문에 INSERT/UPDATE/DELETE 시 쓰기 비용이 증가하고, 디스크/메모리도 더 사용합니다.

요약하면:

* 읽기(SELECT)를 빠르게 하기 위해
* 쓰기(INSERT/UPDATE/DELETE) 비용을 일부 희생하는 구조입니다.

---

## 2. 인덱스가 왜 필요한가

인덱스가 없으면 대부분의 SELECT는 다음처럼 동작합니다.

* 테이블 전체를 처음부터 끝까지 스캔(Full Table Scan)
* 행이 100만 개면 100만 개를 모두 읽어 보면서 조건에 맞는지 확인

인덱스가 있으면:

* 인덱스 트리를 타고, 필요한 키 범위만 빠르게 찾아갑니다.
* 일반적으로 O(N)이 아니라 O(log N)에 가까운 복잡도로 접근할 수 있습니다.

예시

```sql
-- name 컬럼에 인덱스 없음
SELECT *
FROM customer
WHERE name = '홍길동';
```

* 인덱스가 없으면 모든 행을 훑으면서 name = '홍길동'인지 확인합니다.

```sql
-- name 컬럼에 B-Tree 인덱스 생성
CREATE INDEX idx_customer_name ON customer(name);

SELECT *
FROM customer
WHERE name = '홍길동';
```

* 인덱스가 있으면 정렬된 name 인덱스를 빠르게 검색해서 해당 row만 찾아옵니다.

실무 포인트

* 조건에 많이 쓰이는 컬럼, JOIN 조건에 자주 쓰이는 컬럼, 정렬/집계에 자주 쓰이는 컬럼에 인덱스를 두시면 좋습니다.
* 다만 “많이 쓴다고 다 인덱스”를 걸면 쓰기 성능과 스토리지가 크게 나빠지므로 균형이 중요합니다.

---

## 3. PK와 인덱스 – 왜 PK에 인덱스를 다는가

PK(Primary Key)는 테이블에서 행을 유일하게 식별하기 위한 컬럼입니다.

특징

* 값이 중복될 수 없습니다 (UNIQUE).
* NULL 값을 허용하지 않습니다 (NOT NULL).
* 대부분의 DB에서 PK를 만들면 자동으로 인덱스가 생성됩니다.

DB별 동작

* PostgreSQL
  PK를 정의하면 내부적으로 UNIQUE B-Tree 인덱스를 생성합니다.
* MySQL(InnoDB)
  PK가 곧 클러스터드 인덱스입니다. 테이블 데이터 자체가 PK 순서대로 저장됩니다.
* Oracle
  PK 제약조건을 만들면 그 컬럼에 UNIQUE 인덱스를 자동 생성합니다.

왜 PK에 인덱스가 필요한가

* PK는 JOIN, WHERE, UPDATE/DELETE 조건에 자주 사용됩니다.
* 행을 “정확히 하나” 찾는 키이기 때문에 인덱스를 통해 빠르게 찾아가는 게 매우 중요합니다.
* MySQL(InnoDB)은 PK 기반 클러스터드 인덱스를 사용하므로, PK 설계가 곧 테이블의 물리적 정렬 기준이 됩니다.

실무 포인트

* 가능하면 단순하고 짧은 정수형 PK(예: BIGINT AUTO_INCREMENT, SEQUENCE)를 사용하는 것이 좋습니다.
* UUID를 PK로 쓸 경우, 랜덤성 때문에 B-Tree 분할이 잦아져 쓰기 성능이 떨어질 수 있습니다.
  이때는 순차성 있는 UUID 전략을 쓰거나, surrogate key(숫자 PK)를 두고 UUID는 보조 인덱스로 두는 것도 고려합니다.

---

## 4. B-Tree 인덱스 구조

대부분의 RDBMS는 기본 인덱스로 B-Tree(정확히는 B+Tree)를 사용합니다.

구조 개념

* 루트 노드, 중간 노드, 리프 노드로 구성된 트리 구조입니다.
* 리프 노드에는 실제 인덱스 키(예: user_id 값)와 테이블 행을 가리키는 포인터(또는 PK)가 저장됩니다.
* 키는 항상 정렬된 상태로 유지됩니다.

검색 과정

1. 루트 노드에서 찾고자 하는 값의 범위를 좁혀 갑니다.
2. 중간 노드를 타고 내려가며 적절한 리프 노드를 찾습니다.
3. 리프 노드에서 정확한 키를 찾아 테이블 row에 접근합니다.

장점

* 범위 조회(>, <, BETWEEN)에 특히 강합니다.
  예: created_at BETWEEN '2024-01-01' AND '2024-01-31'
* 정렬된 자료구조라 ORDER BY + LIMIT도 인덱스를 잘 타면 빠르게 동작합니다.

실무 포인트

* B-Tree는 “존재 여부 확인”뿐 아니라 “범위 검색”에서 큰 힘을 발휘합니다.
* created_at, amount, score 같은 숫자·날짜 컬럼에 자주 사용됩니다.

---

## 5. 클러스터드 vs 논클러스터드 인덱스

클러스터드 인덱스

* 테이블의 데이터 자체가 인덱스 순서대로 저장되는 구조입니다.
* 테이블당 하나만 존재할 수 있습니다.
* MySQL InnoDB

  * PK가 클러스터드 인덱스입니다.
  * PK 순서대로 데이터 페이지에 저장됩니다.
  * 세컨더리 인덱스(보조 인덱스)는 리프에 PK를 저장했다가, PK로 다시 한 번 찾아가는 구조입니다.

논클러스터드 인덱스

* 인덱스는 별도의 구조로 존재하고, 실제 데이터는 별도 “힙(Heap)” 구조에 저장됩니다.
* PostgreSQL, Oracle은 기본 테이블이 힙 구조이며, 인덱스는 모두 논클러스터드에 가깝습니다.
* 인덱스 리프에는 “테이블 row를 가리키는 포인터(TID)”가 들어 있고, 그 포인터로 원본 row를 찾아갑니다.

실무 포인트

* InnoDB에서는 PK 설계가 특히 중요합니다.
  정렬 기준이면서, 세컨더리 인덱스가 모두 PK를 포함하기 때문입니다.
* PostgreSQL/Oracle은 테이블 저장 순서와 인덱스 순서가 느슨하게 연결되어 있고,
  필요시 CLUSTER 명령이나 파티셔닝으로 물리적 정렬을 어느 정도 맞추기도 합니다.

---

## 6. 커버링 인덱스(Covering Index)

커버링 인덱스란

* 쿼리에 필요한 컬럼이 모두 인덱스에 포함되어 있어서,
  “테이블 데이터 페이지를 따로 읽지 않고 인덱스만으로” 결과를 반환할 수 있는 인덱스를 말합니다.
* 즉, 인덱스만으로 쿼리를 커버할 수 있는 상태입니다.

예시 (MySQL / InnoDB)

```sql
CREATE INDEX idx_orders_cid_date_amount
ON orders (customer_id, order_date, amount);

SELECT customer_id, order_date, amount
FROM   orders
WHERE  customer_id = 123
ORDER BY order_date DESC
LIMIT  10;
```

* 이 쿼리는 SELECT 컬럼과 WHERE, ORDER BY에 쓰는 컬럼이 모두 인덱스에 포함되어 있습니다.
* InnoDB에서는 세컨더리 인덱스 리프에 PK도 포함되므로, 필요한 컬럼을 인덱스에서만 읽고 끝낼 수 있습니다.

PostgreSQL의 Index Only Scan

* PostgreSQL은 커버링 인덱스를 별도 문법으로 만들지는 않지만,
  상황에 따라 Index Only Scan 실행 계획을 사용해 “인덱스만으로” 처리하기도 합니다.
* visibility map 등 조건이 맞아야 완전히 테이블 페이지를 안 읽고 끝낼 수 있습니다.

단점 및 유의사항

* 인덱스에 컬럼을 너무 많이 넣으면 인덱스 크기가 비대해져서 캐시 효율이 떨어지고, 쓰기 비용이 커집니다.
* “정말 자주 쓰는 쿼리 + 꼭 필요한 컬럼”만 커버링 형태로 묶는 것이 좋습니다.

---

## 7. 인덱스를 잘 못 타는(또는 안 타는) 조건들

실무에서 자주 보는 “인덱스가 안 먹는” 패턴입니다.

1. 컬럼에 함수를 적용한 경우

```sql
WHERE DATE(created_at) = '2024-01-01';
```

* created_at에 인덱스가 있어도 대부분 이 조건으로는 인덱스 사용이 어렵습니다.
* 범위 조건으로 바꿔야 합니다.

```sql
WHERE created_at >= '2024-01-01 00:00:00'
  AND created_at <  '2024-01-02 00:00:00';
```

2. 앞에 %가 붙은 LIKE

```sql
WHERE name LIKE '%길동';   -- 인덱스 사용 어려움
WHERE name LIKE '길동%';   -- 인덱스 사용 가능(접두어 검색)
```

3. 타입 캐스팅이 발생하는 비교

```sql
-- created_at이 TIMESTAMP인데 문자열 비교
WHERE created_at = '2024-01-01';

-- id가 INT인데 문자열 비교
WHERE id = '123';
```

* DB가 내부적으로 캐스팅을 수행하면 인덱스를 제대로 못 타거나 비용이 증가합니다.
* 컬럼 타입과 맞는 리터럴/파라미터 타입을 사용하시는 게 좋습니다.

4. 선택도가 낮은 컬럼

```sql
WHERE gender = 'M';    -- 전체 데이터의 상당수가 'M'이라면
```

* 너무 많은 행을 가져와야 하는 조건이면, 옵티마이저가 “그냥 Full Scan이 싸다”고 판단할 수 있습니다.

5. 복잡한 OR 조건

```sql
WHERE status = 'PAID'
   OR status = 'CANCELED'
   OR created_at > '2024-01-01';
```

* 인덱스 사용이 복잡해지고, 실행 계획이 좋지 않게 나올 수 있습니다.
* 경우에 따라 IN, UNION ALL, 조건 분리 등으로 단순화하는 것이 더 좋을 수 있습니다.

6. 아주 작은 테이블

* 데이터가 극단적으로 적은 테이블은 인덱스가 있어도 Full Scan이 비용이 더 적을 수 있습니다.

---

## 8. ORDER BY + LIMIT에서 인덱스를 타는 조건

리스트 화면, 대시보드, 로그 조회 등에서 많이 사용하는 패턴입니다.

자주 쓰는 패턴

```sql
CREATE INDEX idx_orders_cid_created
ON orders (customer_id, created_at DESC);

SELECT  order_id,
        customer_id,
        created_at
FROM    orders
WHERE   customer_id = 123
ORDER BY created_at DESC
LIMIT   50;
```

조건

* WHERE 조건 컬럼과 ORDER BY 컬럼이 인덱스 정의와 같은 순서로 앞쪽에 있어야 합니다.
* 정렬 방향도 인덱스 정의와 맞도록 설계하면 좋습니다.
* LIMIT으로 상위 N개만 필요하므로, 인덱스 스캔 후 바로 멈출 수 있습니다.

잘 못 타는 패턴 예시

```sql
-- 인덱스: (customer_id, created_at)
SELECT  *
FROM    orders
WHERE   created_at >= '2024-01-01'
ORDER BY customer_id, created_at;
```

* WHERE는 created_at만 쓰고, ORDER BY는 customer_id, created_at 순이라 인덱스 활용이 애매해질 수 있습니다.

실무 포인트

* “특정 사용자/회사 + 최신순” 조회를 가장 자주 한다면 그 패턴에 맞춰 인덱스를 설계하시는 게 좋습니다.
* 페이징 성능과 Top-N 조회 성능이 크게 좋아집니다.

---

## 9. 복합 인덱스 컬럼 순서 결정 방법

복합 인덱스는 여러 컬럼을 묶어 하나의 인덱스로 만든 것입니다.

예시

```sql
CREATE INDEX idx_orders_company_status_created
ON orders (company_id, status, created_at);
```

중요한 개념

1. 왼쪽 접두사(left prefix) 규칙

* 위 인덱스로 잘 타기 쉬운 조합

  * company_id
  * company_id + status
  * company_id + status + created_at
* 하지만 status 단독, created_at 단독으로는 이 인덱스를 온전히 활용하기 어렵습니다.

2. WHERE 절에서 자주 쓰이는 “동등 조건(=)” 컬럼을 앞쪽에 두는 것

```sql
WHERE company_id = ?
  AND status = ?
  AND created_at >= ?;
```

* company_id, status: 동등 조건
* created_at: 범위 조건

이런 패턴이면 (company_id, status, created_at) 순서가 자연스럽습니다.

3. ORDER BY와의 조합

```sql
WHERE company_id = ?
  AND status = ?
ORDER BY created_at DESC
LIMIT  50;
```

* 인덱스를 (company_id, status, created_at DESC)로 만들면
  WHERE + ORDER BY + LIMIT를 한 번에 잘 탈 수 있습니다.

4. 선택도(Selectivity)와 조회 패턴의 균형

* 가능한 한 “결과를 많이 줄여주는 컬럼(선택도 높은 컬럼)”을 앞쪽에 두는 것이 좋습니다.
* 하지만 WHERE/ORDER BY 패턴을 완전히 무시하고 선택도만 보고 순서를 정하면, 정작 가장 자주 쓰는 쿼리에서 인덱스를 제대로 못 쓰는 상황이 생길 수 있습니다.
* 결국 “자주 쓰이는 쿼리 패턴 + 선택도”를 함께 보고 설계해야 합니다.

5. 날짜 기반 조회가 매우 많은 경우 고려

예를 들어 이런 패턴들이 많다고 가정해 보겠습니다.

* 매일 새벽 배치: 회사 상관없이 “어제 데이터 전체”를 스캔
  `WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'`
* 대시보드: “특정 날짜/월 전체 매출”
  `WHERE created_at BETWEEN ...`
* 정산 시스템: “월별 전체 거래 집계”

이 경우에는 created_at을 앞에 둔 인덱스가 유리할 수 있습니다.

```sql
-- 날짜 기준 조회가 압도적으로 많을 때 예시
CREATE INDEX idx_orders_created_company_status
ON orders (created_at, company_id, status);
```

장단점

* 장점

  * 날짜 범위로 먼저 크게 줄이고, 그 안에서 company_id, status 조건을 추가로 활용할 수 있습니다.
  * “특정 일자/월 전체 집계”처럼 날짜 중심 배치·통계 쿼리에 유리합니다.
* 단점

  * 반대로 사이트 구조가 “회사별 서비스”라서 대부분 쿼리가
    `WHERE company_id = ? AND status = ? AND created_at BETWEEN ...` 형태라면,
    created_at이 맨 앞에 있는 인덱스보다 (company_id, status, created_at)이 앞에 오는 인덱스가 더 효율적입니다.

실무적으로는

* “회사별 화면, 고객별 화면” 중심이면: (company_id, status, created_at)
* “날짜 기준 전체 집계나 배치가 매우 많음”이면: (created_at, company_id, status)
* 둘 다 중요하면:

  * 인덱스를 2개 두거나
  * 날짜 파티셔닝 테이블 + 회사 기준 인덱스 조합처럼 물리 설계를 분리하는 방식도 고려합니다.

정리하면, 날짜 기반으로 많이 조회된다고 무조건 created_at을 맨 앞에 둘 게 아니라

* 정말 대부분 쿼리가 날짜만으로 큰 범위를 지정하고 보는지
* 아니면 “회사 + 날짜”처럼 회사 키가 항상 먼저 오는지

이 패턴을 보고 결정하시는 게 좋습니다.

6. 인덱스 개수와 비용

* 복합 인덱스를 너무 많이 만들면, 인덱스 개수 폭발로 쓰기 성능이 크게 떨어집니다.
* “가장 자주 쓰는 쿼리 패턴 2~3개”에 맞춰 핵심 복합 인덱스를 만들고, 나머지는 단일 인덱스로 커버하거나 실행 계획을 보고 튜닝하는 식으로 가져가시는 게 좋습니다.

---

## 10. 인덱스 실무 체크리스트 및 키워드 정리

| 구분           | 키워드                       | 한 줄 요약                                         |
| ------------ | ------------------------- | ---------------------------------------------- |
| 필요성          | 인덱스가 왜 필요한가               | Full Scan 대신 B-Tree를 이용해 빠르게 행을 찾기 위한 구조       |
| PK           | PK 인덱스, 클러스터드(InnoDB)     | PK는 행 식별과 JOIN에 필수, InnoDB에서는 PK=클러스터드 인덱스     |
| 자료구조         | B-Tree 인덱스                | 정렬된 트리 구조로, 점/범위 조회, ORDER BY에 강함              |
| 저장 방식        | 클러스터드 vs 논클러스터드           | InnoDB는 PK 기준 정렬, PostgreSQL/Oracle은 힙 + 인덱스   |
| 커버링 인덱스      | Index Only Scan           | 필요한 컬럼이 모두 인덱스에 있어서 테이블을 안 보고 끝내는 패턴           |
| 인덱스를 못 타는 패턴 | 함수, 앞 % LIKE, 캐스팅, 낮은 선택도 | 컬럼에 함수/캐스팅, 잘못된 조건 설계로 인덱스 비활성화 위험             |
| 정렬·페이징       | ORDER BY + LIMIT          | WHERE/ORDER BY 패턴에 맞춘 인덱스로 Top-N/페이징 최적화       |
| 복합 인덱스       | 컬럼 순서, left prefix, 날짜 기반 | = 조건 컬럼을 앞에 두되, 날짜 기반 배치가 많으면 created_at 앞도 고려 |
| 날짜 중심 설계     | created_at 우선 인덱스, 파티셔닝   | “날짜 전체 집계 vs 회사별 조회” 비중을 보고 인덱스/파티셔닝 결정        |

