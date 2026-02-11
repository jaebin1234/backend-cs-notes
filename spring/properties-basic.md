# application.properties Basic

## 개요
application.properties는 스프링 부트 애플리케이션의 설정 값을 모아두는 파일입니다.
서버 포트, DB 연결, 로그 레벨 같은 “실행 환경”을 코드가 아니라 설정으로 바꿀 수 있습니다.

## 정의
- application.properties: key=value 형태로 설정을 적는 파일
- 설정 값(Property): 스프링 부트가 실행될 때 참고하는 값
- 기본값: 스프링 부트가 자동으로 잡아주는 기본 설정(예: 기본 포트 8080)

## 핵심 개념
1) 설정은 코드보다 먼저 적용된다  
코드를 수정하지 않아도 properties 값만 바꾸면 서버 동작이 바뀝니다.

2) 스프링 부트는 기본값이 있고, 내가 적으면 덮어쓴다  
예를 들어 기본 포트가 8080이라도 `server.port=8081`을 적으면 8081로 실행됩니다.

3) key는 “어떤 설정을 바꾸는지”, value는 “바꿀 값”이다  
- key: server.port, spring.datasource.url 같은 이름
- value: 8081, jdbc:... 같은 값

## 동작 방식
1) 애플리케이션 실행
2) 스프링 부트가 application.properties를 읽음
3) 읽은 설정을 서버/스프링 내부 구성에 반영
4) 반영된 설정대로 서버가 실행됨

## 자주 보는 설정 예시(기초)
### 서버
- server.port=8081  
  서버 포트를 8081로 변경

### DB 연결(예시)
- spring.datasource.url=jdbc:postgresql://localhost:5432/testdb
- spring.datasource.username=test
- spring.datasource.password=1234

### 로그(예시)
- logging.level.root=INFO
- logging.level.org.springframework.web=DEBUG

## 실무에서 고려할 포인트
- 설정을 바꾸면 서버 동작이 바로 바뀌니, 변경 후 실행 확인이 중요합니다.
- DB 비밀번호 같은 값은 깃허브에 그대로 올리지 않도록 주의합니다.
- 값 오타가 나면 “설정이 적용이 안 됐는데도 실행은 되는” 경우가 있어서, 동작으로 확인하는 습관이 필요합니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| key=value | 설정을 적는 기본 형태 |
| server.port | 서버 포트 설정 키 |
| spring.datasource.* | DB 연결 관련 설정 키 |
| logging.level.* | 로그 레벨 설정 키 |

## 공부하면서 떠올린 질문
- server.port를 바꾸면 브라우저에서 접속 주소는 어떻게 바뀔까?
- properties에 비밀번호를 적는 건 왜 위험할까?
- 설정을 바꿨는데 적용이 안 된 것 같으면 어디부터 확인할까?
