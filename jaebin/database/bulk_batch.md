# 벌크(Bulk)와 배치(Batch) 처리 정리

## 1. 한 줄 요약

* 벌크(Bulk): 한 개의 SQL로 여러 행을 한 번에 처리하는 방식.
* 배치(Batch):

  * JDBC 배치: 같은 SQL을 여러 번 실행하는 것을 드라이버가 묶어서 보내는 방식.
  * 배치 처리(Job): 정해진 시간/주기로 대량 데이터를 일괄로 처리하는 작업(Spring Batch 등).

핵심 포인트는 다음 둘이다.

* 벌크 SQL vs JDBC 배치: 전송/실행 방식의 차이
* 벌크/배치 처리 vs 실시간 단건 처리: 처리 시점·트랜잭션 전략의 차이

---

## 2. 용어 정리

### 2.1 벌크(Bulk) 처리란?

* 한 개의 SQL 문장이 여러 행을 한 번에 처리하는 방식.
* DB 엔진 입장에서 set 기반으로 최적화되며, 네트워크 왕복과 파싱/플랜 비용이 한 번만 든다.

예시(SQL)

```
-- 다중 insert
INSERT INTO point_history (company_id, yyyymmdd, amount)
VALUES
  (1, '20250101', 1000),
  (2, '20250101', 2000),
  (3, '20250101', 1500);

-- 업서트(UPSERT) 벌크
INSERT INTO point_settlement (company_id, yyyymm, amount)
VALUES
  (1, '202501', 1000),
  (2, '202501', 2000)
ON CONFLICT (company_id, yyyymm)
DO UPDATE SET amount = EXCLUDED.amount;
```

특징

* SQL 1개로 N행 처리.
* 네트워크 왕복 1회, 파싱·플랜(cache) 1회.
* 실패 시 기본적으로 그 SQL 전체가 롤백됨(부분 성공/부분 실패 제어가 어렵다).

---

### 2.2 JDBC 배치(JDBC Batch)란?

* 동일한 유형의 SQL(INSERT/UPDATE 등)을 여러 번 준비하고, 드라이버가 이를 묶어서 DB로 보내는 방식.
* SQL 템플릿은 동일하고 파라미터만 다른 경우에 사용.

예시(Java, PreparedStatement)

```
PreparedStatement ps = con.prepareStatement(
  "INSERT INTO point_settlement (company_id, yyyymm, amount) " +
  "VALUES (?, ?, ?) " +
  "ON CONFLICT (company_id, yyyymm) DO UPDATE SET amount = EXCLUDED.amount"
);

for (Row r : rows) {
    ps.setLong(1, r.companyId());
    ps.setString(2, r.yyyymm());
    ps.setLong(3, r.amount());
    ps.addBatch();
}

ps.executeBatch(); // 드라이버가 묶어서 전송
```

특징

* SQL N개를 한 번에 보내지만, DB 입장에선 N번 실행하는 것과 가깝다.
* 네트워크 왕복 수를 줄이고, 파싱/바인딩 비용을 최적화.
* 드라이버/DB에 따라 내부적으로 벌크 형태로 재작성될 수도 있다(PostgreSQL: `reWriteBatchedInserts=true` 등).

---

### 2.3 배치 처리(Batch Processing, Job)란?

* 특정 시점/주기(매 10분, 매일 새벽 등)에 대량 데이터를 일괄 처리하는 전체 작업.
* 보통 스케줄러 + 배치 프레임워크(Spring Batch)를 사용한다.
* 내부에서 벌크 SQL, JDBC 배치를 함께 사용할 수 있다.

예시

* 매일 02:00에 포인트 정산 Job 실행.
* 5분마다 주문/결제 집계 Job 실행.
* 어제 데이터 기준으로 통계 테이블 리빌드.

---

## 3. 벌크 vs 배치(JDBC) 차이 정리

### 3.1 벌크 SQL vs JDBC 배치 비교

