# applicationcontext-vs-beanfactory-basic.md

## 개요
Spring IoC 컨테이너라고 할 때, 실제로는 BeanFactory와 ApplicationContext라는 “두 이름”이 자주 나옵니다.
정리하면 BeanFactory는 컨테이너의 최소 기능이고, ApplicationContext는 실무에서 쓰는 확장형 컨테이너입니다.

## 정의
- BeanFactory
  - 스프링 컨테이너의 가장 기본 인터페이스입니다.
  - 핵심은 빈을 “생성하고”, “찾아주고(getBean)”, “의존성 주입까지 포함한 관리”를 하는 것입니다.
- ApplicationContext
  - BeanFactory를 포함(상속)하는 상위 컨테이너입니다.
  - 실무에서 필요한 기능(이벤트, 메시지, 리소스 로딩 등)을 더 많이 제공합니다.

## 핵심 개념
- 관계
  - ApplicationContext는 BeanFactory의 기능을 기본으로 포함합니다.
  - 즉, ApplicationContext를 쓰면 BeanFactory 기능도 같이 쓰는 셈입니다.
- 관점
  - BeanFactory는 “빈 공장”
  - ApplicationContext는 “애플리케이션 운영 환경까지 포함한 컨테이너”
- 왜 구분하나
  - 스프링이 커지면서 “빈 생성/관리” 외에 애플리케이션 레벨 기능이 많이 필요해졌기 때문입니다.

## 동작 방식
- 공통: 둘 다 빈을 생성하고, 보관하고, 주입합니다.
- 차이: ApplicationContext는 빈 관리 외 기능이 추가됩니다.

ApplicationContext가 자주 제공하는 추가 기능 예시
- 국제화 메시지 처리(메시지 소스)
- 애플리케이션 이벤트 발행/구독(이벤트 기반)
- 리소스 로딩(파일, 클래스패스 리소스 등)
- 환경/프로퍼티 관리(환경 변수, 설정 값)

## 전체 흐름
1) 애플리케이션 시작 시, 스프링 부트는 ApplicationContext를 만듭니다.
2) ApplicationContext는 내부적으로 빈 등록 목록을 만들고 빈을 생성/주입합니다.
3) 이후 런타임에서 getBean을 통해 빈을 꺼내 쓰거나, 프레임워크가 필요한 빈을 찾아 연결합니다.
4) 동시에 이벤트, 메시지, 리소스 같은 부가 기능도 ApplicationContext가 담당합니다.

## 실무 포인트
- 대부분의 스프링 부트 프로젝트는 ApplicationContext를 사용합니다.
  - 우리가 직접 BeanFactory를 고르는 경우는 거의 없습니다.
- 면접에서 이렇게 말하면 충분합니다.
  - BeanFactory는 가장 기본 컨테이너이고, ApplicationContext는 실무 기능이 추가된 상위 컨테이너입니다.
- 진짜 실무에서 중요한 포인트는 이거 하나입니다.
  - “내가 쓰는 스프링 컨테이너는 보통 ApplicationContext다”
  - “그래서 빈 관리 외 기능도 함께 제공된다”

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| BeanFactory | 스프링 컨테이너의 최소 기능(빈 생성/조회/관리) |
| ApplicationContext | BeanFactory 확장, 실무 기능(이벤트/메시지/리소스 등) 포함 |
| getBean | 컨테이너에서 빈을 조회하는 대표 메서드 |
| Context | 애플리케이션 실행 환경을 포함하는 상위 개념 |

## 질문
- 스프링 부트에서 기본 컨테이너는 BeanFactory일까요 ApplicationContext일까요?
- ApplicationContext가 제공하는 “빈 관리 외 기능” 3가지를 말해볼 수 있나요?
- 이벤트 발행/구독을 컨테이너가 지원하면 어떤 점이 편해질까요?
- BeanFactory만으로도 DI는 가능한데, 왜 대부분 ApplicationContext를 쓸까요?
