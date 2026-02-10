# 클라이언트 → 웹서버 → Tomcat → Spring 컨테이너 요청 처리 흐름

## 1. 전체 개요

클라이언트가 보낸 HTTP 요청은 일반적으로 다음과 같은 순서로 흐릅니다.

1. 클라이언트(브라우저, 앱)가 HTTP 요청 전송
2. 웹서버(Nginx, Apache, 혹은 Tomcat의 웹서버 부분)가 요청 수신
3. 정적 리소스인 경우 웹서버에서 바로 응답, 동적 처리 필요 시 WAS(Tomcat 서블릿 컨테이너)로 전달
4. Tomcat 서블릿 컨테이너가 FilterChain과 DispatcherServlet을 통해 Spring 컨테이너(WebApplicationContext, Root WebApplicationContext)의 컴포넌트들과 상호작용하여 요청 처리
5. Spring 영역의 컨트롤러, 서비스, 리포지토리, Spring Security 등이 동작한 뒤 응답을 생성하고, 이를 역방향으로 전송하여 클라이언트에 응답

이 문서는 특히 다음 두 영역을 중심으로 정리합니다.

* 서블릿 컨테이너 필터 체인(FilterChain)
* Spring MVC 인터셉터 체인(HandlerInterceptor) 및 내부 단계

---

## 2. 전체 시퀀스 요약

HTTP 요청 처리 흐름을 단계별로 요약하면 다음과 같습니다.

1. 클라이언트가 HTTP 요청 전송

2. 웹서버가 정적/동적 요청을 구분

3. 동적 요청은 Tomcat 서블릿 컨테이너로 전달

4. Tomcat 필터 체인(FilterChain) 실행

   * 인코딩 필터, 로깅 필터, DelegatingFilterProxy 등

5. DelegatingFilterProxy가 Spring Security FilterChainProxy 호출

   * SecurityFilterChain 여러 개를 거치며 인증/인가 수행

6. 필터 체인 마지막에서 DispatcherServlet 호출

7. DispatcherServlet 내부 처리

   * HandlerInterceptor `preHandle`
   * HandlerMapping으로 컨트롤러 검색
   * HandlerAdapter를 통해 컨트롤러 실행
   * 비즈니스 로직(Service, Repository 등) 수행
   * HandlerExceptionResolver로 예외 처리
   * ViewResolver 또는 HttpMessageConverter를 통한 응답 변환
   * HandlerInterceptor `postHandle`, `afterCompletion`

8. DispatcherServlet의 반환값이 서블릿 컨테이너 → 웹서버 → 클라이언트로 전송

---

## 3. 웹서버(Web Server) 레이어

### 3-1. Web Server 역할

웹서버는 다음과 같은 역할을 수행합니다.

* 정적 리소스(HTML, CSS, JS, 이미지)를 직접 서빙
* 동적 처리 필요 시 요청을 WAS(Tomcat)로 포워딩

실제 운영 환경에서는 다음과 같은 구성이 많이 사용됩니다.

* 운영 환경: Nginx/Apache를 프론트로 두고, 뒤에 Tomcat을 여러 대 배치
* 개발 환경: Spring Boot 내장 Tomcat만 단독으로 사용하는 경우

---

## 4. Tomcat과 Servlet Container 레이어

### 4-1. 서블릿 컨테이너(Servlet Container)의 정의

서블릿 컨테이너는 다음과 같은 특징을 가집니다.

* 웹 애플리케이션을 실행하고 관리하는 환경
* JSP, Servlet 같은 웹 컴포넌트를 실행할 수 있도록 해주는 서버
* Tomcat은 자바 서블릿과 JSP를 실행하는 웹 서버이자 서블릿 컨테이너 역할을 수행

주요 역할은 다음과 같습니다.

* HTTP 요청을 적절한 서블릿에 매핑
* `HttpServletRequest` / `HttpServletResponse` 객체 생성 및 전달
* FilterChain 실행
* 서블릿의 초기화 및 소멸 관리
* 멀티 스레딩 기반 요청 처리

### 4-2. FilterChain

필터 체인(FilterChain)은 서블릿 실행 전후에 여러 Filter를 연결한 구조입니다.

각 Filter는 다음과 같은 책임을 가집니다.

* 요청 사전 처리

  * 인코딩 설정
  * 공통 로그 기록
  * 보안 관련 프리 체크 등

* 응답 후처리

  * 응답 헤더 추가
  * 응답 로그 기록 등

* 다음 필터로 요청을 넘길지, 현재 필터에서 응답을 종료할지 결정

일반적인 필터 체인의 예시는 다음과 같습니다.

* `CharacterEncodingFilter`
* 로깅 필터
* `DelegatingFilterProxy` (Spring Security 진입점)
* 기타 사용자 정의 필터
* 마지막에 DispatcherServlet

