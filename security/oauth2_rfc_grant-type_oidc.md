# OAuth 2.0 개요

## OAuth 2.0이란?

* 사용자의 자원(API, 데이터)에 대해
  제3의 애플리케이션(Client)에게 제한된 권한만 위임하는 인가(Authorization) 프레임워크.
* 비밀번호를 직접 넘기지 않고, Access Token을 매개로 권한을 위임.
* 소셜 로그인, 파트너 API 연동, 마이크로서비스 간 호출 등에 널리 사용.

---

# OAuth 2.0 역할(Role) 정리

OAuth 2.0에서 자주 등장하는 4가지 역할을 먼저 분리해서 정리.

* Resource Owner
  실제 자원 소유자. 보통 최종 사용자.
* Client
  Resource Owner를 대신해 보호 자원에 접근하려는 애플리케이션.
  예: 우리 서비스 서버, 모바일 앱 등.
* Authorization Server
  로그인/동의 UI를 제공하고 Access Token, Refresh Token, ID Token(OIDC)을 발급하는 서버.
* Resource Server
  보호 자원(API)을 실제로 제공하는 서버.
  예: 구글 프로필 API, 사내 포인트 조회 API 등.

---

# Grant Type이란 무엇인가?

## Grant Type의 정확한 의미

* “클라이언트가 Authorization Server에게 Access Token을 요청하는 방식·시나리오의 종류”
* 토큰 엔드포인트 호출 시 `grant_type` 파라미터로 전달되는 값.
* 어떤 자격 증명과 어떤 플로우를 사용해 토큰을 받는지를 규정하는 단위.

정리하면:

* Role(누가): Resource Owner, Client, AS, RS
* Grant Type(어떻게): Authorization Code, Client Credentials, Refresh Token 등
* Token(무엇을 받나): Access Token, Refresh Token, ID Token(OIDC)

---

# 주요 Grant Type 정리

## Authorization Code Grant

서버 사이드 웹·일반적인 OIDC 로그인에 쓰이는 기본 패턴.

* 용도

  * 브라우저 + 백엔드 서버 구조.
  * 소셜 로그인, 사내 SSO 등.
* 특징

  * 브라우저는 Authorization Code만 받고, 실제 토큰 교환은 백엔드 서버에서 수행.
  * 클라이언트 시크릿을 서버에 안전하게 보관 가능.
* 플로우 요약

  1. 브라우저가 Authorization Server로 리다이렉트 (response_type=code).
  2. 사용자 로그인/동의.
  3. Authorization Server가 redirect_uri로 code 전달.
  4. 클라이언트 서버가 토큰 엔드포인트에 code를 보내 Access Token(및 Refresh/ID Token) 획득.

`grant_type=authorization_code`

---

## Authorization Code + PKCE

SPA·모바일 같은 “공용 클라이언트”에서 Authorization Code를 안전하게 쓰기 위한 확장.

* 용도

  * 모바일 앱, SPA(React, Vue 등).
  * 클라이언트 시크릿을 숨기기 어려운 환경.
* 특징

  * code_verifier / code_challenge를 이용해 “code 탈취” 방어.
  * 요즘은 웹/앱 모두 Authorization Code + PKCE를 권장.
* 플로우 요약

  1. 클라이언트가 임의 문자열 code_verifier 생성.
  2. 이를 해시해 code_challenge로 authorization 요청에 포함.
  3. 토큰 교환 시 code와 함께 code_verifier를 보내 검증.

`grant_type=authorization_code` 이지만, PKCE 관련 파라미터를 함께 사용.

---

## Client Credentials Grant

사용자 계정이 아닌 “애플리케이션 자체”를 인증할 때.

* 용도

  * 서버-서버 통신, 내부 마이크로서비스 간 호출.
  * 배치 서버 → API 서버 호출 등.
* 특징

  * Resource Owner(사용자) 없이 Client 자체가 주체.
  * Client ID/Secret로 토큰 발급.
* 플로우 요약

  1. 클라이언트가 Client ID/Secret로 토큰 엔드포인트에 요청.
  2. Authorization Server가 Access Token 발급.
  3. 이 토큰으로 Resource Server 호출.

`grant_type=client_credentials`

---

## Resource Owner Password Credentials Grant (ROPC, 지양)

