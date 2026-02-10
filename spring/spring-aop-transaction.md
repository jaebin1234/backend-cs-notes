# Spring AOP와 @Transactional 정리

---

## 1. 개요

Spring AOP는 핵심 비즈니스 로직과 공통 관심사(로깅, 트랜잭션, 보안 등)를 분리하기 위한 기술입니다.
스프링은 프록시 기반 AOP를 사용해서, 메서드 호출 전후에 부가 기능을 끼워 넣을 수 있도록 도와줍니다.

특히 `@Transactional` 같은 선언적 트랜잭션은 내부적으로 AOP를 통해 동작합니다.

정리하면:

* 비즈니스 로직은 최대한 순수하게 유지
* 공통 기능은 별도의 모듈(Aspect)로 분리
* 스프링이 프록시를 만들어 메서드 호출 시 공통 기능을 자동으로 적용

---

## 2. AOP 핵심 개념

AOP를 이해할 때 자주 나오는 용어들을 먼저 정리합니다.

### 2.1 AOP 용어 정리

| 용어                     | 한 줄 정의                         | 예시/이미지                                                        |
| ---------------------- | ------------------------------ | ------------------------------------------------------------- |
| Aspect                 | 공통 기능(관심사)을 묶어놓은 모듈            | 로깅, 트랜잭션, 데이터소스 스위칭을 한 클래스에 모아둔 것                             |
| Join Point             | AOP를 걸 수 있는 지점                 | 서비스 메서드 실행 시점, 예: `AuthService.login()` 호출                    |
| Pointcut               | 어떤 Join Point에 적용할지 결정하는 조건/필터 | `execution(* com.example..*Service.*(..))` 처럼 Service 메서드만 선택 |
| Advice                 | 실제로 실행되는 부가 기능 코드              | 메서드 전후에 로그 찍기, 트랜잭션 시작/종료, 데이터소스 변경 로직                        |
| Before Advice          | 대상 메서드 실행 전에 실행되는 Advice       | 컨트롤러 진입 전에 인증/인가 검사                                           |
| After / AfterReturning | 대상 메서드 실행 후에 실행되는 Advice       | 메서드 실행 후 로그 남기기, 리턴값 기록                                       |
| Around Advice          | 대상 메서드를 감싸 전후 모두 제어하는 Advice   | `proceed()` 호출 전 트랜잭션 시작, 호출 후 커밋/롤백 처리                       |
| Target Object          | 실제 비즈니스 로직이 들어 있는 객체           | `OrderService`, `PointService` 같은 서비스 빈                       |
| Proxy                  | Target을 감싸는 대리 객체              | 클라이언트는 Proxy를 부르고, Proxy가 AOP 실행 후 Target 호출                  |
| Weaving                | Aspect를 Target에 결합하는 과정        | 스프링이 런타임에 프록시를 생성해 AOP를 적용하는 것                                |

한 줄 정리:

> AOP는 Aspect(공통 기능)를 Pointcut으로 지정한 Join Point(메서드 실행 지점)에 Advice(@Around, @Before 등) 형태로 끼워 넣고, 스프링이 Proxy를 통해 실제 Target 메서드를 감싸서 실행하는 방식입니다.

---

## 3. Spring AOP 동작 방식

### 3.1 프록시 기반 AOP

스프링은 대부분 다음 방식으로 프록시를 생성합니다.

* 인터페이스가 있으면: JDK Dynamic Proxy
* 인터페이스가 없거나 클래스 기반일 때: CGLIB 프록시

동작 흐름:

1. 스프링이 애플리케이션 구동 시 빈을 스캔
2. AOP 대상이 되는 빈을 찾고, 실제 빈 대신 프록시 객체를 등록
3. 클라이언트가 서비스 빈을 호출하면, 실제로는 프록시 객체가 먼저 호출됨
4. 프록시가 자신의 Pointcut 조건을 확인
5. 매칭되면 등록된 Advice(로깅, 트랜잭션 등)를 실행
6. 필요 시 실제 Target 메서드를 호출하고, 결과를 후처리한 뒤 반환

클라이언트 입장에서는 서비스 메서드를 호출하는 것처럼 보이지만,
중간에 프록시가 부가 기능을 끼워 넣는 구조입니다.

---

## 4. @Transactional 과 Spring AOP

`@Transactional`은 대표적인 AOP 기반 기능입니다.
선언적 트랜잭션 관리의 전체 흐름에는 크게 세 가지 요소가 등장합니다.

* 트랜잭션 매니저 (PlatformTransactionManager 구현체)
* 트랜잭션 AOP 프록시
* 트랜잭션 동기화 매니저 (TransactionSynchronizationManager)

### 4.1 전체 흐름

서비스 메서드에 `@Transactional`이 붙었다고 가정하면, 호출 흐름은 대략 다음과 같습니다.

