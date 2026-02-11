# Spring MVC Basic

## 개요
Spring MVC는 웹 요청을 받아서 컨트롤러로 보내고, 처리 결과를 응답으로 돌려주는 스프링의 웹 프레임워크입니다.
Spring Boot에서는 내부적으로 DispatcherServlet이 프론트 컨트롤러 역할을 하면서 요청 흐름을 표준화합니다.

- 프론트 컨트롤러는 “모든 HTTP 요청이 먼저 들어오는 단일 진입점”입니다. Spring MVC에서는 DispatcherServlet이 그 역할

## 정의
- Spring MVC: 서블릿(Servlet) 기반의 웹 요청 처리 프레임워크
- MVC: Model(데이터/비즈니스), View(화면), Controller(요청 처리) 역할을 분리하는 패턴
- DispatcherServlet: 모든 HTTP 요청의 진입점(Front Controller)
- Controller: 요청을 받아서 서비스 호출, 결과를 반환하는 계층
- HandlerMapping: 어떤 URL 요청을 어떤 컨트롤러 메서드로 보낼지 찾는 역할
- HandlerAdapter: 찾은 컨트롤러를 실제로 실행해주는 역할
- HttpMessageConverter: 객체를 JSON 같은 응답 바디로 변환하거나, 요청 바디(JSON)를 객체로 변환하는 역할

## 핵심 개념
1) Front Controller 패턴
- 요청 입구를 DispatcherServlet 하나로 통일합니다.
- 공통 흐름(인자 바인딩, 예외 처리, 응답 변환)을 한 곳에서 재사용합니다.

2) URL 매핑과 메서드 실행 분리
- HandlerMapping이 "누가 처리할지"를 찾고,
- HandlerAdapter가 "어떻게 실행할지"를 책임집니다.
- 그래서 컨트롤러 구현 방식이 달라도 같은 흐름으로 동작할 수 있습니다.

3) API 서버에서는 View보다 JSON 응답이 기본
- @RestController 또는 @ResponseBody를 쓰면 View 렌더링 대신 응답 바디(JSON)를 만듭니다.
- 이때 HttpMessageConverter가 핵심입니다.

4) 싱글톤 컨트롤러와 요청 스레드
- 컨트롤러 빈은 기본 싱글톤입니다.
- 요청은 보통 "요청 1개, 스레드 1개"로 처리됩니다.
- 그래서 컨트롤러 필드에 요청 상태를 저장하면 동시성 문제가 생깁니다.

## 동작 방식
1) 클라이언트가 HTTP 요청을 보냅니다.
2) 서블릿 컨테이너(Tomcat)가 요청을 받고 DispatcherServlet에 전달합니다.
3) DispatcherServlet이 HandlerMapping으로 컨트롤러 메서드를 찾습니다.
4) HandlerAdapter가 컨트롤러 메서드를 실행합니다.
5) 실행 중 요청 파라미터, 경로 변수, 바디(JSON)가 메서드 파라미터로 바인딩됩니다.
6) 컨트롤러 반환값을 기준으로 응답을 만듭니다.
- 화면 응답이면 ViewResolver를 통해 뷰를 찾고 렌더링합니다.
- API 응답이면 HttpMessageConverter로 JSON을 만들어 반환합니다.

## 전체 흐름
1) HTTP 요청 수신
2) (필요 시) Filter 체인 실행 (예: 인증, 로깅, CORS)
3) DispatcherServlet 진입
4) HandlerMapping이 컨트롤러 메서드 선택
5) HandlerAdapter가 컨트롤러 호출
6) 파라미터 바인딩
- @RequestParam, @PathVariable, @RequestHeader
- @RequestBody (JSON 바디를 객체로 변환)
7) 컨트롤러 로직 실행 (서비스 호출)
8) 반환값 처리
- @ResponseBody 또는 @RestController면 JSON 응답
- 아니면 View 이름으로 보고 화면 렌더링
9) 예외 발생 시 예외 처리 체인(@ControllerAdvice 등)으로 응답 생성
10) HTTP 응답 반환

## 실무 포인트
- API 서버라면 @Controller보다 @RestController가 기본입니다.
- 요청 DTO 검증(@Valid)과 예외 처리(@RestControllerAdvice)를 같이 묶어서 설계하는 게 실무 기본입니다.
- Filter와 Interceptor를 구분해두면 장애 분석이 빨라집니다.
  - Filter: DispatcherServlet 앞단, 요청/응답 자체 중심
  - Interceptor: 컨트롤러 전후, 어떤 핸들러(메서드)인지까지 접근 가능
- 컨트롤러/서비스에 요청별 상태를 필드로 저장하면 동시성 버그가 나기 쉽습니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| Spring MVC | 서블릿 기반 웹 요청 처리 프레임워크 |
| MVC | Model, View, Controller 역할 분리 패턴 |
| DispatcherServlet | 모든 요청의 진입점(Front Controller) |
| Controller | 요청 처리 담당 계층 |
| HandlerMapping | 요청 URL과 컨트롤러 메서드 매핑 |
| HandlerAdapter | 컨트롤러를 실제로 실행 |
| ViewResolver | View 이름을 실제 View로 해석 |
| HttpMessageConverter | 요청/응답 바디 변환(JSON 등) |
| @RestController | 반환값을 응답 바디로 처리(주로 JSON) |
| @RequestBody | 요청 바디(JSON)를 객체로 변환 |

## 공부하면서 떠올린 질문
- DispatcherServlet이 없으면 각 요청마다 무엇이 불편해질까?
- HandlerMapping과 HandlerAdapter를 왜 굳이 분리했을까?
- @Controller와 @RestController의 가장 큰 차이는 무엇일까?
- Filter와 Interceptor는 언제 어떤 기준으로 나눠서 쓸까?
- 컨트롤러 필드에 값을 저장하면 어떤 상황에서 버그가 터질까?