이제는 거의 쓰지 말라고 권장되는 방식.

* 용도

  * 레거시 환경, 완전히 신뢰되는 1st-party 클라이언트에서만 사용 가능.
* 특징

  * 클라이언트가 사용자의 ID/PW 자체를 받아서 Authorization Server에 전달.
  * ID/PW를 클라이언트와 공유한다는 점에서 현대적인 보안 기준에 맞지 않음.
* 플로우 요약

  1. 클라이언트가 로그인 폼에서 ID/PW 수집.
  2. 토큰 엔드포인트에 ID/PW와 함께 요청.
  3. Access Token 발급.

`grant_type=password` (실무에서는 거의 피해야 하는 값)

---

## Refresh Token Grant

이미 발급된 Refresh Token을 사용해 Access Token을 재발급 받을 때 쓰는 별도 Grant Type.

* 용도

  * Access Token을 짧게 유지하면서 UX는 유지.
* 특징

  * 보안 수준이 높은 저장소에만 보관해야 함.
  * 탈취되면 장기간 악용 가능.
* 플로우 요약

  1. 클라이언트가 만료되거나 만료 직전인 Access Token 대신 Refresh Token으로 토큰 엔드포인트 호출.
  2. Authorization Server가 새 Access Token(및 새 Refresh Token)을 발급.

`grant_type=refresh_token`

---

## Device Code Grant (옵션)

TV, 콘솔 등 키보드 입력이 어려운 디바이스에서 사용하는 별도 플로우(RFC 8628).

* 용도

  * 스마트 TV 로그인, 셋톱박스, 콘솔 등.
* 특징

  * 디바이스에 user code / verification URL을 보여주고,
    사용자가 별도 기기(모바일·PC)로 로그인·동의.
* 플로우 요약

  1. 디바이스에서 device_code, user_code를 발급받음.
  2. 사용자가 다른 기기에서 user_code로 인증.
  3. 디바이스가 주기적으로 토큰 엔드포인트를 폴링해 Access Token 획득.

`grant_type=urn:ietf:params:oauth:grant-type:device_code` 등 구현체마다 약간씩 다를 수 있음.

---

# Authorization Code Flow 예시 (간단)

브라우저 + 백엔드 서버 환경에서의 기본 흐름.

1. 사용자가 클라이언트(우리 서비스)의 로그인 버튼 클릭.
2. 클라이언트가 Authorization Server로 리다이렉트.

   * client_id, redirect_uri, scope, response_type=code, state, (PKCE면 code_challenge)
3. 사용자는 Authorization Server에서 로그인·동의.
4. Authorization Server가 redirect_uri로 Authorization Code 전달.
5. 클라이언트 서버가 Authorization Server의 토큰 엔드포인트로 코드 교환 요청.
6. Authorization Server가 Access Token(+ Refresh Token, OIDC면 ID Token 포함)을 응답.
7. 클라이언트 서버는 Access Token으로 Resource Server 호출,
   OIDC면 ID Token을 검증하고 세션/쿠키 생성.

---

# 토큰과 데이터 구조 예시

## Token Request / Response 예시

토큰 요청

```http
POST /oauth2/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=abc123&
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcallback&
client_id=client-app&
client_secret=client-secret&
code_verifier=original-code-verifier
```

토큰 응답

```json
{
  "access_token": "ACCESS_TOKEN_VALUE",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "REFRESH_TOKEN_VALUE",
  "scope": "openid profile email",
  "id_token": "ID_TOKEN_JWT_VALUE"
}
```

---

# OpenID Connect(OIDC) 개요

## OIDC란?

* OAuth 2.0 위에 “로그인(사용자 인증)”을 표준화한 프로토콜/스펙.
* OAuth 2.0은 인가(권한 위임)에 초점.
  OIDC는 “이 토큰이 누구의 것인지”를 표준 형태로 담아주는 ID 계층.

## OIDC가 표준화한 것

* ID Token

  * JWT 형식으로 사용자 ID·인증 정보(sub, iss, aud, exp, iat 등)를 담는 토큰.
  * 클라이언트가 이 토큰을 검증해 “어떤 사용자가, 언제, 어떤 AS에서 인증되었는지” 판단.
* Discovery 문서

  * `/.well-known/openid-configuration`
  * authorization_endpoint, token_endpoint, jwks_uri, issuer 등 메타데이터 표준.