1. 클라이언트가 `orderService.placeOrder()` 호출
2. 스프링 컨테이너는 실제 `orderService` 대신 트랜잭션 AOP 프록시를 가지고 있음
3. 프록시가 `@Transactional` 메타 데이터를 보고 트랜잭션이 필요한지 판단
4. 필요하다면 트랜잭션 매니저를 통해 트랜잭션 시작 요청
   (예: `DataSourceTransactionManager`, `JpaTransactionManager`)
5. 트랜잭션 매니저가 데이터소스를 통해 커넥션(또는 JPA라면 EntityManager)을 가져와 트랜잭션 시작
6. 트랜잭션 동기화 매니저에 현재 쓰고 있는 커넥션/리소스를 바인딩
   (같은 스레드에서 실행되는 DAO, 리포지토리 코드들이 이 커넥션을 공유)
7. 실제 비즈니스 메서드 호출 (`OrderService` Target 객체)
8. 메서드 실행이 정상 종료되면 트랜잭션 매니저에게 커밋 요청
   (예외가 발생하면 롤백 규칙에 따라 롤백)
9. 트랜잭션 동기화 매니저에서 커넥션/리소스를 정리하고 반환

개발자는 이 복잡한 과정 대신, 서비스 메서드에 `@Transactional` 한 줄만 붙이면 됩니다.

---

## 5. 트랜잭션 매니저 & 동기화 매니저

### 5.1 트랜잭션 매니저(Transaction Manager)

* 스프링이 제공하는 트랜잭션 추상화 인터페이스 `PlatformTransactionManager`의 구현체입니다.
* 주요 구현체:

  * `DataSourceTransactionManager` (JDBC, MyBatis)
  * `JpaTransactionManager` (JPA, Hibernate)
* 역할:

  * 트랜잭션 시작, 커밋, 롤백 담당
  * 데이터소스에서 커넥션을 가져와 트랜잭션을 시작하고, 종료 시 커밋/롤백 후 정리

이 추상화 덕분에 개발자는 JDBC, JPA 등 기술 차이를 의식하지 않고 `@Transactional`만 사용해도 됩니다.

### 5.2 트랜잭션 동기화 매니저(TransactionSynchronizationManager)

* 역할:

  * 한 트랜잭션 안에서 여러 레이어(서비스, 리포지토리 등)가 같은 커넥션/리소스를 공유하도록 도와줌
  * 현재 스레드에 트랜잭션 리소스를 바인딩 (ThreadLocal)
* 이게 없으면:

  * 서비스 메서드에서 얻은 커넥션을 DAO 메서드마다 파라미터로 넘기거나, 전역 변수 형태로 공유해야 하는 불편함 발생
* 요약:

  * 트랜잭션 범위 안에서 커넥션을 “보이지 않는 곳에서” 공유하게 해주는 스프링 내부 컨텍스트 저장소

---

## 6. 코드 예시: @Transactional AOP 흐름

간단한 주문 생성 예시입니다.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;

    public OrderService(OrderRepository orderRepository,
                        PaymentClient paymentClient) {
        this.orderRepository = orderRepository;
        this.paymentClient = paymentClient;
    }

    @Transactional
    public void placeOrder(OrderRequest req) {
        Order order = Order.create(req);
        orderRepository.save(order);

        paymentClient.pay(order); // 외부 결제 연동
        order.complete();

        // JPA라면 트랜잭션 종료 시점에 flush, commit
    }
}
```

내부적으로는 다음 순서로 동작합니다.

* 프록시가 `placeOrder()` 호출을 가로챔
* 트랜잭션 매니저를 통해 트랜잭션 시작
* 트랜잭션 동기화 매니저에 리소스 바인딩
* 실제 Target `OrderService`의 `placeOrder()` 실행
* 예외 여부에 따라 커밋/롤백
* 리소스 정리 및 결과 반환

---

## 7. 실무 기반 AOP 활용 예시

### 7.1 읽기/쓰기 데이터소스 스위칭

실무에서 자주 쓰는 패턴 중 하나는
읽기 전용 쿼리는 슬레이브 DB, 쓰기 작업은 마스터 DB를 사용하도록 스위칭하는 AOP입니다.

흐름 예시:

1. `@ReadOnlyDataSource` 같은 커스텀 어노테이션 정의
2. AOP Aspect에서 이 어노테이션을 Pointcut으로 설정
3. Around Advice에서:

   * 실행 전: 현재 스레드의 데이터소스를 슬레이브로 스위칭
   * `proceed()` 호출로 실제 메서드 실행
   * 실행 후: 데이터소스를 원래대로 복구
4. 기본 트랜잭션(쓰기)은 마스터 DB를 사용

결국 Spring AOP + ThreadLocal + `AbstractRoutingDataSource` 조합으로 구현합니다.

### 7.2 공통 로깅, 모니터링

실무에서 자주 분리하는 공통 기능:

* API 요청/응답 시간 측정
* 특정 계층(Controller/Service) 로그 패턴 통일
* 중요 도메인 메서드 호출에 대한 audit 로그 기록
* 메서드 호출/예외 발생 수를 메트릭으로 전송 (Prometheus, NewRelic 등)

예시:

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.api..*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        try {
            return joinPoint.proceed();
        } finally {
            long end = System.currentTimeMillis();
            String methodName = joinPoint.getSignature().toShortString();
            System.out.println(methodName + " took " + (end - start) + " ms");
        }
    }
}
```

