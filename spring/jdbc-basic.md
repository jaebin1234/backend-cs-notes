# jdbc-basic

## 개요
애플리케이션 서버는 요청을 처리하려면 DB와 소통해야 합니다.
이때 자바에서 DB에 연결하고(SQL 보내고, 결과를 받는) 표준 방법이 JDBC입니다.

## 정의
- JDBC(Java Database Connectivity): 자바에서 DB에 접근할 수 있게 해주는 표준 API
- JDBC 드라이버: 각 DB 회사가 JDBC 표준 인터페이스를 자기 DB에 맞게 구현해서 제공하는 라이브러리
- Connection: DB 연결(커넥션)
- Statement/PreparedStatement: SQL 실행을 위한 객체(실무에서는 PreparedStatement 개념이 중요)
- ResultSet: SELECT 결과(조회 결과 집합)
- JdbcTemplate: JDBC 사용 시 반복되는 작업(연결, 실행, 종료 등)을 줄여주는 스프링 도구

## JDBC가 왜 필요했나
### 문제 상황
- 애플리케이션이 DB에 접근하려면 보통 아래 작업이 필요합니다.
  1) DB 커넥션 연결
  2) SQL 작성 후 전달
  3) 결과 응답 받기
- 그런데 MySQL에서 PostgreSQL로 바꾸면,
  연결 방식/전달 방식/응답 처리 방식이 달라져서 애플리케이션 코드도 크게 수정될 수 있습니다.

### 해결
- DB가 바뀌어도 애플리케이션 코드는 “표준 인터페이스(JDBC)”에만 의존하게 만들자.
- DB별 차이는 “드라이버 교체”로 해결하자.
- 즉, 애플리케이션은 JDBC 표준을 쓰고, DB별 구현은 드라이버가 담당합니다.

## JDBC 구성(큰 그림)
- 애플리케이션
  - JDBC 표준 인터페이스를 사용(Connection, Statement, ResultSet 등)
- DB 벤더(예: MySQL, PostgreSQL)
  - JDBC 표준을 구현한 드라이버를 제공
- 그래서 DB를 바꿀 때는 “드라이버만 교체”하면 되는 구조가 됩니다.

## JdbcTemplate이 왜 나왔나
- JDBC 덕분에 DB 교체는 쉬워졌지만, 여전히 불편한 점이 남습니다.
  - 매번 커넥션 열고 닫기
  - Statement 준비 및 실행
  - 예외 처리
  - ResultSet 처리
- 이런 반복/중복 작업을 줄이려고 스프링이 JdbcTemplate을 제공합니다.

## JdbcTemplate이 해주는 일(개념)
- 반복되는 JDBC 작업을 템플릿화해서 대신 처리합니다.
  - 커넥션 획득/반납
  - SQL 실행 흐름
  - 예외를 스프링 방식으로 변환
- 개발자는 “SQL과 파라미터”, “조회 결과 매핑” 같은 핵심에 집중할 수 있습니다.

## SELECT 결과 매핑(RowMapper) 개념
- SELECT는 결과가 여러 행으로 올 수 있습니다.
- 그래서 “한 행을 자바 객체로 바꾸는 규칙”이 필요합니다.
- RowMapper는 결과의 한 행을 DTO 같은 객체로 변환하는 역할을 합니다.

## 실무 포인트
- JdbcTemplate은 JDBC의 불편함을 많이 줄이지만, 여전히 SQL과 매핑 작업이 남습니다.
- 그래서 실무에서는 보통 ORM(JPA 등)을 더 많이 사용하고,
  JdbcTemplate은 “단순 쿼리, 배치성 작업, 성능 민감 구간”에서 선택적으로 쓰는 경우가 많습니다.
- “DB 커넥션은 비용이 비싸다”가 기본 전제라서, 실무에서는 커넥션 풀(DataSource)을 같이 이해해야 합니다.
  - 이건 보통 다음 단계에서 자연스럽게 나옵니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| JDBC | 자바의 DB 접근 표준 API |
| JDBC 드라이버 | DB 벤더가 제공하는 JDBC 구현체 |
| Connection | DB 연결 |
| Statement/PreparedStatement | SQL 실행 객체 |
| ResultSet | 조회 결과 |
| JdbcTemplate | JDBC 반복 작업을 줄이는 스프링 도구 |
| RowMapper | ResultSet 한 행을 객체로 변환하는 규칙 |

## 공부하면서 떠올린 질문
- “JDBC 표준”이 없다면 DB 교체 비용이 왜 커질까?
- JdbcTemplate이 대신 해주는 “반복 작업”은 정확히 어떤 것들일까?
- SELECT에서 RowMapper가 없으면 어떤 불편함이 생길까?
- JdbcTemplate과 ORM(JPA)은 어떤 기준으로 선택할까?
