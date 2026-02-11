# Jackson Basic

## 개요
Jackson은 JSON 데이터를 다루기 위한 라이브러리입니다.
자바 객체를 JSON 문자열로 바꾸거나, JSON 문자열을 자바 객체로 바꾸는 일을 합니다.

Spring은 3.0 이후부터 Jackson 관련 API를 제공해서, 우리가 JSON 변환 코드를 직접 많이 쓰지 않아도 자동으로 처리해주는 흐름이 많습니다.
Spring Boot의 starter-web에는 Jackson 관련 라이브러리가 기본으로 포함됩니다.

## 정의
- Jackson: JSON 데이터 구조를 처리하는 라이브러리
- 직렬화(Serialization): Object를 JSON 문자열로 변환
- 역직렬화(Deserialization): JSON 문자열을 Object로 변환
- ObjectMapper: Jackson에서 변환을 수행하는 핵심 도구

## 핵심 개념
### 1) Spring MVC에서 Jackson이 왜 중요한가
- 컨트롤러에서 객체를 반환하면 JSON으로 응답이 내려가는 경우가 많습니다.
- 이때 내부적으로 Jackson이 객체를 JSON으로 변환합니다.

### 2) ObjectMapper는 언제 쓰는가
- 보통은 Spring이 자동 변환을 해주므로 직접 쓸 일이 줄어듭니다.
- 하지만 아래처럼 "직접 JSON 변환이 필요한 상황"에서는 ObjectMapper를 사용합니다.
  - 외부 API 연동에서 JSON 문자열을 직접 파싱해야 할 때
  - 로그나 메시지에 JSON 문자열로 담아야 할 때
  - 테스트에서 변환 결과를 검증해야 할 때

### 3) 변환에 필요한 클래스 조건(강의 기준)
- Object를 JSON 문자열로 바꾸려면 보통 getter가 필요합니다.
- JSON 문자열을 Object로 바꾸려면 기본 생성자와 getter 또는 setter가 필요합니다.

## 동작 방식
1) 직렬화(Object to JSON)
- 자바 객체를 입력으로 받아 JSON 문자열을 생성합니다.
- 내부적으로 객체의 프로퍼티를 읽어 JSON 필드로 만듭니다.

2) 역직렬화(JSON to Object)
- JSON 문자열과 목표 클래스 타입을 입력으로 받아 객체를 생성합니다.
- 내부적으로 기본 생성자로 객체를 만들고, JSON 필드를 객체에 매핑합니다.

## 전체 흐름
- 자동 흐름(Spring MVC)
  1) 컨트롤러가 객체를 반환
  2) Spring이 응답 바디를 만들 때 Jackson을 호출
  3) 객체가 JSON으로 변환되어 응답됨

- 수동 흐름(직접 변환)
  1) ObjectMapper를 준비
  2) 직렬화 또는 역직렬화를 호출
  3) JSON 문자열 또는 객체를 얻음

## 실무 포인트
- API 서버에서 JSON 응답이 기대와 다르게 나오면, Jackson 변환 과정에서의 조건 문제를 먼저 의심합니다.
  - getter 누락
  - 기본 생성자 누락
  - 필드명과 JSON 키 매칭 문제
- Spring Boot에서는 Jackson이 기본 탑재라서, "직접 JSON 문자열 만들기"는 피하고 객체 기반 응답을 유지하는 편이 안정적입니다.
- 테스트에서는 변환 결과를 확인하기 위해 ObjectMapper를 직접 쓰는 경우가 많습니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| Jackson | JSON 처리 라이브러리 |
| 직렬화 | Object를 JSON 문자열로 변환 |
| 역직렬화 | JSON 문자열을 Object로 변환 |
| ObjectMapper | Jackson 변환 도구 |

## 공부하면서 떠올린 질문
- Spring MVC에서 객체를 반환하면, Jackson은 정확히 어느 시점에 호출될까?
- getter나 기본 생성자가 없으면 어떤 형태로 실패할까?
- JSON 키 이름과 자바 필드 이름이 다르면 어떤 문제가 생길까?
