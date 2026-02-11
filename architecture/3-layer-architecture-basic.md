# 3-layer-architecture-basic

## 개요
처음에는 Controller 하나로 모든 API를 처리해도 동작은 합니다.
하지만 기능이 늘어나면 한 클래스에 코드가 너무 많아져서 이해하기 어렵고, 수정/추가 시 영향 범위를 예측하기 힘들어집니다.
이때 문제의 핵심은 “유지 보수 비용 증가”인데, 보통 결합도(커플링)와 종속성(의존성) 구조가 나빠질수록 유지 보수가 어려워집니다.

그래서 서버 개발에서는 처리 흐름이 반복된다는 점을 이용해 Controller, Service, Repository로 역할을 분리합니다.

## 정의
- 3 Layer Architecture: 서버의 처리 과정을 Controller, Service, Repository 3개 계층으로 나눠 역할을 분리하는 구조
- 결합도(Coupling): 한 코드가 다른 코드의 내부 구현에 얼마나 강하게 묶여 있는지의 정도
- 종속성(Dependency): A가 B를 사용해야만 동작하는 관계, 보통 “A가 B에 의존한다”라고 말함
- Controller: 요청을 받고, 응답을 반환하는 계층
- Service: 비즈니스 로직을 처리하는 계층
- Repository: DB 접근과 CRUD를 담당하는 계층

## 핵심 개념
### 1) 유지 보수가 어려워지는 이유(결합도, 종속성 관점)
Controller에 모든 로직이 몰리면, 다음 문제가 같이 생깁니다.

- 결합도가 높아짐
  - HTTP 요청 파싱, 검증, 비즈니스 로직, DB 접근, 응답 포맷이 한 곳에 섞입니다.
  - 한 부분을 바꾸면 다른 부분까지 같이 깨질 가능성이 커집니다.

- 종속성이 복잡해짐
  - Controller가 DB 접근 코드까지 직접 들고 있으면,
    “Controller가 DB 구현 상세에 의존”하는 형태가 됩니다.
  - DB 구조, SQL, 연결 방식이 바뀌면 Controller 코드도 같이 바뀌게 됩니다.

- 변경 영향 범위가 커짐
  - 작은 기능 수정이 “Controller 전체 수정”으로 커지기 쉽습니다.
  - 테스트 범위도 커지고, 실수 확률도 증가합니다.

정리하면, “유지 보수 어렵다”는 말의 실체는
- 결합도 증가로 인한 변경 전파
- 종속성 증가로 인한 수정 범위 확대
이 두 가지가 커지는 것입니다.

### 2) 3계층 분리는 결합도를 낮추고 종속성을 단순화한다
- Controller는 HTTP 관련 관심사만 남깁니다.
- Service는 비즈니스 로직만 집중합니다.
- Repository는 DB 접근을 캡슐화합니다.

이렇게 나누면
- 결합도가 낮아집니다.
  - HTTP 로직 변경이 DB 로직을 건드릴 필요가 줄어듭니다.
  - DB 접근 방식 변경이 컨트롤러까지 올라오는 일이 줄어듭니다.
- 종속성이 단순해집니다.
  - Controller는 Service에만 의존
  - Service는 Repository에 의존
  - 위로 올라가거나 옆으로 새는 의존을 줄이게 됩니다.

## 각 계층의 책임
- Controller
  - 클라이언트 요청을 받습니다.
  - 요청 데이터를 정리해서 Service에 전달합니다.
  - Service 결과를 응답 포맷으로 만들어 클라이언트에 반환합니다.

- Service
  - 사용자의 요구사항을 처리하는 비즈니스 로직 담당입니다.
  - DB 저장/조회가 필요하면 Repository에 요청합니다.

- Repository
  - DB 관리(연결, 해제, 자원 관리)를 합니다.
  - DB CRUD 작업을 처리합니다.

## 전체 흐름
1) Client가 요청을 보냅니다.
2) Controller가 요청을 받고, 필요한 데이터를 Service에 전달합니다.
3) Service가 비즈니스 로직을 수행합니다.
4) DB 작업이 필요하면 Service가 Repository를 호출합니다.
5) Repository가 DB에 CRUD를 수행합니다.
6) Service가 결과를 정리해 Controller에 반환합니다.
7) Controller가 클라이언트에게 응답합니다.

## 실무 포인트
- Controller는 얇게 유지하는 편이 좋습니다.
  - 요청 파싱, 검증, 라우팅, 응답 반환 정도
- Service는 비대해지기 쉬워서 기능 단위로 분리하는 기준이 필요합니다.
- Repository가 DB 접근을 캡슐화하면,
  DB 관련 변경이 Service/Controller로 전파되는 것을 줄일 수 있습니다.
- 종속성 방향을 단순하게 유지하는 게 중요합니다.
  - Controller -> Service -> Repository 방향으로 흐르게 유지
  - Controller가 Repository를 직접 호출하는 패턴은 피하는 편이 관리에 유리합니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| 결합도 | 한 코드가 다른 코드 내부 구현에 얼마나 묶여 있는지 |
| 종속성 | A가 B를 사용해야만 동작하는 의존 관계 |
| 3 Layer Architecture | Controller, Service, Repository로 역할 분리 |
| Controller | 요청/응답 담당 |
| Service | 비즈니스 로직 담당 |
| Repository | DB 접근/CRUD 담당 |

## 공부하면서 떠올린 질문
- Controller가 Repository를 직접 호출하면 결합도/종속성 관점에서 뭐가 나빠질까?
- DB 컬럼이 변경되면 어떤 계층까지 영향이 가는 게 “이상적”일까?
- 유지 보수가 쉬운 구조를 만들기 위한 의존 방향 규칙은 무엇일까?
