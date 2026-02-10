# Spring에서 @Async 메소드의 메커니즘

## 핵심 메커니즘

### 프록시 기반 호출 가로채기
- `@EnableAsync`를 활성화하면 스프링이 `@Async` 메서드가 선언된 빈을 프록시(JDK 동적 프록시 또는 CGLIB)로 감쌉니다. 
- 다른 빈에서 이 메서드를 호출하면, 프록시가 호출을 가로채서 동기 실행 대신 비동기 실행으로 넘깁니다.  
  - 자기 자신(this) 안에서 같은 클래스의 `@Async` 메서드를 직접 호출하면 프록시를 거치지 않기 때문에 비동기가 적용되지 않습니다(self-invocation 한계).

### 스레드 풀에 위임  
- 프록시는 내부적으로 `TaskExecutor`에 작업을 제출하고, 별도의 스레드에서 메서드를 실행합니다.  
- 기본 구현은 `SimpleAsyncTaskExecutor`이지만, 보통 `ThreadPoolTaskExecutor`를 빈으로 등록해서 사용합니다.  
- 필요하면 `@Async("poolName")`처럼 풀 이름을 지정해서 여러 풀을 분리해서 사용할 수도 있습니다.

### 리턴 타입 규칙  
- `void` : 호출 측은 결과를 기다리지 않고 그냥 보내는 방식(fire-and-forget)입니다. 예외는 별도 처리 설정이 없으면 로그 정도로만 남습니다.  
- `Future<T>`, `ListenableFuture<T>`, `CompletableFuture<T>` : 호출 측에서 비동기 결과를 나중에 조인하거나 콜백으로 처리할 수 있습니다.

### 컨텍스트 전파  
- `@Async`는 새로운 스레드에서 실행되기 때문에, 기존 요청 스레드의 트랜잭션, 요청 스코프, 보안 컨텍스트 등이 그대로 따라오지 않습니다.  
- `@Transactional`이 붙어 있으면 호출부의 트랜잭션과 분리된 새로운 트랜잭션으로 실행되거나, 트랜잭션이 없는 상태로 실행될 수 있습니다.  
- `SecurityContext` 같은 정보가 필요할 때는 `DelegatingSecurityContextAsyncTaskExecutor`와 같은 전용 executor를 사용해서 컨텍스트를 넘겨주는 방식이 필요합니다.

### 제약 사항  
- 프록시 방식 특성상 private 메서드, final 메서드, final 클래스에는 `@Async`가 적용되지 않습니다.  
- 보통 public 메서드에 붙이고, 다른 빈을 통해 호출하는 구조로 사용하는 것이 안전합니다.

---

## 실무에서 고려할 포인트

### 설계  
- 프록시를 반드시 거치도록, 다른 빈에서 `@Async` 메서드를 호출하는 구조로 설계하시는 것이 좋습니다.  
- 같은 클래스 안에서 내부 메서드 호출로 비동기를 기대하면 적용되지 않습니다.

### 스레드 풀 설정  
- 스레드 풀 크기, 큐 용량, 거부 정책을 상황에 맞게 설정하지 않으면, 풀 고갈로 인해 전체 응답이 지연될 수 있습니다.  
- CPU 바운드 작업과 IO 바운드 작업에 맞게 풀 설정을 분리하는 것도 도움이 됩니다.

### 예외 처리  
- `CompletableFuture`를 사용할 경우 `exceptionally`, `handle` 등을 통해 비동기 예외를 처리할 수 있습니다.  
- `void` 리턴 메서드의 예외는 `AsyncUncaughtExceptionHandler`를 등록해서 공통 처리하는 방법을 사용할 수 있습니다.

### 트랜잭션  
- 비동기 메서드에서 트랜잭션이 필요하다면, `@Async` 메서드 자체에 `@Transactional`을 명시적으로 붙여서 경계를 분리하는 것이 좋습니다.

### 대안 검토  
- 순수하게 무거운 작업을 장시간 처리해야 하는 경우, `@Async` 외에 배치(Scheduler, Spring Batch), 메시지 큐(Kafka, RabbitMQ)나 리액티브(WebFlux)와 같은 구조도 함께 고려하시는 것이 좋습니다.

---

## 용어 정리

| 키워드 | 설명                                                                                       |
|----|------------------------------------------------------------------------------------------|
| @Async | 메서드를 비동기 스레드에서 실행하도록 표시하는 스프링 애노테이션                                                      |
| @EnableAsync | 애플리케이션에서 @Async 기능을 켜는 설정 애노테이션                                                          |
| 프록시(Proxy) | 원본 빈 앞단에 서서 메서드 호출을 가로채고, 부가 공통 기능(@Async, @Transactional 등)을 적용하는 객체                    |
| JDK 동적 프록시 | 인터페이스 기반으로 동적으로 생성되는 프록시 구현 방식                                                           |
| CGLIB 프록시 | 구체 클래스(인터페이스 없음)를 상속해서 만드는 프록시 구현 방식                                                     |
| self-invocation | 같은 클래스 내부에서 this로 자기 메서드를 직접 호출하는 패턴. 프록시를 거치지 않아 @Async, @Transactional 등이 적용되지 않을 수 있음 |
| TaskExecutor | 비동기 작업 실행을 추상화한 스프링 인터페이스. Runnable을 제출해서 실행하는 역할                                        |
| SimpleAsyncTaskExecutor | 요청마다 새 스레드를 만들어 실행하는 기본 Executor 구현. 테스트 용도로는 편하지만 실서비스에는 비추천                            |
| ThreadPoolTaskExecutor | 스레드 풀 기반의 Executor 구현. 코어/최대 스레드 수, 큐 용량, 거부 정책 등을 설정할 수 있음                              |
| Future | 비동기 결과를 나중에 꺼낼 수 있는 자바 표준 인터페이스                                                          |
| ListenableFuture | 콜백 등록이 가능한 스프링 확장 Future                                                                 |
| CompletableFuture | 비동기 파이프라인 구성, 조합, 예외 처리까지 지원하는 자바 8 비동기 API                                              |
| 트랜잭션 전파 | 한 트랜잭션이 다른 트랜잭션 경계와 만날 때, 함께 묶일지, 새로 시작할지 등을 정의하는 규칙                                     |
| @Transactional | 메서드나 클래스에 트랜잭션 경계를 선언하는 스프링 애노테이션                                                        |
| SecurityContext | 스프링 시큐리티에서 현재 인증된 사용자 정보 등을 담고 있는 컨텍스트                                                   |
| DelegatingSecurityContextAsyncTaskExecutor | 비동기 실행 시 SecurityContext를 함께 전달해 주는 Executor 구현                                          |
| AsyncUncaughtExceptionHandler | 리턴 타입이 void인 @Async 메서드에서 발생한 예외를 처리하기 위한 콜백 인터페이스                                       |
| 풀 고갈(Thread pool exhaustion) | 스레드 풀의 스레드와 큐가 모두 꽉 차서 새 작업을 처리하지 못하는 상태                                                 |
| 거부 정책(RejectedExecutionHandler) | 스레드 풀에서 더 이상 작업을 받을 수 없을 때 어떻게 처리할지 정의하는 정책                                              |
| fire-and-forget | 호출 결과를 기다리지 않고 작업만 던져놓는 비동기 호출 패턴                                                        |

