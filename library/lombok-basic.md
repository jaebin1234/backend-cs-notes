# Lombok Basic

## 개요
Lombok은 자바에서 Getter, Setter, 생성자 같은 반복 코드를 자동으로 만들어주는 라이브러리입니다.

## 정의
- Lombok: 반복 코드를 줄여주는 라이브러리
- Annotation Processing: 컴파일 시점에 코드가 자동 생성되도록 처리하는 기능

## 핵심 개념
- 어노테이션을 붙이면 컴파일 시점에 메서드/생성자가 자동 생성됩니다.
- 소스에 안 보여도 실제 클래스에는 생성된 코드가 들어갑니다.

## 동작 방식
1) Lombok 의존성 추가
2) IDE에서 Annotation Processing 활성화
3) 어노테이션 적용
4) 컴파일 시 자동 코드 생성

## 자주 쓰는 어노테이션(기초)
- @Getter / @Setter: getter, setter 생성
- @ToString: toString 생성
- @EqualsAndHashCode: equals, hashCode 생성
- @NoArgsConstructor: 기본 생성자 생성
- @AllArgsConstructor: 전체 필드 생성자 생성
- @RequiredArgsConstructor: final 또는 필수 필드 생성자 생성
- @Builder: 빌더 패턴 코드 생성
- @Slf4j: log 변수 자동 생성(로깅)

## 실무 포인트
- @Setter를 클래스 전체에 열어두면 상태 변경 지점이 늘어 유지보수가 어려울 수 있습니다.
- @ToString은 객체 안에 민감 정보가 있으면 로그로 노출될 수 있어 주의합니다.
- IDE 설정(Annotation Processing)이 꺼져 있으면 “메서드가 없다고” 에러가 날 수 있습니다.

## 용어 정리
| 용어 | 뜻 |
| --- | --- |
| @Getter | get 메서드 자동 생성 |
| @Setter | set 메서드 자동 생성 |
| @Builder | 빌더 패턴 코드 자동 생성 |
| @Slf4j | Logger 변수 자동 생성 |
| Annotation Processing | 컴파일 단계 코드 생성 처리 |

## 공부하면서 떠올린 질문
- @AllArgsConstructor와 @RequiredArgsConstructor는 언제 구분해서 쓸까?
- @Setter를 엔티티에 열어두면 어떤 문제가 생길까?
- @Builder는 생성자랑 비교해서 뭐가 편할까?
- @ToString이 자동 생성되면 로그에 어떤 정보가 찍힐까?