| 관점       | 벌크 SQL                      | JDBC 배치(JDBC Batch)              |
| -------- | --------------------------- | -------------------------------- |
| 전송 방식    | SQL 1개에 여러 행을 담아서 전송        | 같은 SQL 템플릿 + 서로 다른 파라미터 N개       |
| DB 실행 방식 | 한 SQL이 여러 행을 set 기반으로 처리    | SQL이 N번 실행되는 구조에 가깝다             |
| 네트워크 왕복  | 거의 항상 1회                    | 보통 1회 (배치 크기에 따라)                |
| 장점       | 가장 빠른 경우가 많음, set 최적화       | SQL이 짧고, 부분 실패 결과를 각 행별로 받기 좋음   |
| 단점       | SQL이 길어질수록 파서/메모리 부담, 전체 롤백 | 드라이버/DB 설정에 따라 성능 편차, 설계 필요      |
| 부분 실패 처리 | 어렵고, 보통 전체 롤백               | `executeBatch()` 결과로 일부 실패 파악 가능 |
| 가독성/유지보수 | SQL이 복잡해지기 쉬움               | 애플리케이션 코드 레벨에서 제어하기 쉬움           |

---

### 3.2 벌크/배치 vs 실시간 단건 처리

| 관점    | 실시간 단건 처리            | 벌크/배치 처리                 |
| ----- | -------------------- | ------------------------ |
| 처리 시점 | 요청 즉시                | 모아서 특정 시점에               |
| 트랜잭션  | 짧은 트랜잭션, 작은 데이터      | 긴 트랜잭션, 많은 데이터           |
| 장점    | 사용자 응답 즉시, 문제 범위가 작음 | 처리량 높음, 리소스 효율적          |
| 단점    | 요청 수가 많으면 부하↑        | 장애 시 영향 범위 큼, 롤백 비용 큼    |
| 사용 예시 | 주문 생성/포인트 차감, 로그인    | 정산, 통계 집계, 로그 적재, 마이그레이션 |

---

## 4. 실무 예시 (PostgreSQL + Spring)

### 4.1 포인트 정산 Job 예시 (Spring Batch 관점)

1. Reader

   * JPA/쿼리로 포인트 내역을 기간 단위로 페이지/청크 조회.
   * 예: 회사별, 일자별 합계 값을 스트리밍.

2. Processor

   * 비즈니스 로직으로 정산 금액 계산.
   * DTO 또는 도메인 객체로 변환.

3. Writer

   * 쓰기 전략 선택:

     * 단일 테이블 업서트:

       * PostgreSQL `INSERT ... VALUES (...), ... ON CONFLICT ... DO UPDATE` 벌크 SQL
       * 청크 크기(예: 1000행) 단위로 실행.
     * 여러 테이블에 분산:

       * JDBC 배치로 INSERT/UPDATE를 묶어서 실행.

4. 트랜잭션

   * 청크 단위로 커밋(예: 1000행 단위).
   * 실패 시 해당 청크만 롤백 → 재시도/스킵 전략 설정.

---

### 4.2 MyBatis에서의 벌크/배치

* 벌크:

  * `<foreach>`를 사용해서 `VALUES` 목록을 동적으로 생성.
  * 대량 업서트에 효과적.

* JDBC 배치:

  * `ExecutorType.BATCH` 사용.
  * 동일 SQL 다수 실행 시 네트워크 왕복과 파싱 비용을 줄임.

실전 패턴

* 단일 테이블에 같은 형식의 INSERT/UPSERT → 벌크 SQL + 청크.
* 여러 테이블, 조건 분기 많은 복잡한 로직 → JDBC 배치 또는 일반 단건 처리 + 배치 Job로 나누기.

---

### 4.3 JPA/Hibernate에서의 배치

설정 예시

* `spring.jpa.properties.hibernate.jdbc.batch_size=100`
* `spring.jpa.properties.hibernate.order_inserts=true`
* `spring.jpa.properties.hibernate.order_updates=true`
* PostgreSQL JDBC URL에 `reWriteBatchedInserts=true` 옵션.

패턴

1. 일정 개수(예: 100~1000개)마다 `flush()` + `clear()` 호출.
2. 대량 쓰기에서는 엔티티 라이프사이클/이벤트가 필요 없으면 JPQL 벌크(`UPDATE ... WHERE ...`)도 고려.
3. 단, JPQL 벌크는 1차 캐시와 불일치가 생기므로, 실행 직후 1차 캐시 초기화 필요.

---

## 5. 위험성과 주의사항

### 5.1 벌크 SQL 관련 위험

* SQL 문장이 너무 커질 때

  * Statement 길이 제한에 걸릴 수 있음.
  * 파서/플랜 생성시 메모리 사용량 급증.
  * WAL(Write-Ahead Log) 폭증, 체크포인트 지연.
* 한 SQL 안에서 많은 행을 잠그기 때문에

  * 락을 오래 잡고 있게 되고,
  * 다른 트랜잭션 대기/타임아웃/데드락 가능성이 증가.