### 4-3. DelegatingFilterProxy

`DelegatingFilterProxy`는 서블릿 컨테이너 필터 체인에 등록되는 Spring 제공 필터입니다.

* 자체적으로는 실제 보안 로직을 수행하지 않습니다.
* Spring ApplicationContext에 정의된 `FilterChainProxy` Bean으로 요청을 위임합니다.

즉, 서블릿 컨테이너 세계의 필터 체인과 Spring Security 필터 체인을 연결하는 브리지 역할을 수행합니다.

### 4-4. DispatcherServlet (Front Controller)

DispatcherServlet은 Spring MVC의 Front Controller 역할을 하는 서블릿입니다.

Tomcat 관점

* 하나의 서블릿으로 등록되며, URL 매핑에 따라 호출됩니다.

Spring 관점

* Spring MVC의 중심 컴포넌트로 동작합니다.
* 다음과 같은 컴포넌트들과 상호작용합니다.

  * HandlerInterceptor
  * HandlerMapping
  * HandlerAdapter
  * HandlerExceptionResolver
  * ViewResolver

DispatcherServlet은 서블릿 컨테이너와 Spring 컨테이너의 경계에 위치한 특수한 서블릿으로, 두 세계를 연결하는 핵심 진입점입니다.

---

## 5. Spring 컨테이너 구조

Spring은 일반적으로 두 레벨의 컨텍스트를 사용합니다.

* Root WebApplicationContext
* Servlet WebApplicationContext

### 5-1. Root WebApplicationContext

Root WebApplicationContext는 애플리케이션 전체에서 공유되는 전역 Bean을 관리하는 최상위 컨텍스트입니다.

대표적으로 다음과 같은 Bean들이 포함됩니다.

* Service 계층

  * 비즈니스 로직을 담당
  * 여러 Repository와 연동

* Repository 계층

  * 데이터베이스와 직접 통신
  * JPA, MyBatis 등 ORM/매퍼와 연계

* Spring Security 관련 Bean

  * `SecurityContextHolder`
  * `AuthenticationManager`
  * `SecurityFilterChain`
  * `FilterChainProxy` 등

### 5-2. Servlet WebApplicationContext

Servlet WebApplicationContext는 DispatcherServlet이 생성될 때 함께 만들어지는 하위 컨텍스트입니다.

주로 웹 요청 처리에 직접 관련된 Bean들을 포함합니다.

예시

* Controller
* HandlerMapping
* HandlerAdapter
* HandlerInterceptor
* ViewResolver
* WebMvcConfigurer 및 기타 웹 설정 Bean

컨텍스트 간 관계

* Root WebApplicationContext: 부모 컨텍스트
* Servlet WebApplicationContext: 자식 컨텍스트

자식 컨텍스트는 부모 컨텍스트의 Bean에 접근할 수 있으며, 웹 계층에서 비즈니스/데이터 계층의 Bean을 활용하는 구조가 됩니다.

---

## 6. DispatcherServlet 내부 처리 단계

DispatcherServlet 내부에서 요청이 처리되는 주요 단계들을 Spring MVC 컴포넌트 이름과 함께 정리하면 다음과 같습니다.

### 6-1. HandlerInterceptor

HandlerInterceptor는 DispatcherServlet 내부에서 컨트롤러 호출 전후에 동작하는 컴포넌트입니다.

주요 메서드

* `preHandle`

  * 컨트롤러 호출 전 실행
  * 인증 검사, 접근 권한 체크, 로깅, 공통 파라미터 세팅 등에 사용
  * `false`를 반환하면 이후 체인을 중단하고 바로 응답을 반환할 수 있습니다.

* `postHandle`

  * 컨트롤러가 정상적으로 반환한 후, View 렌더링 전에 호출
  * Model 수정, 응답 데이터 추가 가공 등에 사용

* `afterCompletion`

  * View 렌더링까지 모두 끝난 후 호출
  * 예외 발생 여부와 관계없이 실행
  * 리소스 정리, 최종 로깅 등을 수행

관점 비교

* Filter: 서블릿 컨테이너 레벨에서 동작
* HandlerInterceptor: Spring MVC(DispatcherServlet 내부) 레벨에서 동작

### 6-2. HandlerMapping

HandlerMapping은 요청 URL과 HTTP 메서드에 맞는 컨트롤러(핸들러)를 찾는 역할을 수행합니다.

예시

* 요청 경로: `/user/profile`
* HTTP 메서드: `GET`
* 매핑 정보에 따라 해당 경로와 메서드에 연결된 컨트롤러 메서드를 탐색

