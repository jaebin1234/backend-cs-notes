# Spring 로컬 캐시 정리

## 1. 로컬 캐시 개념

로컬 캐시는 애플리케이션 인스턴스 내부 메모리에 데이터를 저장해 두고, 같은 데이터를 다시 조회할 때 DB나 외부 API를 거치지 않고 메모리에서 바로 가져오는 방식입니다.

Spring에서는 캐시 추상화(Spring Cache Abstraction)를 통해 로컬 캐시, Redis 캐시, Ehcache, Caffeine 등 다양한 캐시 구현체를 같은 어노테이션 방식으로 사용할 수 있도록 해줍니다.

로컬 캐시의 기본 특징은 다음과 같습니다.

* 서버 인스턴스마다 별도의 메모리 캐시를 가집니다.
* 네트워크 호출 없이 매우 빠르게 조회할 수 있습니다.
* 서버를 재시작하면 캐시가 모두 사라지는 휘발성입니다.
* 여러 인스턴스 간의 캐시 일관성은 자동으로 맞춰지지 않습니다.

## 2. Spring 캐시 추상화 구조

Spring 캐시는 AOP 프록시 기반으로 동작합니다. 흐름을 간단히 정리하면 다음과 같습니다.

1. 클라이언트가 서비스 메서드를 호출합니다.
2. 해당 메서드에 캐시 어노테이션(`@Cacheable` 등)이 붙어 있으면 Spring AOP 프록시가 먼저 가로챕니다.
3. 프록시는 CacheManager로부터 캐시(Cache)를 가져와 key에 해당하는 값이 있는지 조회합니다.
4. 캐시에 값이 있으면 실제 메서드를 호출하지 않고 캐시 값을 바로 반환합니다.
5. 캐시에 값이 없으면 실제 메서드를 실행하고, 그 결과를 캐시에 저장한 뒤 반환합니다.

구성 요소를 개념 위주로 정리하면 다음과 같습니다.

* CacheManager
  여러 개의 캐시 공간을 관리하는 매니저입니다. 예) ConcurrentMapCacheManager, CaffeineCacheManager, RedisCacheManager 등

* Cache
  실제로 key-value 데이터를 저장하는 캐시 공간입니다. 캐시 이름별로 하나의 Cache 인스턴스가 존재합니다.

* 캐시 어노테이션
  메서드 단위로 캐시 정책을 선언하기 위한 어노테이션입니다.
  예) `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`, `@CacheConfig`

* AOP 프록시
  메서드 호출을 가로채 캐시를 먼저 확인하고, 필요할 때만 실제 메서드를 호출하는 역할을 합니다.
  트랜잭션 AOP와 비슷한 구조로 동작한다고 이해하시면 편합니다.

## 3. 기본 로컬 캐시 사용 방법

Spring Boot에서 별도 설정이 없으면 기본적으로 ConcurrentMap 기반의 로컬 캐시를 사용할 수 있습니다. 이해를 위해 설정과 사용 방법을 같이 정리하겠습니다.

### 3.1 의존성

일반 웹 애플리케이션이면 보통 다음과 같이 사용하실 수 있습니다.

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-cache'
}
```

### 3.2 캐시 활성화와 CacheManager 설정

`@EnableCaching` 으로 캐시 기능을 활성화하고, 기본 CacheManager를 설정할 수 있습니다.
ConcurrentMapCacheManager는 JDK ConcurrentMap 기반으로 동작하는 단순 로컬 캐시 구현체입니다.

```java
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        // "users", "products" 라는 이름의 캐시 공간을 만듭니다.
        return new ConcurrentMapCacheManager("users", "products");
    }
}
```

Spring Boot 기본 설정을 그대로 사용해도 되지만, 이렇게 명시해 두면 어떤 캐시들이 있는지 코드에서 바로 확인할 수 있습니다.

### 3.3 주요 캐시 어노테이션

실무에서 주로 사용하는 어노테이션 위주로 정리하겠습니다.

1. `@Cacheable`
   조회 메서드에 붙이며, 처음 호출 결과를 캐시에 저장하고 다음 호출부터는 캐시에서 값을 가져옵니다.

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Cacheable(cacheNames = "users", key = "#userId")
    public User getUser(Long userId) {
        simulateSlowQuery();
        return findUserFromDB(userId);
    }

    private void simulateSlowQuery() {
        try {
            Thread.sleep(1000L);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private User findUserFromDB(Long userId) {
        return new User(userId, "홍길동");
    }
}
```

