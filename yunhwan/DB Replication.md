## DB Replication 정리 (MySQL 기준)
- DB Replication은 데이터베이스의 고가용성과 안정성을 보장하기 위해 원본(Source) 서버의 데이터를 복제(Replica) 서버로 동기화하는 기술입니다. 특히 대규모 환경에서는 장애 발생 시 서비스 연속성과 신뢰성을 확보하기 위해 필수적으로 사용됩니다.

### Binary Log 기반 방식
복제의 핵심은 Source 서버에서 생성되는 Binary log이며, MySQL은 이를 세 가지 방식으로 기록
- **Row 기반**
  - 행 단위로 변경 내용을 기록
  - 데이터 일관성이 높음
  - 단점: 로그 크기가 커져 저장 공간 부담이 있음
- **Statement 기반**
  - 실행된 SQL 문을 그대로 기록
  - 로그 크기가 작아 효율적
  - 단점: `NOW()`, `RAND()`와 같은 비결정적 쿼리 실행 시 Replica와 결과 불일치 발생 가능
- **Mixed 기반**
  - 두 방식을 혼합
  - 일반 쿼리는 Statement 방식으로 효율성 유지
  - 비결정적 쿼리는 Row 방식으로 일관성을 확보
  - 장점: 두 방식의 장점을 취합
  - 단점: 구현이 다소 복잡

### 복제 과정
1. Source 서버에서 데이터 변경 발생 -> Binary log 기록
2. Replica 서버의 IO Thread가 Binary log를 읽어 **Relay log**에 저장
3. SQL Thread가 Relay log를 기반으로 실제 데이터 반영
4. 이 과정은 최적화되어 일반적으로 100ms 이내 동기화가 이루어짐 -> 실시간에 가까운 일관성 유지

---

## 결론
Row 방식은 정확성, Statement 방식은 효율성, Mixed 방식은 균형을 갖추고 있으며, MySQL의 Replication은 Binary log -> Relay log -> SQL Thread 적용 과정으로 빠른 데이터 동기화를 보장