* 실패 시 전체 롤백

  * 중간까지 처리된 데이터도 모두 되돌아감.
  * 부분 성공/부분 실패 요구사항에는 적절하지 않음.

### 5.2 JDBC 배치 관련 위험

* 드라이버/DB 설정에 따라 성능이 크게 달라짐

  * PostgreSQL: `reWriteBatchedInserts` 미설정 시, 그냥 순차 실행에 가까울 수 있음.
* 서로 다른 SQL/테이블을 섞어 Batch에 넣으면

  * 드라이버의 최적화가 깨짐.
* 에러 처리

  * `executeBatch()` 결과 배열로 실패 인덱스는 알 수 있지만,
  * 실패 행만 재시도/저장하려면 별도의 로직이 필요.

### 5.3 배치 Job(Spring Batch 등) 자체의 위험

* 긴 트랜잭션

  * MVCC를 사용하는 DB(PostgreSQL 등)에서 오래 열린 트랜잭션은 VACUUM을 막고, 테이블/인덱스 부풀림을 일으킴.
  * 청크 단위 커밋을 강제해야 한다.
* 재실행/중복 실행

  * Job 실패 후 재시작 시, 같은 데이터를 두 번 쓰지 않도록 멱등성 보장 필요.
  * 예: 정산 테이블은 항상 `UPSERT` 사용, 히스토리는 자연키 + `ON CONFLICT DO NOTHING` 등.
* 운영 장애와 영향 범위

  * 실시간 API와 같은 DB를 사용한다면,
  * 배치 Job이 과도하게 리소스를 사용해 실시간 트래픽에 영향을 줄 수 있다(락/IO/CPU).

---

## 6. 언제 무엇을 선택할까? (실무 선택 가이드)

1. 단일 테이블 대량 적재/업서트

   * 데이터 건수: 수만~수십만 이상
   * 요구사항: 최대 성능, 간단한 구조
   * 전략:

     * 벌크 SQL + `ON CONFLICT` 조합 사용.
     * 1000~5000행 정도로 청크 나눠서 실행.
     * 너무 큰 단일 SQL은 피한다.

2. 여러 테이블/조건 분기 많은 도메인 로직

   * 예: 포인트 정산 시 이력 테이블, 정산 테이블, 로그 테이블 등 여러 곳에 동시에 쓰기
   * 전략:

     * JDBC 배치를 통해 각 테이블별 INSERT/UPDATE를 묶어 전송.
     * 필요하다면 Spring Batch로 청크 단위 배치 Job 구성.

3. 비즈니스 규칙/도메인 이벤트 중요한 경우

   * 예: 엔티티 이벤트, 도메인 서비스 호출, 감사 로그 등
   * 전략:

     * JPA 엔티티 중심 로직 + JDBC 배치 조합.
     * 정말 필요한 일부만 벌크 SQL/JPQL로 최적화.

4. 정기 정산·통계·집계

   * 전략:

     * 배치 Job(Spring Batch)으로 스케줄링.
     * Reader에서 읽고, Processor에서 계산, Writer에서 벌크/배치 조합.
     * 청크 사이즈·배치 사이즈를 모니터링 기반으로 튜닝.

---

## 7. 요약

* 벌크

  * SQL 1개로 N행 처리.
  * 최대 성능을 끌어낼 수 있지만, 문장이 너무 커지면 DB에 부담.
  * 실패 시 전체 롤백, 락 홀드 시간 증가.

* JDBC 배치

  * 같은 유형의 SQL N개를 드라이버가 묶어 전송.
  * 네트워크/파싱 절약, 부분 실패 처리 용이.
  * 드라이버/DB 설정에 민감.

* 배치 처리(Job)

  * 일정 주기로 대량 데이터를 일괄 처리하는 전체 작업(보통 Spring Batch).
  * 내부에서 벌크 SQL과 JDBC 배치를 적절히 섞어 사용.

실무에서는 보통 다음 전략을 많이 쓴다.

* 읽기는 JPA/쿼리로 안정적으로 페이징/청크 처리.
* 쓰기는

  * 단일 테이블 업서트 → 벌크 SQL + 청크 커밋.
  * 복잡한 다중 테이블 → JDBC 배치 + 멱등성.
* 모든 대량 처리에는

  * 청크 단위 커밋,
  * 재실행을 고려한 멱등성 설계,
  * 모니터링 기반의 청크/배치 사이즈 튜닝을 함께 가져간다.