스프링 부트 환경에서는 `RequestMappingHandlerMapping`이 `@RequestMapping`, `@GetMapping`, `@PostMapping` 등이 붙은 메서드와 URL을 매핑합니다.

### 6-3. HandlerAdapter

HandlerAdapter는 HandlerMapping이 찾은 핸들러를 실제로 호출하는 역할을 담당합니다.

이 컴포넌트가 필요한 이유는 핸들러의 형태가 다양하기 때문입니다.

* `@RequestMapping` 기반 컨트롤러
* 특정 인터페이스를 구현한 컨트롤러
* HTTP 요청 전용 핸들러 등

HandlerAdapter는 각 핸들러 타입에 맞는 호출 방식을 알고 있으며, 다음과 같은 일을 수행합니다.

* 요청 파라미터 → 메서드 파라미터 바인딩
* 메시지 컨버터를 사용한 JSON → 객체, 객체 → JSON 변환
* 컨트롤러 메서드를 호출하고 결과(ModelAndView, DTO, ResponseEntity 등)를 DispatcherServlet에 반환

### 6-4. HandlerExceptionResolver

HandlerExceptionResolver는 컨트롤러 혹은 그 이후 단계에서 예외가 발생했을 때 이를 처리하는 컴포넌트입니다.

역할

* 발생한 예외를 해석하여 적절한 HTTP 상태코드와 응답 바디로 변환합니다.

* 예시

  * 인증 실패 → 401 Unauthorized
  * 권한 없음 → 403 Forbidden
  * 리소스 없음 → 404 Not Found
  * 서버 내부 오류 → 500 Internal Server Error

* `@RestControllerAdvice` 와 결합하여 공통 에러 응답 형식을 제공하는 경우가 많습니다.

### 6-5. ViewResolver

ViewResolver는 컨트롤러가 반환한 뷰 이름을 실제 뷰 리소스로 변환하는 역할을 합니다.

예시

* 컨트롤러 반환 값: `"user/profile"`
* ViewResolver가 이를 `templates/user/profile.html` 과 같은 실제 템플릿 리소스로 해석

전통적인 MVC에서는 템플릿 엔진(Thymeleaf, JSP 등)을 통해 HTML을 렌더링합니다.
REST API 서버에서는 템플릿 대신 HttpMessageConverter를 사용해 객체를 JSON, XML 등의 포맷으로 변환하여 응답합니다.

---

## 7. Spring Security 구성과 필터 체인

### 7-1. Spring Security 주요 구성 요소

Spring Security는 다음과 같은 핵심 컴포넌트들로 구성됩니다.

* `SecurityContextHolder`

  * 현재 스레드(요청)에 대한 인증 정보를 저장하는 컨텍스트

* `AuthenticationManager`

  * 로그인 시 사용자 정보 검증을 수행하는 컴포넌트

* `SecurityFilterChain`

  * 여러 보안 필터(인증, 인가, 세션 관리, CSRF 등)로 구성된 체인

* `FilterChainProxy`

  * 여러 SecurityFilterChain을 관리하는 프록시 Bean

### 7-2. DelegatingFilterProxy와 FilterChainProxy

두 컴포넌트의 역할은 다음과 같이 정리할 수 있습니다.

* DelegatingFilterProxy

  * 서블릿 컨테이너(Tomcat)의 Filter로 등록되는 Spring 제공 필터
  * Spring Security의 진입점 역할
  * 실제 보안 처리는 수행하지 않고, Spring ApplicationContext에 존재하는 `FilterChainProxy` Bean으로 위임

* FilterChainProxy

  * Spring ApplicationContext에 등록된 Bean
  * 하나 이상의 `SecurityFilterChain`을 관리
  * DelegatingFilterProxy로부터 요청을 전달받아, 보안 필터를 순서대로 실행

요청 처리 흐름

1. Tomcat FilterChain에서 DelegatingFilterProxy 실행
2. DelegatingFilterProxy가 Spring 컨테이너의 FilterChainProxy Bean 호출
3. FilterChainProxy가 SecurityFilterChain 내부의 보안 필터들을 순서대로 실행
4. 인증/인가 절차가 통과되면 DispatcherServlet으로 제어가 넘어감

---

## 8. Filter와 HandlerInterceptor 비교

Filter와 HandlerInterceptor는 모두 “공통 처리”를 담당하지만, 동작 위치와 용도가 다릅니다.

