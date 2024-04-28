---
title: 김영한 자바 입문 향상된 for문
description: 김영한 자바 입문 향상된 for문
categories:
- Java
- 김영한 자바 입문

tags:
- Java
- Loop Statement

---

> 앞서 [반복문](/java/김영한%20자바%20입문/2024/04/21/Java_loop/)에서 설명하지 않은 내용인 향상된 for문(Enhanced For Loop)에 대해 알아보자. 향상된 for문을 살펴보기 전에 [배열](/java/김영한%20자바%20입문/2024/04/28/Java_array_1D/)에 대한 이해가 우선되어야 한다.

<!-- more -->


`향상된 for문` 또는 `for-each문`은 다음과 같이 정의하여 사용한다.
```java
// 향상된 for문 정의
for (변수 : 배열 또는 컬렉션) {
	// 배열 또는 컬렉션의 요소를 순회하면서 수행할 작업
}
```

다음 향상된 for문 사용 예제를 살펴보자.
```java
public class EnhancedFor1 {

    public static void main(String[] args) {
        int[] numbers = {1, 2, 3, 4, 5};

        //일반 for문
        for (int i = 0; i < numbers.length; i++) {
            int number = numbers[i];
            System.out.println(number);
        }

        //향상된 for문, for-each문
        for (int number : numbers) {
            System.out.println(number);
        }

        //for-each문을 사용할 수 없는 경우, 증가하는 index 값 필요
        for (int i = 0; i < numbers.length; i++) {
            System.out.println("number" + i + "번의 결과는: " + numbers[i]);
        }

    }
}
```

### 향상된 for문 장점

실무에서 배열은 처음부터 끝까지 순서대로 읽어서 사용하는 경우가 많다. 그런데 일반 for문 사용 할 경우 배열의 값을 읽기 위해 `int i` 와 같은 인덱스 변수를 선언해야 한다. 그리고 `i < numbers.length` 와 같이 배열의 끝 조건을 지정해주어야 한다. **개발자 입장에서는 그냥 배열을 순서대로 처음부터 끝까지 탐색하고 싶은데 너무 번잡한 일을 해주어야 한다. 그래서 향상된 for문이 등장했다.**

> 향상된 for문은 배열의 인덱스를 사용하지 않고도 배열의 요소를 순회할 수 있기 때문에 코드가 간결하고 가독성이 좋다.


### 향상된 for문을 사용하지 못하는 경우

향상된 for문은 인덱스 값이 감추어져 있기 때문에 사용하지 못하는 경우가 있다. 즉, `int i`와 같은 인덱스 값을 직접 사용해야 하는 경우에는 향상된 for문을 사용할 수 없다.

```java
//for-each문을 사용할 수 없는 경우, 증가하는 index 값 필요
for (int i = 0; i < numbers.length; i++) {
    System.out.println("number" + i + "번의 결과는: " + numbers[i]);
}
```

**단축키 tip**
> `iter`: 직전에 선언된 배열 또는 컬렉션을 가지고 향상된 for문을 자동완성 시켜준다.



---
