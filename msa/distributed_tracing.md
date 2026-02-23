# distributed_tracing.md

## 개요

분산 추적은 한 번의 요청이 여러 서비스를 거칠 때, 그 흐름을 끝까지 따라가면서 확인하는 방법입니다.
서비스가 많아질수록 “어디서 느려졌는지”, “어디서 에러가 났는지”를 한눈에 보기 어려워지는데, 분산 추적이 그걸 잡아줍니다.

---

## 분산 추적에서 자주 나오는 개념

* Trace
  하나의 요청이 시작해서 끝날 때까지 전체 흐름
  같은 요청이면 trace id가 같습니다.

* Span
  trace 안에 있는 작은 작업 단위
  예를 들면 Order 서비스 처리, Product 서비스 호출 같은 단계가 span이 됩니다.
  span id는 작업 단위를 구분합니다.
  부모-자식 관계로 호출 구조가 잡힙니다.

* Context
  서비스가 바뀌어도 trace id/span id 정보를 계속 들고 가게 만드는 정보 묶음
  이게 전달되니까 “요청 전체 흐름”이 끊기지 않고 이어집니다.

---

## 왜 필요한가

MSA는 요청이 한 서비스에서 끝나지 않고 여러 군데를 지나갑니다.
그러다 문제가 생기면 이런 상황이 생깁니다.

* 호출은 성공했는데 느려짐, 어디가 병목인지 모르겠음
* 일부만 실패했는데 원인이 어디인지 추적이 안 됨
* 로그가 서비스별로 흩어져 있어서 하나의 요청을 묶어서 보기 힘듦

분산 추적은 요청을 기준으로 “서비스 호출 흐름과 시간”을 묶어서 보여줍니다.

---

## Micrometer

Micrometer는 스프링에서 메트릭을 수집하는 라이브러리입니다.
메트릭을 Prometheus, Grafana 같은 도구로 넘겨서 보는 데도 쓰고, 추적(tracing) 쪽도 같이 연결할 수 있습니다.

---

## Zipkin

Zipkin은 trace/span 데이터를 모아서 저장하고, 화면에서 흐름을 보여주는 시스템입니다.
어떤 요청이 어떤 서비스를 거쳤고, 각 단계가 얼마나 걸렸는지 한 번에 확인할 수 있습니다.

---

## Zipkin 실행

Zipkin은 도커로 간단히 띄울 수 있습니다.

```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

대시보드는 여기로 들어갑니다.

* [http://localhost:9411](http://localhost:9411)

---

## 실습 요약

Order 서비스가 Product 서비스를 호출하는 흐름을 Zipkin에서 확인하는 방식입니다.

### 1) Order, Product에 추적 관련 의존성 추가

둘 다 비슷하게 들어갑니다.

* actuator
* micrometer tracing bridge(brave)
* zipkin reporter(brave)
* feign micrometer(Feign 호출도 span으로 잡히게)

### 2) Zipkin endpoint 설정

application.yml에서 Zipkin으로 span을 보낼 주소를 설정합니다.

* endpoint: [http://localhost:9411/api/v2/spans](http://localhost:9411/api/v2/spans)
* sampling.probability: 1.0 (전부 보내도록)

### 3) 실행과 확인

* Eureka 실행
* Order 실행
* Product 실행
* /order/1 호출해서 Order가 Product를 호출하게 만든다
* Zipkin 화면에서 Query를 돌리면 trace가 뜨고, spans 개수가 보인다
* trace를 열어보면 Order가 Product를 호출한 흐름이 단계별로 보인다

---

## 용어 정리

| 용어                  | 의미                               |
| ------------------- | -------------------------------- |
| Distributed Tracing | 요청이 여러 서비스를 거칠 때 흐름을 추적하는 방식     |
| Trace               | 하나의 요청 전체 흐름                     |
| Span                | trace 안의 작업 단위                   |
| Context             | trace/span 정보를 서비스 간에 전달하기 위한 정보 |
| Micrometer          | 스프링 메트릭/트레이싱 수집 라이브러리            |
| Zipkin              | trace/span 수집 및 시각화 시스템          |
| Sampling            | trace를 얼마나 수집할지 비율 설정            |

---

## 공부하면서 떠올릴 질문

* sampling을 1.0으로 두면 항상 좋을까? 어느 순간 부담이 될 수도 있을까?
* Feign 호출이 span으로 잡히는 건 어떤 식으로 연결되는 걸까?
* trace id를 로그에도 같이 남기려면 어떻게 해야 할까?
