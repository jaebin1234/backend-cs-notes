# controller-basic

## 개요
Controller는 HTTP 요청을 받아 “어떤 처리를 할지” 결정하고, 그 결과를 응답으로 돌려주는 계층입니다.
Spring MVC에서는 Front Controller(DispatcherServlet)가 요청을 먼저 받고, 알맞은 Controller 메서드로 연결해줍니다.

## 정의
- Controller: 요청을 처리할 진입점, 보통 서비스 호출과 응답 반환을 담당
- @Controller: 해당 클래스를 Spring MVC 컨트롤러로 등록
- @RequestMapping: URL을 컨트롤러(클래스/메서드)에 매핑
- @GetMapping/@PostMapping/@PutMapping/@DeleteMapping: HTTP Method별 매핑
- @ResponseBody: 반환값을 “뷰 이름”이 아니라 “응답 바디”로 내려줌

## 핵심 개념
### 1) Controller가 왜 필요한가
- 서블릿만으로 구현하면 URL 단위로 클래스가 늘어나기 쉽습니다.
- Spring MVC는 Front Controller 패턴으로 요청 입구를 하나로 모으고,
  URL 매핑 기반으로 컨트롤러 메서드에 연결해서 “구조적으로 관리”할 수 있게 합니다.

### 2) 유사한 API는 하나의 컨트롤러로 묶는다
- 보통 리소스/도메인 단위로 나눕니다.
  - 예: user 관련, order 관련, product 관련
- 한 컨트롤러에 모든 API를 몰아넣지 않습니다.

### 3) 메서드 이름보다 “매핑 애너테이션”이 중요하다
- 어떤 메서드가 호출될지는 메서드 이름이 아니라 매핑 애너테이션의 값(URL, HTTP Method)로 결정됩니다.

### 4) 반환이 View인지 Body인지가 갈린다
- @Controller는 기본적으로 “뷰 렌더링” 흐름과 자주 같이 쓰입니다.
- 문자열을 반환하면 “뷰 이름”으로 해석될 수 있습니다.
- JSON/문자열을 그대로 응답으로 내리려면 @ResponseBody가 필요합니다.

## 동작 방식
1) 요청이 DispatcherServlet로 들어옵니다.
2) HandlerMapping이 URL과 HTTP Method 기준으로 실행할 컨트롤러 메서드를 찾습니다.
3) 컨트롤러 메서드를 실행합니다.
4) 반환값 처리
- @ResponseBody가 있으면 응답 바디로 내려갑니다.
- 없으면 뷰 이름으로 해석되어 뷰 렌더링 흐름으로 갈 수 있습니다.

## 실무 포인트
- API 서버라면 보통 @Controller보다는 @RestController를 기본으로 사용합니다.
  - @RestController는 “클래스 전체에 @ResponseBody가 적용된 효과”입니다.
- 컨트롤러는 얇게 유지하는 편이 좋습니다.
  - 요청 검증, 라우팅, 서비스 호출, 응답 변환 정도만 두고
  - 비즈니스 로직은 서비스로 내려보내는 게 유지보수에 유리합니다.
- 컨트롤러는 기본 싱글톤 빈이므로, 요청별 상태를 필드에 저장하지 않는 게 안전합니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| @Controller | 컨트롤러 빈 등록(웹 요청 처리) |
| @RequestMapping | URL 매핑(클래스/메서드) |
| @GetMapping 등 | HTTP Method별 URL 매핑 |
| @ResponseBody | 반환값을 응답 바디로 사용 |
| @RestController | @Controller + @ResponseBody 효과 |

## 공부하면서 떠올린 질문
- @ResponseBody가 없을 때 문자열 반환이 왜 뷰 이름으로 해석될까?
- 컨트롤러를 도메인 단위로 나누는 기준은 무엇이 좋을까?
- 컨트롤러가 두꺼워지면 테스트/유지보수에서 어떤 문제가 생길까?
