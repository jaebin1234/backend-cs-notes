# spring-cloud-config.md

## 개요

Spring Cloud Config는 여러 서비스가 쓰는 설정을 한 곳에서 관리하는 방식입니다.
각 서비스가 설정 파일을 따로 들고 있지 않고, 실행할 때 Config Server에서 설정을 받아서 사용합니다.
설정을 바꾸면 서비스 재배포 없이 값만 갱신하는 흐름도 만들 수 있습니다.

---

## Config Server / Config Client

구성은 서버와 클라이언트로 나뉩니다.

* Config Server
  설정 파일을 보관하고 내려주는 역할
  설정 저장소로 Git, 파일 시스템, DB 같은 방식들을 붙일 수 있습니다.

* Config Client
  Config Server에서 설정을 가져와서 사용하는 서비스

---

## 환경별 설정

환경(dev, local, prod)에 따라 다른 설정을 내려줄 수 있습니다.
보통 파일을 분리해서 관리합니다.

예를 들면 이런 식입니다.

* product-service.yml
* product-service-local.yml

그리고 서비스에서는 profiles로 어떤 설정을 받을지 선택합니다.

---

## 설정 갱신

설정이 바뀌었을 때 반영하는 방법은 크게 두 갈래로 이해하면 됩니다.

* 자동 전파 방식
  메시징을 통해 변경을 퍼뜨리는 방식(Spring Cloud Bus 같은 흐름)
* 수동 갱신 방식
  클라이언트에서 /actuator/refresh 호출해서 반영하는 방식

---

## 실습 요약

목표는 “product 서비스가 포트와 message 값을 Config Server에서 받아서 쓰고, message 변경을 갱신하는 것”입니다.

구성은 이렇게 잡습니다.

* Eureka
* Config Server(네이티브 모드)
* Product Service(Config Client)

### 1) Config Server

* config server를 띄우고, 리소스 폴더 안에 config-repo 폴더를 만들어 설정 파일을 둡니다.
* 네이티브 모드라서 설정 파일은 classpath 안에서 읽습니다.
* product-service.yml / product-service-local.yml 같은 파일을 만들어 둡니다.

예시로 product-service-local.yml에는 이런 값이 들어갑니다.

* server.port
* message

### 2) Product Service

* config 의존성을 추가합니다.
* profiles를 local로 두고, config server에서 설정을 가져오게 합니다.
* server.port는 0으로 두고, 실제 포트는 config에서 내려받은 값으로 덮어씁니다.
* message도 기본값을 두고, config에서 내려받은 값으로 바뀌는 걸 확인합니다.

컨트롤러에서 확인하는 값

* server.port
* message

그리고 값 갱신이 되게 하려고 RefreshScope를 사용합니다.

### 3) 실행과 확인

* Eureka 실행
* Config Server 실행
* Product 실행

확인 흐름

1. /product 호출해서 포트와 message가 config에서 온 값인지 확인
2. config server 쪽 설정 파일의 message를 변경
3. config server 재시작(네이티브 모드라서)
4. /actuator/refresh 를 POST로 호출
5. 다시 /product 호출해서 message가 바뀐 걸 확인

추가로, config server에서 제공하는 설정 조회 URL로 현재 설정 값을 확인할 수도 있습니다.

* /{application-name}/{profile}

---

## 용어 정리

| 용어                  | 의미                                |
| ------------------- | --------------------------------- |
| Spring Cloud Config | 중앙에서 설정을 관리하고 내려주는 방식             |
| Config Server       | 설정 제공 서버                          |
| Config Client       | 설정을 받아서 쓰는 서비스                    |
| Profile             | 환경 구분(local, dev, prod 등)         |
| Native Mode         | 로컬 파일(classpath)에서 설정을 읽는 방식      |
| /actuator/refresh   | 설정 값을 다시 읽어오게 하는 엔드포인트            |
| RefreshScope        | refresh 시 빈이 설정 값을 다시 반영하도록 하는 범위 |

---

## 공부하면서 떠올릴 질문

* profiles를 바꾸면 어떤 설정 파일을 우선으로 읽게 될까?
* server.port=0으로 두고 config에서 포트를 받는 방식은 어떤 상황에서 유용할까?
* /actuator/refresh는 모든 설정을 다 바꾸는 걸까, 일부만 바꾸는 걸까?
