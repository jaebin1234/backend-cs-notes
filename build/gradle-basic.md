# Gradle Basic

## 개요
Gradle은 Java(Spring Boot) 프로젝트를 빌드하고, 필요한 라이브러리를 자동으로 내려받아 연결해주는 빌드 도구입니다. 프로젝트를 실행 가능한 결과물(jar 등)로 만들기 위한 과정과 의존성 관리가 핵심입니다.

## 정의
- Gradle: 빌드 자동화 및 의존성 관리 도구
- Build: 소스 코드를 실행 가능한 결과물로 만드는 과정
- build.gradle: 빌드 설정과 의존성 설정을 적는 파일(Groovy 또는 Kotlin DSL)

## 핵심 메커니즘(핵심 개념)
1) 빌드 스크립트 기반 자동화  
build.gradle에 적힌 설정을 기준으로 빌드 작업을 자동으로 수행합니다.

2) 의존성 자동 다운로드  
dependencies에 라이브러리를 적으면 Gradle이 저장소에서 자동으로 다운로드합니다.

3) 의존성 전이(Transitive)  
내가 추가한 라이브러리가 내부적으로 필요로 하는 라이브러리도 함께 내려받습니다.

## 동작 방식
- build.gradle에 라이브러리를 추가합니다.
- IDE에서 Gradle 동기화가 일어나며 라이브러리를 다운로드합니다.
- 내려받은 라이브러리는 IDE의 External Libraries에서 확인할 수 있습니다.

## 전체 흐름
1) plugins, 버전 설정 확인  
2) repositories 설정 확인(예: mavenCentral)  
3) dependencies에 필요한 라이브러리 추가  
4) Gradle sync로 다운로드 및 적용  
5) 빌드 실행, 결과물 생성(jar 등)

## 실무에서 고려할 포인트
- 강의 환경과 내 로컬 환경의 JDK, Spring Boot, Gradle 버전이 다르면 빌드나 실행 오류가 날 수 있으니 버전 호환성을 유의합니다.
- 라이브러리는 지금 필요한 것만 최소로 추가하는 습관이 좋습니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| Gradle | 빌드 자동화 및 의존성 관리 도구 |
| Build | 소스 코드를 실행 가능한 결과물로 만드는 과정 |
| build.gradle | Gradle 빌드 스크립트 파일 |
| dependency(의존성) | 프로젝트가 가져와서 쓰는 외부 라이브러리 |
| repository | 라이브러리를 내려받는 저장소(예: mavenCentral) |
| External Libraries | IDE에서 내려받은 라이브러리 목록 |

## 공부하면서 떠올릴 질문
- implementation과 testImplementation은 왜 나뉘어 있을까?
- 의존성 전이 때문에 원치 않는 라이브러리가 같이 딸려오면 어떻게 확인할까?
- Gradle sync가 실패하면 보통 어떤 원인이 많을까?
