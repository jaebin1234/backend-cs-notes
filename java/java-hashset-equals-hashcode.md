# HashSet, equals, hashCode로 좌표 중복 체크 정리

---

## 1. 개요

코딩 테스트에서 좌표나 상태를 `Set`에 넣고 중복 여부를 확인하는 경우가 많습니다.

이때 `int[]` 배열을 그대로 `HashSet<int[]>`에 넣으면, 값이 같아도 `contains`가 제대로 동작하지 않습니다.

이 문서는 다음 내용을 정리합니다.

- HashSet이 동작하는 기본 원리
- int[]를 그대로 Set에 쓰면 안 되는 이유
- 좌표를 Set에 넣는 정석적인 방법 (클래스 / record + equals/hashCode)
- 코딩 테스트에서 빠르게 쓸 수 있는 꼼수 (문자열, 정수 인코딩)

---

## 2. HashSet의 동작 메커니즘

HashSet(및 HashMap)은 내부적으로 두 가지 메서드를 사용합니다.

1. `hashCode()`
    - 객체가 어느 버킷(bucket)에 들어갈지 결정합니다.
    - 같은 해시값을 가진 객체들은 같은 버킷에 모입니다.
2. `equals()`
    - 같은 버킷 안에서 “논리적으로 같은 객체인지” 최종 비교합니다.
    - 즉, 중복 여부를 확정하는 마지막 단계입니다.

따라서 다음 규칙이 꼭 지켜져야 합니다.

- `equals`가 `true`인 두 객체는 반드시 같은 `hashCode`를 가져야 합니다.
- `hashCode`와 `equals`는 항상 한 쌍으로 설계해야 합니다.
- 이 규칙을 지키지 않으면 `contains`, `remove`, 중복 체크가 의도대로 동작하지 않을 수 있습니다.

---

## 3. int[]를 HashSet에 쓰면 안 되는 이유

배열 타입(`int[]`, `char[]` 등)은 `equals`를 재정의하지 않고, `Object`의 기본 구현을 그대로 사용합니다.

즉, “내용 비교”가 아니라 “참조(주소) 비교”를 합니다.

``` java
int[] a = {1, 2};
int[] b = {1, 2};

System.out.println(a.equals(b)); // false
System.out.println(a == b);      // false

```

위 코드에서 `a`와 `b`는 값은 같지만 서로 다른 배열 인스턴스이기 때문에

`equals`도, `==`도 모두 `false`입니다.

Set에 넣어보면 문제가 더 분명해집니다.

``` java
Set<int[]> set = new HashSet<>();
set.add(new int[]{1, 2});

System.out.println(set.contains(new int[]{1, 2})); // 거의 항상 false

```

- 우리가 원하는 것은 “좌표 (1, 2)가 이미 있는지”를 값 기준으로 확인하는 것입니다.
- 하지만 `HashSet` 입장에서는 두 배열이 서로 다른 참조이기 때문에
    
    “완전히 다른 객체”로 인식하고, `contains`가 `false`가 됩니다.
    

배열은 값 비교용이 아니라 “그냥 객체”로 취급된다고 보시면 됩니다.

---

## 4. 좌표를 Set에 넣는 정석적인 방법

좌표를 값 기준으로 정확하게 비교하려면

좌표를 표현하는 전용 값 객체를 만들고, `equals`와 `hashCode`를 재정의해서 사용하시는 것이 정석입니다.

### 4.1. 클래스로 구현하는 방법

``` java
import java.util.Objects;

public class Pos {
    int x;
    int y;

    public Pos(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                           // 동일 객체
        if (o == null || getClass() != o.getClass()) return false;

        Pos pos = (Pos) o;
        return x == pos.x && y == pos.y;                      // 좌표 값 비교
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);                            // x, y 기반 해시
    }
}

```

사용 예시는 다음과 같습니다.

``` java
import java.util.HashSet;
import java.util.Set;

public class Example {
    public static void main(String[] args) {
        Set<Pos> set = new HashSet<>();

        set.add(new Pos(1, 2));
        set.add(new Pos(1, 2)); // 중복 → 추가되지 않음

        System.out.println(set.size());                      // 1
        System.out.println(set.contains(new Pos(1, 2)));     // true
    }
}

```

이제 `HashSet`은 “좌표 값 (x, y)이 같은지” 기준으로 원소를 비교합니다.

### 4.2. record를 사용하는 방법 (Java 16+)

Java 16 이상에서는 record를 사용하면

생성자, `equals`, `hashCode`, `toString`이 자동으로 생성됩니다.

``` java
public record Pos(int x, int y) { }

```

사용 예시:

``` java
import java.util.HashSet;
import java.util.Set;

public class Example {
    public static void main(String[] args) {
        Set<Pos> set = new HashSet<>();

        set.add(new Pos(1, 2));
        System.out.println(set.contains(new Pos(1, 2)));  // true
    }
}

```

record는 값 객체를 짧게 정의할 수 있어서,

코딩 테스트나 사이드 프로젝트에서 좌표, 상태 등을 표현할 때 매우 편리합니다.

---

## 5. 코딩 테스트에서 쓸 수 있는 예시

클래스나 record를 만드는 것이 가장 깔끔하지만,

