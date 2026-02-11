# strong-coupling-problems-3layer.md

## 개요
3-Layer(Controller, Service, Repository)에서 상위 계층이 하위 계층을 직접 new로 생성하면 강한 결합이 됩니다.
강한 결합은 변경이 연쇄로 퍼지고, 테스트가 어려워지고, 코드 재사용이 깨지는 문제가 생깁니다.
이 문서는 강한 결합의 문제를 흐름(플로우)으로 이해하는 데 집중합니다.

## 정의
- 강한 결합
  - Controller가 Service 구현체를 직접 생성하고
  - Service가 Repository 구현체를 직접 생성하는 구조
- 느슨한 결합
  - Controller는 Service의 역할(인터페이스)에만 의존하고
  - Service는 Repository의 역할(인터페이스)에만 의존하며
  - 실제 객체 생성과 연결은 외부에서 조립(주입)하는 구조(DI)

## 핵심 개념
- 변경 전파(연쇄 수정)
  - Repository 생성자나 내부 구현이 바뀌면 Service가 바뀌고
  - Service 생성 방식이 바뀌면 Controller가 바뀌고
  - Controller가 여러 개면 수정 범위가 폭발합니다.
- 중복 생성
  - 같은 Service를 여러 Controller가 각각 new로 만들면 인스턴스가 여러 개 생깁니다.
  - 동일 설정(예: DB 연결 정보)을 여러 곳에서 반복하게 됩니다.
- 테스트 어려움
  - Controller 테스트에서 Service를 교체하기 어렵고
  - Service 테스트에서 Repository를 가짜로 바꾸기 어렵습니다.
  - 결과적으로 단위 테스트가 힘들어지고 통합 테스트 의존도가 커집니다.

## 동작 방식

### 강한 결합 구조(문제 구조) 플로우차트
[요청 처리 흐름]
Client
  -> Controller
      -> Service
          -> Repository
              -> DB

[객체 생성/연결 흐름]
Controller 내부에서 Service를 new로 생성
  -> Service 내부에서 Repository를 new로 생성

[변경 발생 예시]
Repository 생성자에 파라미터가 추가됨
  -> Service의 new 코드 수정 필요
      -> Controller의 new 코드 수정 필요
          -> Controller가 여러 개면 전부 수정 필요

핵심은 "하위 변경이 상위 코드 수정으로 강제된다" 입니다.

### 느슨한 결합 구조(DI 적용) 플로우차트
[요청 처리 흐름]
Client
  -> Controller
      -> Service
          -> Repository
              -> DB

[객체 생성/연결 흐름]
외부(조립자/컨테이너)가 Repository 생성
  -> Repository를 Service에 주입
      -> Service를 Controller에 주입

[변경 발생 시]
Repository 생성자 변경
  -> 외부 조립 코드만 수정
  -> Controller, Service 코드는 수정 최소화

핵심은 "생성과 사용을 분리해서 변경 영향 범위를 줄인다" 입니다.

## 전체 흐름
1) 강한 결합은 상위 계층이 하위 계층 생성까지 책임지는 구조입니다.
2) 하위 계층 변경이 상위 계층 코드 수정으로 이어집니다.
3) DI를 적용하면 의존 객체를 외부에서 만들어 주입받습니다.
4) 결과적으로 변경 전파가 줄고 테스트 대체가 쉬워집니다.

## 실무 포인트
- Controller에서 Service를 new로 만들지 않습니다.
  - 요청/응답 처리와 라우팅에 집중합니다.
- Service에서 Repository를 new로 만들지 않습니다.
  - 비즈니스 흐름과 트랜잭션 경계에 집중합니다.
- 의존은 구현체가 아니라 역할에 둡니다.
  - 교체 가능성이 생기고 변경 비용이 내려갑니다.
- "객체 생성은 한 번, 재사용" 방향으로 구조를 잡습니다.
  - 같은 기능을 여러 Controller가 써도 서비스 인스턴스를 중복 생성하지 않게 합니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| 강한 결합 | 상위가 하위를 직접 생성하고 구현에 묶이는 구조 |
| 느슨한 결합 | 상위는 역할에만 의존하고 구현 연결은 외부에서 하는 구조 |
| 변경 전파 | 하위 변경이 상위 코드 수정으로 연쇄 확산되는 현상 |
| DI | 의존 객체를 외부에서 만들어 주입하는 방식 |
| IoC | 객체 생성/연결의 제어권을 외부로 넘기는 설계 원칙 |

## 질문
- Repository 생성자가 바뀌면 지금 구조에서 어디까지 수정이 퍼지나요?
- Controller가 Service를 직접 생성하면 테스트에서 어떤 점이 가장 먼저 막히나요?
- Service가 Repository를 직접 생성하면 어떤 대체(가짜) 전략이 어려워지나요?
- "생성과 사용 분리"를 한 문장으로 설명하면 어떻게 말할 수 있나요?
