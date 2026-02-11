# controller-annotation-basic

## 개요
컨트롤러 메서드는 “요청 URL, 쿼리스트링, 폼 바디” 같은 다양한 형태로 들어오는 데이터를 받아야 합니다.
Spring MVC는 어노테이션으로 “어디에 들어있는 데이터인지”를 명확히 지정해서 파라미터로 꺼내 쓸 수 있게 합니다.

## 정의
- @PathVariable: URL 경로(path) 안에 포함된 값을 꺼내서 파라미터로 받습니다.
- @RequestParam: 쿼리스트링(query string) 또는 폼 데이터(application/x-www-form-urlencoded)의 key-value 값을 파라미터로 받습니다.
- required 옵션: 값이 없을 때 에러를 낼지, 허용할지 결정합니다.

## 핵심 개념
### 1) 클라이언트가 데이터를 보내는 대표 방식 2가지
- Path Variable 방식
  - URL 경로 자체에 값을 포함해서 전송합니다.
  - 보통 “리소스 식별”에 어울립니다.
- Request Param 방식
  - URL 끝에 ?key=value 형태로 전송합니다(쿼리스트링).
  - 또는 form POST에서 body에 key=value 형태로 전송됩니다.
  - 보통 “검색 조건, 옵션, 필터”에 어울립니다.

### 2) @PathVariable
- URL 경로의 특정 위치에 있는 값을 변수로 받습니다.
- 경로 템플릿에서 선언한 이름과 파라미터 이름(또는 지정한 이름)이 매칭됩니다.
- required=false 옵션도 존재하며, 값이 없으면 null이 될 수 있습니다(타입 주의).

### 3) @RequestParam
- key-value로 넘어온 값을 파라미터로 받습니다.
- GET 쿼리스트링과 POST 폼 데이터(application/x-www-form-urlencoded) 둘 다 같은 방식으로 받을 수 있습니다.
- required=false로 두면 해당 key가 없어도 에러가 나지 않고 null로 들어올 수 있습니다(타입 주의).
- 실무에선 “필수/선택 파라미터” 구분을 이 옵션과 함께 설계합니다.

## 동작 방식
1) 요청이 들어오면 Spring이 컨트롤러 메서드 파라미터를 분석합니다.
2) @PathVariable은 URL 경로에서 값을 추출합니다.
3) @RequestParam은 쿼리스트링 또는 폼 바디에서 key 기준으로 값을 추출합니다.
4) 문자열을 숫자 등으로 타입 변환해서 파라미터에 넣습니다.
5) required=true(기본)인데 값이 없으면 오류가 납니다.
6) required=false면 값이 없을 때 null로 들어갈 수 있습니다.

## 실무 포인트
- Path Variable은 “식별자”에, Request Param은 “옵션/조건”에 쓰는 습관을 들이면 URL 설계가 깔끔해집니다.
- required=false를 쓰면 null 가능성이 생기므로,
  - 타입을 원시형(int)보다 래퍼(Integer)로 받는 게 안전한 경우가 많습니다.
- 폼 POST(application/x-www-form-urlencoded)도 결국 key-value라서 @RequestParam으로 받을 수 있습니다.
- 값이 없을 때 기본값이 필요하면 required=false와 함께 기본값 전략을 별도로 세우는 게 좋습니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| Path Variable | URL 경로에 포함된 값 |
| Query String | URL 뒤에 붙는 ?key=value 형태 |
| Form Data | POST에서 body로 보내는 key=value 형태 |
| required | 파라미터 필수 여부 옵션 |

## 공부하면서 떠올린 질문
- Path Variable과 Request Param을 나누는 기준을 한 문장으로 말하면?
- required=false를 켰을 때 “타입(int vs Integer)”은 왜 중요할까?
- GET 쿼리스트링과 POST 폼 데이터가 @RequestParam으로 같은 방식으로 처리되는 이유는 뭘까?