2. `@CachePut`
   메서드를 항상 실행하면서, 리턴값을 캐시에 무조건 갱신합니다.
   데이터 수정 후 캐시를 갱신하고 싶을 때 사용합니다.

```java
import org.springframework.cache.annotation.CachePut;

@Service
public class UserService {

    @CachePut(cacheNames = "users", key = "#user.id")
    public User updateUser(User user) {
        return updateUserToDB(user);
    }

    private User updateUserToDB(User user) {
        return user;
    }
}
```

3. `@CacheEvict`
   특정 key 또는 캐시 전체를 삭제할 때 사용합니다.

```java
import org.springframework.cache.annotation.CacheEvict;

@Service
public class UserService {

    @CacheEvict(cacheNames = "users", key = "#userId")
    public void deleteUser(Long userId) {
        deleteUserFromDB(userId);
    }

    @CacheEvict(cacheNames = "users", allEntries = true)
    public void clearUserCache() {
        // 로그만 남기는 등 추가 작업을 할 수 있습니다.
    }
}
```

4. `@Caching`
   여러 캐시 어노테이션을 한 번에 묶어서 사용할 때 사용합니다.

```java
import org.springframework.cache.annotation.Caching;

@Service
public class UserService {

    @Caching(
        put = {
            @CachePut(cacheNames = "users", key = "#result.id")
        },
        evict = {
            @CacheEvict(cacheNames = "userList", allEntries = true)
        }
    )
    public User createUser(UserCreateRequest request) {
        return createUserToDB(request);
    }
}
```

5. `@CacheConfig`
   클래스 단위로 공통 캐시 이름 등을 지정할 수 있습니다.

```java
import org.springframework.cache.annotation.CacheConfig;

@Service
@CacheConfig(cacheNames = "users")
public class UserService {

    @Cacheable(key = "#userId")
    public User getUser(Long userId) {
        return findUserFromDB(userId);
    }

    @CacheEvict(key = "#userId")
    public void deleteUser(Long userId) {
        deleteUserFromDB(userId);
    }
}
```

### 3.4 key, 조건, SpEL 사용

Spring 캐시는 SpEL을 사용하여 key, 조건 등을 유연하게 표현할 수 있습니다.

예시


``` java
// 파라미터 값이 0보다 클 때만 캐시
@Cacheable(cacheNames = "products", key = "#productId", condition = "#productId > 0")
public Product getProduct(Long productId) { ... }

// 리턴값이 null 이 아닐 때만 캐시
@Cacheable(cacheNames = "products", key = "#productId", unless = "#result == null")
public Product getProduct(Long productId) { ... }
```


## 4. 로컬 캐시의 장단점

### 4.1 장점

* 네트워크 호출이 없기 때문에 가장 빠른 캐시입니다.
* ConcurrentMapCacheManager는 JDK 컬렉션만 사용하므로 의존성이 거의 없습니다.
* 단일 서버 또는 인스턴스 수가 많지 않은 서비스에서 쉽게 도입할 수 있습니다.
* 특정 서버 인스턴스에만 필요한 데이터 캐싱에 적합합니다.

### 4.2 단점

