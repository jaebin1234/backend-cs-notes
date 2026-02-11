# http-object-data-controller-processing

## 개요
컨트롤러는 HTTP 요청으로 들어온 데이터를 꺼내서 자바 타입(문자열, 숫자, DTO 등)으로 받아 처리합니다.
이때 데이터가 어디에 담겨서 왔는지에 따라 받는 방식이 달라집니다.

- key-value 형태(form, query string): @ModelAttribute 중심
- JSON 형태(body): @RequestBody 중심

## 정의
- @ModelAttribute: 요청 파라미터(key-value)를 객체로 바인딩합니다.
  - GET 쿼리스트링, POST form(application/x-www-form-urlencoded)에서 주로 사용합니다.
- @RequestBody: HTTP Body에 담긴 JSON을 객체로 역직렬화해서 바인딩합니다.
- Query String: URL 뒤의 ?name=Robbie&age=95 같은 형식
- Form 데이터: POST body에 name=Robbie&age=95 같은 형식(Content-Type이 application/x-www-form-urlencoded)
- JSON Body: POST body에 {"name":"Robbie","age":"95"} 같은 형식(Content-Type이 application/json)

## 핵심 개념
### 1) key-value 데이터는 객체로 묶어서 받기 좋다 (@ModelAttribute)
- 데이터가 1~2개면 @RequestParam으로 하나씩 받아도 되지만,
  값이 많아지면 파라미터가 길어지고 유지보수가 어려워집니다.
- 이때 @ModelAttribute로 DTO 하나로 묶어 받는 방식이 깔끔합니다.

### 2) JSON Body는 @RequestBody로 받는다
- 프론트/백엔드 분리 구조에서는 JSON으로 요청하는 경우가 많습니다.
- 이때는 @RequestBody로 “Body의 JSON”을 객체로 받아옵니다.
- 내부적으로 Jackson 같은 라이브러리가 JSON을 객체로 변환합니다.

### 3) @ModelAttribute는 생략될 수 있다
- Spring은 컨트롤러 파라미터에서 어노테이션이 생략돼도 어느 정도 추론합니다.
- 구분 기준(강의 기준)
  - 파라미터 타입이 SimpleValueType이면 @RequestParam으로 간주
  - 아니면 @ModelAttribute로 간주
- SimpleValueType 예시: 원시 타입(int), 래퍼 타입(Integer), Date 등

## 동작 방식
### 1) @ModelAttribute 동작
1) 요청에서 key-value 파라미터를 수집합니다.
2) 객체를 생성합니다.
3) 파라미터 이름과 객체 필드/프로퍼티를 매칭합니다.
4) setter 또는 생성자 등을 통해 값을 주입합니다.

### 2) @RequestBody 동작
1) HTTP Body를 읽습니다.
2) Content-Type이 application/json인지 확인합니다.
3) JSON을 객체로 역직렬화합니다.
4) 변환된 객체를 컨트롤러 파라미터에 주입합니다.

## 주의할 점
- 객체로 데이터를 받을 때, 값이 객체 필드에 채워지려면 아래 중 하나가 필요합니다.
  - setter/getter 같은 프로퍼티 접근 수단
  - 또는 오버로딩 생성자(상황에 따라)
- DTO에 이런 요소가 없으면 “요청은 들어왔는데 값이 안 채워지는” 문제가 생길 수 있습니다.

## 실무 포인트
- “식별자”는 Path Variable, “옵션/필터”는 Request Param, “복잡한 입력”은 RequestBody로 가는 습관을 들이면 API 설계가 안정적입니다.
- @ModelAttribute로 받는 경우, required=false 같은 옵션이 섞이면 null 처리와 타입(원시형 vs 래퍼형)을 신경 써야 합니다.
- JSON 요청인데 값이 null로 들어오면 먼저 확인할 것
  - Content-Type이 application/json인지
  - JSON 필드명과 DTO 필드명이 맞는지
  - DTO에 기본 생성자/프로퍼티 접근 수단이 있는지

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| Query String | URL 뒤에 붙는 key-value 파라미터 |
| Form 데이터 | POST body에 key=value로 담기는 데이터 |
| JSON Body | POST body에 JSON으로 담기는 데이터 |
| @ModelAttribute | key-value를 객체로 바인딩 |
| @RequestBody | JSON Body를 객체로 바인딩 |
| SimpleValueType | Spring이 “단일 값”으로 보는 타입(원시, 래퍼, Date 등) |

## 공부하면서 떠올린 질문
- @ModelAttribute와 @RequestBody를 나누는 기준을 한 문장으로 말하면?
- @ModelAttribute 생략 시, Spring이 타입으로 판단하는 방식의 장단점은?
- JSON인데 @RequestBody 없이 DTO 파라미터로 받으면 어떤 일이 생길까?
- 값이 null로 들어올 때, Content-Type 문제와 DTO 구조 문제를 어떻게 구분할까?