* UserInfo Endpoint

  * Access Token으로 호출하는 사용자 정보 API.
  * 이메일, 이름 등 기본 프로필을 표준 JSON 필드로 제공.
* Scope

  * `openid` 필수.
  * `profile`, `email`, `phone`, `address` 등 확장 스코프 정의.

## OIDC와 OAuth 2.0 관계

* 토큰 발급, Authorization Code Flow 자체는 OAuth 2.0 규칙을 그대로 사용.
* 다만, 발급되는 토큰 중 하나로 ID Token이 추가되고,
  Discovery, UserInfo, 표준 클레임 형식을 요구하는 것이 OIDC의 역할.
* 그래서 자주 쓰는 패턴이
  “OIDC + Authorization Code Flow(+PKCE)” = 소셜 로그인 / SSO.

---

# OIDC 흐름 예시 (Authorization Code)

1. 클라이언트가 Authorization Server(OIDC Provider)로 리다이렉트

   * scope에 최소 `openid` 포함 (`openid profile email` 등).
2. 사용자 로그인·동의.
3. Authorization Server가 Authorization Code를 redirect_uri로 전달.
4. 클라이언트가 토큰 엔드포인트로 code를 교환해 Access Token + ID Token(+ Refresh Token)을 받음.
5. 클라이언트는

   * ID Token(JWT)의 서명과 클레임(iss, aud, exp 등)을 검증해 사용자 ID(sub)를 확인.
   * 필요하면 Access Token으로 UserInfo Endpoint를 호출해 상세 프로필 조회.
6. 애플리케이션은 검증된 사용자 ID를 자체 계정과 매핑하고 세션/쿠키를 생성.

---

# 용어 정리

## OAuth 2.0 핵심 용어

| 키워드                  | 설명                                           |
| -------------------- | -------------------------------------------- |
| OAuth 2.0            | 제3자 클라이언트에게 자원 접근 권한을 위임하는 인가 프레임워크          |
| Resource Owner       | 실제 자원 소유자. 보통 최종 사용자                         |
| Client               | 자원에 접근하려는 애플리케이션                             |
| Authorization Server | 로그인·동의 처리, 토큰 발급 담당 서버                       |
| Resource Server      | Access Token을 검증하고 보호 자원을 제공하는 API 서버        |
| Access Token         | 보호 자원 접근에 사용하는 토큰                            |
| Refresh Token        | Access Token을 재발급 받을 때 사용하는 토큰               |
| Scope                | 토큰이 가지는 권한 범위                                |
| Grant Type           | 클라이언트가 토큰을 얻는 방식·플로우 종류. `grant_type` 파라미터 값 |

## Grant Type 값

| Grant Type 값       | 설명                                     |
| ------------------ | -------------------------------------- |
| authorization_code | 브라우저 + 백엔드 서버에서 쓰는 기본 코드 기반 플로우        |
| client_credentials | 서버-서버 통신 등 클라이언트 애플리케이션 자체를 인증할 때      |
| password           | 사용자의 ID/PW를 직접 넘기는 레거시 플로우. 지양해야 함     |
| refresh_token      | Refresh Token으로 새 Access Token을 발급받을 때 |
| device_code 등      | TV, 콘솔 등 입력이 어려운 디바이스용 플로우(RFC 8628)   |

## OIDC 관련 용어

| 키워드                  | 설명                                                          |
| -------------------- | ----------------------------------------------------------- |
| OpenID Connect(OIDC) | OAuth 2.0 위에 로그인(사용자 인증)을 표준화한 프로토콜                         |
| ID Token             | 사용자 ID·인증 정보를 담은 JWT. OIDC의 핵심 결과물                          |
| UserInfo Endpoint    | Access Token으로 호출하는 표준 사용자 정보 API                           |
| Discovery Document   | OIDC Provider 메타데이터 문서(`/.well-known/openid-configuration`) |
| iss                  | Issuer. 토큰 발급자 식별자(Authorization Server URL)                |
| sub                  | Subject. 사용자 고유 ID                                          |
| aud                  | Audience. 토큰 수신 대상(Client ID)                               |
| OIDC Provider / IdP  | OIDC·OAuth 2.0을 제공하는 로그인 서버, 인증 공급자                         |