* 각 인스턴스별로 캐시가 따로 존재하므로 멀티 인스턴스 환경에서 캐시 일관성이 깨질 수 있습니다.
* 서버 재시작 시 캐시가 모두 사라집니다.
* 캐시 용량과 만료 정책을 직접 관리하지 않으면 메모리 사용량이 계속 늘어날 수 있습니다.
* 트래픽이 많은 대규모 분산 환경에서는 같은 데이터를 서버마다 중복해서 들고 있어 비효율적일 수 있습니다.

## 5. 실무 기반 예시

### 5.1 코드 값, 설정 값, 등급 조회

실무에서 로컬 캐시를 가장 많이 사용하는 패턴 중 하나는 변경이 자주 일어나지 않는 코드 값, 설정 값, 등급 정보 등에 대한 캐싱입니다.

예시 상황

* 회원 등급 정보는 하루에 몇 번만 변경되지만
* 조회는 여러 API에서 매우 자주 발생하는 경우

이 때 매번 DB에서 SELECT 하는 대신, 로컬 캐시로 조회하면 성능을 크게 개선할 수 있습니다.

```java
@Service
@CacheConfig(cacheNames = "memberGrades")
public class MemberGradeService {

    @Cacheable(key = "#memberId")
    public Grade findGrade(Long memberId) {
        return findGradeFromDB(memberId);
    }

    @CacheEvict(key = "#memberId")
    public void refreshGrade(Long memberId) {
        // 등급 변경 시 캐시 삭제 → 다음 조회 때 새 값으로 채워지도록 합니다.
    }
}
```

특징

* 조회는 대부분 캐시에서 처리합니다.
* 변경 이벤트 발생 시에만 캐시를 비워 새로 채웁니다.
* 변경 빈도가 낮고, 읽기 비율이 높은 데이터에 적합합니다.

### 5.2 도메인별 캐시 전략 분리

예를 들어, 포인트·정산·결제 등 트랜잭션이 중요한 도메인은 Redis 캐시나 DB 중심으로 설계하고,
코드값, 템플릿, UI 설정, 환경 설정 등은 로컬 캐시로 처리하는 식으로 도메인을 나누어 사용할 수 있습니다.

이렇게 하면

* 핵심 비즈니스 데이터는 일관성을 우선
* 참조용 데이터는 응답 속도와 단순성을 우선

하는 구조로 설계하실 수 있습니다.

## 6. 로컬 캐시 사용 시 유의할 점

1. 멀티 인스턴스 환경에서의 일관성

* 서버가 여러 대일 경우, 한 서버의 캐시만 갱신되고 다른 서버는 이전 값을 그대로 들고 있을 수 있습니다.
* 이 경우 캐시가 필요한 데이터가 비즈니스적으로 중요한 값이라면 Redis 같은 분산 캐시를 검토하셔야 합니다.

2. 캐시에 올리는 대상 선정

* 변경 빈도가 낮고, 읽기 위주인 데이터를 캐시하는 것이 좋습니다.
* 캐시에서 꺼낸 객체를 외부에서 변경하면, 캐시 안의 데이터와 실제 데이터가 섞여버릴 수 있으므로 주의가 필요합니다.

3. AOP 적용 범위

* `@Cacheable` 등의 어노테이션은 프록시 기반이기 때문에

  * 같은 클래스 내부에서 자기 자신의 메서드를 직접 호출하는 경우에는 캐시가 적용되지 않습니다.
  * 일반적으로 public 메서드에 적용하는 것이 기본입니다.
* 이는 트랜잭션 AOP와 동일한 제약입니다.

4. 메모리 사용량과 GC 부담

* ConcurrentMap 기반 로컬 캐시는 기본 TTL, 최대 개수 제한이 없습니다.
* 많은 데이터를 캐시에 올리면 GC 부담과 OOM 위험이 있으므로, 필요하다면 Caffeine 등 eviction 정책이 있는 구현체를 사용하는 것이 좋습니다.

5. 장애 전파 방지