| 구분                | Filter                                       | HandlerInterceptor                               |
| ----------------- | -------------------------------------------- | ------------------------------------------------ |
| 실행 위치             | 서블릿 컨테이너(Tomcat) 레벨                          | Spring MVC(DispatcherServlet 내부) 레벨              |
| 체인 이름             | FilterChain                                  | HandlerInterceptor 체인                            |
| 대표 구현             | `DelegatingFilterProxy`, 인코딩/로깅 필터 등         | `preHandle`, `postHandle`, `afterCompletion` 메서드 |
| 주요 목적             | 인코딩, 로깅, 보안, CORS 등 저수준 공통 처리                | 컨트롤러 전후의 공통 로직(인증 체크, 모델 가공 등)                   |
| HttpServlet 의존 여부 | `HttpServletRequest/Response`를 직접 다루는 경우가 많음 | 보다 추상화된 Web 요청/응답 모델 사용                          |
| 관할 범위             | Spring 바깥 영역까지 포함 가능                         | DispatcherServlet 이후, Spring MVC 영역에 한정          |

요약하면,

* Filter는 톰캣 수준에서 전체 요청을 가로채는 “문지기” 역할에 가깝고,
* HandlerInterceptor는 Spring MVC 내부에서 컨트롤러 앞뒤를 감싸는 “얇은 레이어”에 가깝습니다.

---

## 9. 키워드 정리

| 키워드                           | 설명                                                                                          |
| ----------------------------- | ------------------------------------------------------------------------------------------- |
| 클라이언트(Client)                 | 브라우저, 모바일 앱 등 서버에 HTTP 요청을 보내는 주체                                                           |
| 웹서버(Web Server)               | 정적 리소스를 직접 서빙하고, 동적 요청을 WAS로 포워딩하는 서버(Nginx, Apache 등)                                      |
| WAS                           | Web Application Server. 동적 웹 애플리케이션을 실행하는 서버. 여기서는 Tomcat 서블릿 컨테이너를 의미                      |
| 서블릿 컨테이너(Servlet Container)   | 서블릿, JSP 등 웹 컴포넌트를 실행·관리하고 HTTP 요청을 서블릿으로 매핑하는 런타임 환경                                       |
| Filter                        | 서블릿 실행 전후에 요청·응답을 가로채어 인코딩, 로깅, 보안 등 공통 처리를 수행하는 컴포넌트                                       |
| FilterChain                   | 여러 Filter를 순서대로 연결한 체인 구조로, 각 Filter가 다음 Filter로 제어를 넘기거나 요청을 종료할 수 있는 구조                   |
| DelegatingFilterProxy         | 서블릿 컨테이너에 Filter로 등록되며, 실제 처리는 Spring 컨테이너의 `FilterChainProxy` Bean에 위임하는 Spring 제공 필터      |
| FilterChainProxy              | Spring Security의 보안 필터 체인을 담고 있는 프록시 Bean. 여러 `SecurityFilterChain`을 관리하고 실행                |
| DispatcherServlet             | Spring MVC의 Front Controller 서블릿으로, 요청을 핸들러/뷰 등 Spring 컴포넌트에 분배하고 최종 응답을 생성하는 진입점           |
| Root WebApplicationContext    | 애플리케이션 전체에서 공유되는 최상위 Spring 컨텍스트. Service, Repository, Security 등 전역 Bean을 관리               |
| Servlet WebApplicationContext | DispatcherServlet별로 생성되는 하위 웹 컨텍스트. Controller, HandlerMapping, ViewResolver 등 웹 관련 Bean 포함 |
| HandlerInterceptor            | DispatcherServlet 내부에서 컨트롤러 호출 전·후·완료 시점에 공통 로직을 실행하는 인터셉터 컴포넌트                             |
| HandlerMapping                | 들어온 요청(URL, HTTP 메서드 등)에 매핑되는 컨트롤러(핸들러)를 찾아주는 컴포넌트                                          |
| HandlerAdapter                | HandlerMapping이 찾은 핸들러를 실제로 호출하고, 요청/응답 변환 및 바인딩을 담당하는 어댑터 컴포넌트                             |
| HandlerExceptionResolver      | 컨트롤러나 그 이후 처리 중 발생한 예외를 적절한 HTTP 응답(상태 코드, 바디)으로 변환하는 컴포넌트                                  |
| ViewResolver                  | 컨트롤러가 반환한 뷰 이름을 실제 뷰 리소스(템플릿 파일 등)로 해석하는 컴포넌트                                               |
| SecurityContextHolder         | 현재 스레드(요청)에 대한 인증 정보(Authentication)를 보관하는 Spring Security의 컨텍스트                            |
| AuthenticationManager         | 로그인 시 사용자 인증을 수행하는 컴포넌트. 아이디/비밀번호 검증 후 Authentication 객체를 생성                                |
| SecurityFilterChain           | 인증, 인가, 세션 관리, CSRF 등 보안 관련 Filter들을 순서대로 묶어 놓은 Spring Security 필터 체인                       |
| Spring Security               | 인증·인가, 세션 관리, CSRF 방어 등 보안 기능을 제공하는 Spring 기반 보안 프레임워크                                      |