시간이 부족한 코딩 테스트에서는 “빠르게” 구현하는 것도 중요합니다.

그럴 때 사용할 수 있는 방법 두 가지입니다.

### 5.1. 문자열로 좌표를 묶어서 저장

```
Set<String> visited = new HashSet<>();

String key = x + "," + y;
if (visited.contains(key)) {
    // 이미 방문한 좌표
} else {
    visited.add(key);
}

```

장점

- 구현이 매우 단순합니다.
- `equals`, `hashCode`를 직접 구현할 필요가 없습니다.

단점

- 문자열 생성 비용이 추가로 발생합니다.
- 좌표 개수가 매우 많을 때는 성능에 영향을 줄 수 있습니다.

### 5.2. 하나의 정수로 인코딩 (좌표 범위가 제한된 경우)

좌표 범위가 제한돼 있을 때는 두 좌표를 하나의 정수로 합칠 수 있습니다.

예를 들어 `0 <= x, y <= 100000` 라고 가정하면:


``` java
int key = x * 100_001 + y;   // y의 최대값보다 큰 수로 곱

Set<Integer> visited = new HashSet<>();

if (visited.contains(key)) {
    // 중복 좌표
} else {
    visited.add(key);
}

```

주의사항

- `x * 기준값 + y` 연산에서 `int` 범위를 넘지 않는지 꼭 확인하셔야 합니다.
- 기준값은 “y의 최대값보다 큰 값”으로 설정해야 충돌이 나지 않습니다.
- 좌표 범위를 알고 있을 때만 안전하게 사용할 수 있는 기법입니다.

---

## 6. equals와 hashCode 재정의 시 주의사항

정리해서 기억하시면 좋습니다.

1. `equals`가 `true`인 두 객체는 반드시 같은 `hashCode`를 가져야 합니다.
2. `hashCode`가 같다고 해서 반드시 `equals`가 `true`일 필요는 없습니다.
    
    해시 충돌은 허용되며, 이 경우 HashSet 내부에서 `equals`로 최종 비교합니다.
    
3. HashSet, HashMap의 원소나 키로 사용할 객체를 만들 때는
    
    `equals`와 `hashCode`를 항상 함께 고려해야 합니다.
    
4. 배열(`int[]`, `char[]` 등)은 `equals`가 참조 기준이므로,
    
    내용 기준 비교가 필요하다면 전용 클래스/record 또는 문자열, 정수 인코딩 등을 사용하셔야 합니다.
    

---

## 7. 요약

- HashSet은 `hashCode`로 버킷을 결정하고, 그 안에서 `equals`로 같은 객체인지 최종 비교합니다.
- `int[]`를 HashSet에 넣으면 참조 기준으로만 비교해서,
    
    값이 같은 좌표라도 `contains`가 `false`가 됩니다.
    
- 좌표 중복 체크를 제대로 하려면
    - 전용 좌표 클래스나 record를 만들고
    - `equals`와 `hashCode`를 값 기준으로 구현해서
    - `HashSet<Pos>` 형태로 사용하는 것이 가장 안전합니다.
- 코딩 테스트에서는 상황에 따라
    - 문자열 키(`"x,y"`)나
    - 정수 인코딩(`x * K + y`)
        
        같은 꼼수를 활용해서 구현 속도를 높일 수도 있습니다.
        

---

## 8. 키워드 / 용어 정리

| 키워드 | 설명 |
| --- | --- |
| HashSet | 해시 기반 Set 구현체. `hashCode`와 `equals`를 이용해 중복 없는 집합을 관리합니다. |
| hashCode | 객체를 정수 해시값으로 변환하는 메서드. 어떤 버킷에 들어갈지 결정할 때 사용됩니다. |
| equals | 두 객체가 논리적으로 같은지 비교하는 메서드. HashSet 안에서 최종 중복 여부를 판단할 때 사용됩니다. |
| 버킷(bucket) | 같은 해시값을 가진 객체들이 모여 있는 공간입니다. HashSet은 해시값별로 버킷을 나눕니다. |
| 참조 비교 | 두 참조(주소)가 같은지 비교하는 방식. `==` 연산이나 배열의 기본 `equals`가 사용하는 방식입니다. |
| 값 비교 | 필드 값이 같은지 비교하는 방식. 좌표(x, y) 같이 실제 데이터 기준으로 비교합니다. |
| 값 객체(Value Object) | 여러 필드로 이뤄진 “값”을 표현하는 객체. 좌표, Money, 기간 등과 같이 동일성보다 값이 중요한 객체입니다. |
| record | Java 16+에서 제공하는 값 객체 전용 문법. 자동으로 `equals`, `hashCode`, `toString`을 생성합니다. |
| 인코딩(encoding) | 여러 값을 하나로 합쳐 표현하는 기법. 예를 들어 `(x, y)`를 `x * K + y` 하나의 정수로 바꾸는 것을 의미합니다. |
| 해시 충돌(collision) | 서로 다른 두 객체가 같은 `hashCode`를 가지는 상황입니다. 이 경우 HashSet은 같은 버킷 안에서 `equals`로 다시 비교합니다. |
| 배열(int[] 등) | 기본적으로 참조 기준 `equals`를 사용합니다. 내용 비교가 필요하면 별도 값 객체나 다른 표현을 사용해야 합니다. |

---