* 캐시 조회/저장 과정에서 예외가 발생하더라도, 캐시 때문에 전체 서비스가 죽지 않도록 설계하시는 것이 좋습니다.
* 캐시에 실패하면 그냥 실제 메서드를 호출해서 처리하도록 fallback 구조를 가져가는 것을 권장드립니다.

## 7. Redis 캐시 등 외부 캐시와 트레이드오프

로컬 캐시와 Redis 같은 분산 캐시는 서로 장단점이 명확합니다. 서비스 특성에 따라 적절히 선택하거나 혼합해서 사용하는 것이 일반적입니다.

### 7.1 로컬 캐시가 유리한 경우

* 단일 서버 또는 소수의 서버에서 운영하는 서비스
* 네트워크 지연 없이 가장 빠른 응답이 필요한 경우
* 인스턴스마다 캐시 데이터가 조금 달라도 크게 문제가 되지 않는 경우
  예: UI 코드값, 일부 설정 값, 공통 템플릿 등

### 7.2 Redis 캐시가 유리한 경우

* 서버 인스턴스가 여러 대이고, 모든 인스턴스가 동일한 캐시 데이터를 공유해야 하는 경우
* 세션, 토큰, 포인트 잔액, 주문 상태 등 일관성이 중요한 데이터를 캐싱할 때
* TTL, eviction, persistence 등 캐시 관리 기능이 필요할 때
* 서버 재시작 후에도 캐시를 일정 수준 유지하고 싶은 경우

### 7.3 트레이드오프 정리

* 성능 vs 일관성

  * 로컬 캐시는 가장 빠르지만, 인스턴스별 값이 달라질 수 있습니다.
  * Redis 캐시는 네트워크 비용이 있지만, 여러 서버가 같은 값을 공유할 수 있습니다.

* 단순성 vs 기능

  * 로컬 캐시는 단순하고 빠르지만, TTL·eviction·복제·내구성 등의 기능이 없습니다.
  * Redis는 인프라와 운영 비용이 들지만, 기능이 풍부합니다.

* 적용 전략

  * 핵심 비즈니스 데이터는 Redis·DB 중심으로 일관성을 우선
  * 참조용·설정성 데이터는 로컬 캐시로 속도와 단순성을 우선
  * 필요하다면 로컬 캐시 → Redis → DB 순서로 계층화된 캐시 전략을 설계할 수도 있습니다.

## 8. 대시보드에서의 캐시 활용

대시보드는 통계·지표·리포트 조회가 많아서 캐시 활용도가 높은 영역입니다. 다만 “어떤 대시보드인지”에 따라 전략이 달라집니다.

### 8.1 대시보드에서 캐시를 사용하는 전형적인 상황

1. 운영·통계 대시보드

* 일간 매출, 월별 가입자 수, 일자별 포인트 사용량, 정산 통계 등
* 쿼리 자체가 무겁고, 초 단위까지 완전 실시간일 필요는 없는 경우가 많습니다.
* 이 경우 보통 다음 조합을 많이 사용합니다.

  * 배치나 스케줄러로 미리 집계해 둔 통계 테이블
  * 그 위에 Redis 캐시나 로컬 캐시를 5초~1분 정도 TTL로 적용

2. 리포트·차트형 대시보드

* 한 화면에 여러 개의 그래프·카드가 동시에 그려지는 경우
* API를 위젯 단위로 분리하고, 각 위젯 API에 캐시를 적용하는 패턴이 많습니다.
  예:

  * `GET /dashboard/summary`
  * `GET /dashboard/top-users`
  * `GET /dashboard/point-usage`
    각각을 따로 캐시하면서 TTL을 다르게 가져갈 수 있습니다.

3. 거의 실시간이 아니어도 되는 지표

* 실시간이라고 해도 대부분 서비스에서 1~5초 정도 지연은 허용되는 경우가 많습니다.
* 이 정도면 Redis 캐시나 로컬 캐시로 충분히 부담을 줄일 수 있습니다.

