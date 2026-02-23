# gateway-jwt-auth.md

## 개요

게이트웨이에서 JWT 토큰을 검사해서, 인증된 요청만 뒤쪽 서비스로 보내는 구조를 만든다.
로그인은 auth-service가 맡고, 게이트웨이는 들어오는 요청의 Authorization 헤더를 보고 토큰이 유효한지 확인한다.

---

## 보안이 필요한 이유

서비스가 여러 개로 나뉘면 서비스마다 인증을 다 붙이기도 애매하고, 클라이언트가 직접 각 서비스로 붙으면 통제하기도 어렵다.
그래서 게이트웨이를 단일 진입점으로 두고, 앞단에서 인증을 한 번 걸러주는 방식이 많이 쓰인다.

---

## OAuth2 / JWT 간단 정리

* OAuth2는 토큰 기반 인증/인가 프로토콜이다.
* JWT는 토큰 자체에 정보가 들어있는 형태의 토큰이다.
* 이번 구성에서는 “JWT를 발급하는 서비스(auth-service)”와 “JWT를 검증하는 곳(gateway pre filter)”로 나뉜다.

---

## 구성

* Eureka: 서비스 등록/조회
* gateway-service: 라우팅 + JWT 검증
* auth-service: 로그인 요청을 받아 JWT 발급
* product-service: 보호 대상(토큰 없으면 접근 불가)

---

## Auth Service (토큰 발급)

흐름은 단순하다.

1. /auth/signIn 요청
2. user_id를 받아 JWT 생성
3. access_token을 응답으로 내려준다

설정에서 본 것들

* secret-key: 토큰 서명/검증에 쓰는 키
* access-expiration: 토큰 만료 시간

구현 포인트

* jjwt 라이브러리로 토큰 생성
* claim에 user_id, role 같은 값을 넣을 수 있다
* issuer, issuedAt, expiration 같은 기본 정보도 같이 넣는다
* HS512로 서명한다

---

## Gateway (JWT 검증 + 라우팅)

## 1) 라우팅 추가

게이트웨이에 auth-service 라우팅을 추가한다.

* /auth/signIn 은 auth-service로 보낸다
* /order/**, /product/** 는 기존처럼 각 서비스로 보낸다
* uri는 lb://서비스이름 형태로 둬서 Eureka 기반 라우팅이 되게 한다

## 2) JWT 인증 필터

GlobalFilter로 토큰 검증 필터를 하나 둔다.

필터 로직 흐름

1. 요청 path가 /auth/signIn 이면 그냥 통과(로그인 요청은 토큰이 없으니까 예외)
2. Authorization 헤더에서 Bearer 토큰 추출
3. 토큰이 없거나 검증 실패면 401 반환
4. 검증 성공이면 뒤 서비스로 전달

토큰 검증 방식

* secret-key로 서명 검증을 한다
* 파싱이 실패하면 유효하지 않은 토큰으로 처리한다
* 검증 성공하면 payload를 로그로 확인할 수 있다

---

## 실행 흐름과 확인

1. Eureka 실행
2. Gateway 실행
3. Auth 실행
4. Product 실행

확인 순서

* 게이트웨이로 product 요청을 먼저 날리면 401이 떨어진다
* 게이트웨이로 /auth/signIn 요청해서 토큰을 발급받는다
* 발급받은 토큰을 Authorization: Bearer {token} 형태로 넣고 product 요청을 다시 날린다
* 정상 응답이 온다

---

## Bearer 토큰이란

Authorization 헤더에 “Bearer 토큰” 형태로 넣어서 전달하는 방식이다.
클라이언트는 토큰을 헤더에 넣기만 하면 되고, 서버는 서명/만료 같은 유효성 검증을 해서 요청을 통과시킬지 결정한다.

---

## 용어 정리

| 용어                   | 의미                                  |
| -------------------- | ----------------------------------- |
| Auth Service         | 로그인 요청을 받고 토큰을 발급하는 서비스             |
| API Gateway          | 단일 진입점, 라우팅과 공통 필터 처리               |
| JWT                  | 토큰 자체에 정보가 포함된 문자열 토큰               |
| Authorization Header | 요청 인증 정보를 담는 헤더                     |
| Bearer               | OAuth2에서 쓰는 토큰 전달 방식 표기             |
| Claim                | JWT payload에 담기는 값(user_id, role 등) |

---

## 공부하면서 떠올릴 질문

* gateway에서 토큰 검증만 하고, 각 서비스에서는 어떤 검증을 추가로 해야 할까?
* 토큰 만료/갱신은 어떤 방식으로 처리하는 게 자연스러울까?
* secret-key를 코드나 yml에 두지 않으려면 보통 어떤 방식으로 관리할까?
