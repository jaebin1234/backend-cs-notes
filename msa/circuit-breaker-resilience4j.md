# circuit-breaker-resilience4j.md

## 개요

서킷 브레이커는 마이크로서비스에서 “외부 호출이 계속 실패할 때, 더 망가지기 전에 호출을 끊어주는 패턴”입니다.
실패가 계속되는 상황에서 무작정 재시도하거나 계속 기다리면, 요청이 쌓이면서 장애가 다른 곳까지 번질 수 있어서 그걸 막는 목적이 큽니다.

---

## 서킷 브레이커가 하는 일

* 호출 실패를 감지한다
* 일정 기준을 넘으면 더 이상 호출하지 않고 빠르게 실패시킨다
* 시간이 지나면 일부 요청만 다시 보내서 복구 여부를 확인한다

상태는 보통 3단계로 움직입니다.

* Closed: 정상 호출을 허용하는 상태
* Open: 호출을 차단하고 바로 실패 처리하는 상태
* Half-Open: 일부 요청만 허용해서 회복됐는지 테스트하는 상태

---

## Resilience4j

Resilience4j는 자바에서 서킷 브레이커를 쓰기 위한 라이브러리입니다.
실패를 감지해서 상태를 바꾸고, 필요하면 fallback(대체 응답)을 실행할 수 있습니다.

주요로 보는 기능은 아래 정도입니다.

* 서킷 브레이커 상태 관리(Closed/Open/Half-Open)
* fallback 메서드 지원
* 상태 변화나 실패율 같은 이벤트를 잡아서 로그로 확인 가능
* 메트릭을 뽑아서 모니터링 도구와 연결 가능

---

## 의존성 추가

starter로 추상화된 의존성을 붙이는 대신, resilience4j를 직접 붙여서 사용합니다.
그리고 어노테이션 기반 동작을 위해 AOP도 같이 추가합니다.

```groovy
dependencies {
    implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.2.0'
    implementation 'org.springframework.boot:spring-boot-starter-aop'
}
```

---

## 설정 파일에서 보는 값들

설정은 application.yml에서 합니다. 핵심은 “최근 호출 결과를 보고 상태를 바꾸는 기준”을 정하는 겁니다.

자주 등장하는 값들을 의미 중심으로 정리하면 이렇습니다.

* slidingWindowType
  판단 기준을 “호출 횟수 기준”으로 볼지, “시간 기준”으로 볼지 선택
* slidingWindowSize
  최근 몇 번(또는 몇 초)의 호출 결과를 보고 판단할지
* minimumNumberOfCalls
  최소 몇 번은 호출이 쌓여야 실패율 판단을 시작할지
* failureRateThreshold
  실패율이 몇 퍼센트 넘으면 Open으로 갈지
* permittedNumberOfCallsInHalfOpenState
  Half-Open에서 테스트로 몇 번만 호출 허용할지
* waitDurationInOpenState
  Open 상태를 유지하는 시간, 지나면 Half-Open으로 이동

---

## Fallback

fallback은 외부 호출이 실패했을 때 “대체 로직”을 실행하는 방식입니다.
정상 응답을 못 주더라도, 최소한의 응답을 줄 수 있게 만들거나 에러 전파를 줄이는 목적입니다.

형태는 이런 식입니다.

```java
@CircuitBreaker(name = "productService", fallbackMethod = "fallbackGetProductDetails")
public Product getProductDetails(String productId) {
    // 외부 호출 or 예외 발생
}

public Product fallbackGetProductDetails(String productId, Throwable t) {
    return new Product(productId, "Fallback Product");
}
```

---

## 이벤트로 상태 확인하기

서킷 브레이커는 상태가 바뀌거나 실패율이 초과되는 순간 같은 이벤트가 발생합니다.
이 이벤트에 리스너를 달면 “지금 상태가 어떻게 변했는지”를 로그로 바로 확인할 수 있습니다.

예를 들면 이런 이벤트를 많이 봅니다.

* 상태 전환 이벤트
* 실패율 초과 이벤트
* 호출 차단 이벤트(Open 상태에서 호출하면 발생)
* 내부 에러 이벤트

---

## 모니터링 연결

actuator와 prometheus를 붙이면 /actuator/prometheus 에서 서킷 브레이커 관련 메트릭을 확인할 수 있습니다.
Prometheus가 이 메트릭을 긁어가고, Grafana에서 대시보드로 보는 흐름입니다.

---

## 실습 요약

구성은 간단하게 “상품 조회 API 하나”로 확인합니다.

* /product/{id} 호출
* id가 111이면 일부러 예외를 발생시켜 실패를 만든다
* 실패가 쌓이면 서킷 브레이커가 Open으로 바뀐다
* Open 상태에서는 실제 메서드를 타지 않고 바로 fallback이 실행되는 걸 확인한다
* /actuator/prometheus에서 서킷 브레이커 메트릭이 나오는 것도 확인한다

---

## 용어 정리

| 용어              | 의미                  |
| --------------- | ------------------- |
| Circuit Breaker | 실패를 감지해 호출을 차단하는 패턴 |
| Closed          | 정상 호출 허용 상태         |
| Open            | 호출 차단 상태(바로 실패 처리)  |
| Half-Open       | 제한적으로 호출 허용해서 복구 확인 |
| Fallback        | 실패 시 대체 로직          |
| Sliding Window  | 최근 호출 결과를 모아두는 범위   |
| Failure Rate    | 실패 비율(임계치 넘으면 Open) |

---

## 공부하면서 떠올릴 질문

* Open 상태에서 waitDuration이 끝나면 바로 Closed로 가는 게 아니라 Half-Open으로 가는 이유는 뭘까?
* minimumNumberOfCalls가 너무 작거나 크면 어떤 문제가 생길까?
* fallback은 “항상 성공 응답”을 주는 게 맞을까, 아니면 에러를 내려야 할 때도 있을까?