### 8.2 대시보드 캐시가 까다로워지는 경우

1. 필터 조합이 매우 다양할 때

* 기간, 회사, 지역, 상품, 태그 등 필터 조합이 많아질수록
* 캐시 key 조합이 폭발하고 메모리 낭비가 심해질 수 있습니다.
* 이때는 모든 조합을 캐시하기보다는

  * 자주 쓰는 조합 위주로만 캐시하거나
  * 집계 테이블 자체를 잘 설계해서 쿼리 비용을 줄이는 방식이 필요합니다.

2. 실시간성과 정확도를 동시에 강하게 요구할 때

* 모니터링(에러율, 트래픽, 장애 감시) 대시보드는

  * 보통 Prometheus, 시계열 DB, 별도의 모니터링 스택을 사용합니다.
* 이 영역은 애플리케이션 캐시보다는 수집 구조와 시계열 저장소 설계가 핵심입니다.

3. 데이터 변경이 잦고 캐시 무효화가 복잡할 때

* 어떤 이벤트에서 어떤 캐시를 비워야 할지 복잡해지면 유지 보수가 어려워집니다.
* 이럴 때는 TTL 기반 캐시로 시작하고, 정말 필요한 몇 군데만 선택적으로 무효화하는 전략이 현실적입니다.

### 8.3 실무에서의 무난한 적용 순서

1. 먼저 통계·대시보드용 쿼리와 테이블 구조를 정리합니다.

   * 통계용 별도 테이블, 집계 테이블(Materialized View 느낌) 설계
   * 인덱스·GROUP BY·JOIN 최소화

2. 집계 테이블 위에 대시보드 API를 만들고 실제 응답 시간을 측정합니다.

   * 200~500ms 수준이면 캐시 없이도 버틸 수 있는지 검토
   * 1~2초 이상이고 QPS가 높다면 캐시 도입을 고려

3. API 단위 Redis 캐시 + TTL을 적용합니다.

   * key 예: `dashboard:summary:{companyId}:{dateRange}`
   * TTL은 10초, 30초, 1분 등으로 시작
   * invalidation은 복잡하게 설계하지 말고

     * 주요 데이터 변경 시 전체 캐시 삭제
     * 또는 TTL에만 의존하는 방식으로 단순하게 가져가는 것이 좋습니다.

4. 트래픽과 데이터 종류가 많아지면

   * 지표·로그성 데이터는 별도의 데이터 파이프라인(ETL, 스트리밍)과 시계열 저장소로 분리하는 것을 고려합니다.

정리하면, 대시보드에서는 캐시를 “많이” 사용하지만
처음부터 복잡한 캐시 무효화 정책을 설계하기보다는

* 통계용 집계 테이블 설계
* 위젯·API 단위 Redis 캐시
* 짧은 TTL 중심의 단순한 전략

으로 시작하는 것이 실무에서 가장 현실적인 접근입니다.

## 9. 마무리 정리

* Spring 로컬 캐시는 ConcurrentMapCacheManager 등으로 인스턴스 내부 메모리를 사용하는 캐시입니다.
* `@EnableCaching` 과 `@Cacheable`, `@CachePut`, `@CacheEvict` 등의 어노테이션으로 메서드 단위 캐싱을 선언적으로 관리할 수 있습니다.
* 로컬 캐시는 매우 빠르고 간단하지만, 멀티 인스턴스 환경에서 캐시 일관성 문제가 발생할 수 있습니다.
* Redis 같은 분산 캐시와의 트레이드오프를 고려해, 도메인별로 로컬 캐시와 외부 캐시를 적절히 나누어 사용하는 것이 중요합니다.
* 대시보드 영역에서는 쿼리 비용이 크고 완전 실시간일 필요가 없는 경우가 많아, 집계 테이블 + TTL 캐시 전략이 자주 사용됩니다.