한 번 구현해 두면, 대상 패키지/레이어 전체에 공통적으로 적용할 수 있습니다.

---

## 8. AOP / @Transactional 사용 시 주의사항

실무에서 자주 문제되는 포인트만 정리합니다.

1. 자기 자신 호출(self-invocation)

   * 같은 클래스 안에서 `this.someTransactionalMethod()`로 호출하면 프록시를 거치지 않아 AOP가 적용되지 않습니다.
   * 구조를 나누거나 상위 레이어에서 호출하도록 설계하는 것이 좋습니다.

2. `final` 클래스/메서드

   * CGLIB 프록시는 상속 기반이라 final 클래스/메서드에는 적용이 제한됩니다.

3. 예외에 따른 롤백 규칙

   * 기본: `RuntimeException`, `Error` 계열만 롤백 대상
   * 체크 예외까지 롤백하고 싶다면 `rollbackFor` 옵션 사용
     예: `@Transactional(rollbackFor = Exception.class)`

4. 여러 Aspect의 순서

   * 트랜잭션, 로깅, 보안 등 여러 Aspect가 겹칠 때, 적용 순서에 따라 동작이 달라질 수 있습니다.
   * 필요하면 `@Order` 또는 `Ordered` 인터페이스로 우선순위를 명시합니다.

---

## 9. 마무리 정리

* Spring AOP는 프록시 기반으로 메서드 호출을 가로채고, 공통 기능을 끼워 넣는 기술입니다.
* `@Transactional`은 AOP 위에서 동작하는 선언적 트랜잭션 기능으로,

  * 트랜잭션 매니저
  * 트랜잭션 AOP 프록시
  * 트랜잭션 동기화 매니저
    세 가지가 협력해서 트랜잭션을 시작하고, 커밋/롤백하고, 리소스를 관리합니다.
* 개발자는 서비스 메서드에 어노테이션만 붙여도 안정적인 트랜잭션 처리와 공통 기능 분리를 얻을 수 있습니다.
* 실무에서는 트랜잭션뿐 아니라, 데이터소스 스위칭, 로깅, 모니터링, 보안 등 다양한 공통 기능을 AOP로 분리해 코드 품질과 유지보수성을 높일 수 있습니다.

---

## 10. 키워드 / 용어 정리

| 키워드                               | 설명                                                                           |
| --------------------------------- | ---------------------------------------------------------------------------- |
| AOP                               | 공통 관심사를 분리하기 위한 관점 지향 프로그래밍. 메서드 전후에 부가 기능을 끼워 넣는다.                          |
| Aspect                            | 로깅, 트랜잭션 같은 공통 기능 모듈. 여러 Join Point에 적용될 수 있다.                               |
| Join Point                        | AOP가 적용될 수 있는 지점. 스프링에서는 주로 메서드 실행 시점을 의미한다.                                 |
| Pointcut                          | 어떤 Join Point를 대상으로 할지 정의하는 조건식이다.                                           |
| Advice                            | 실제 실행되는 부가 기능 코드. Before, After, Around 등이 있다.                               |
| Around Advice                     | 메서드를 감싸 전후를 모두 제어하는 Advice. `proceed()`로 실제 메서드를 호출한다.                       |
| Proxy                             | Target 객체를 감싸는 대리 객체. 클라이언트는 Proxy를 호출하고, Proxy가 AOP를 수행한 후 Target을 호출한다.    |
| Target                            | 실제 비즈니스 로직이 들어 있는 객체. 서비스 빈 등이 여기에 해당한다.                                     |
| Weaving                           | Aspect를 Target에 결합하는 과정. 스프링은 주로 런타임에 프록시를 만들어 위빙한다.                         |
| PlatformTransactionManager        | 스프링의 트랜잭션 추상화 인터페이스. 구현체가 실제 트랜잭션 시작/커밋/롤백을 처리한다.                            |
| DataSourceTransactionManager      | JDBC, MyBatis 등 DataSource 기반 트랜잭션을 처리하는 트랜잭션 매니저.                           |
| JpaTransactionManager             | JPA, Hibernate 환경에서 트랜잭션을 처리하는 트랜잭션 매니저.                                     |
| TransactionSynchronizationManager | 현재 스레드와 트랜잭션 리소스를 바인딩하는 스프링 내부 도우미. 같은 트랜잭션 안에서 커넥션/EntityManager를 공유하게 해준다. |
| 선언적 트랜잭션 관리                       | 코드로 begin/commit을 직접 호출하지 않고, 어노테이션 설정으로 트랜잭션을 관리하는 방식.                      |
| JDK Dynamic Proxy                 | 인터페이스 기반 프록시 생성 방식. 인터페이스가 있을 때 사용된다.                                        |
| CGLIB Proxy                       | 클래스 상속 기반 프록시 생성 방식. 인터페이스 없이 클래스만 있을 때 사용된다.                                |

---

