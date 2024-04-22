---
title: 김영한 자바 입문 스코프
description: 김영한 자바 입문 스코프
categories:
- Java

tags:
- Java
- scope

---

> 변수는 선언한 위치에 따라 지역 변수, 멤버 변수(클래스 변수, 인스턴스 변수)와 같이 분류된다. 이번에는 지역 변수(Local Variable, 로컬 변수)와 스코프에 관해서 정리해보자.

<!-- more -->

## 지역 변수

지역 변수의 `지역`은 바로 변수가 선언된 코드 블록 `{}`이다. 지역 변수는 자신이 선언된 코드 블록 `{}` 안에서만 생존할 수 있고, 자신이 선언된 코드 블록을 벗어나면 제거된다.

## Scope
변수의 접근 가능한 범위를 스코프(scope)라고 한다. 즉, 변수가 선언된 코드 블록의 범위를 가리킨다. `{}`

**EX**
```java
public class Scope1 {
    public static void main(String[] args) {
        int m = 10; // 변수 m 생성

        if (true) {
            int x = 20; // 변수 x 생성
            System.out.println("x = " + x);
            System.out.println("m = " + m);
        } // x 제거
        // x 접근 불가능
        System.out.println("m = " + m);
    } // m 제거
}
```

## ⭐️ 스코프 존재 이유

> 변수를 선언한 시점부터 변수를 계속 사용할 수 있게 해도 되지 않을까? 왜 복잡하게 접근 범위(스코프)라는 개념을 만들었을까?

아래 예제 코드를 보고 이유를 생각해보자.

**Bad Example**
```java
package scope;
public class Scope2 {
     public static void main(String[] args) {
         int m = 10;
         int temp = 0;
         if (m > 0) {
						 temp = m * 2;
             System.out.println("temp = " + temp);
         }
         System.out.println("m = " + m);
     }
}
```

위 코드는 좋은 코드라고 보기 어렵다. 왜냐하면 임시 변수 `temp` 는 `if` 조건절에서만 임시로 잠깐 사용하는 변수이다. 그런데 `temp` 변수는 `main()` 코드 블록에 선언되어 있다. 이렇게 되면 다음과 같은 **문제**가 발생한다.

- **비효율적인 메모리 사용**: `main()` 코드 블록이 종료될 때 까지 `temp` 변수는 메모리에 유지되는 비효율적인 메모리 사용이 발생한다.
- **코드 복잡성 증가**: `if`절이 끝나더라도 여전히 `temp`에 접근 가능하다. 예제 코드는 단순하지만, 실무 코드는 매우 복잡하기 때문에 접근 가능한 `temp` 변수를 계속 신경을 써야해서 복잡성이 증가한다.

**Good Example**
```java
 package scope;
 public class Scope3 {
     public static void main(String[] args) {
         int m = 10;
         if (m > 0) {
             int temp = m * 2;
             System.out.println("temp = " + temp);
         }
         System.out.println("m = " + m);
     }
}
```
`temp` 를 `if` 코드 블록 안에서 선언했다. 이제 `temp` 는 `if` 코드 블록 안으로 스코프가 줄어든다. 덕분에 `temp` 변수를 메모리에서 빨리 제거하여 메모리를 효율적으로 사용하고, `temp` 변수를 생각해야 하는 범위를 줄여서 더 유지보수 하기 좋은 코드로 만들었다.


## while문 vs for문 with 스코프 관점

스코프 관점에서 `while`문과 `for`문을 비교해보자.

**while**
```java
 package loop;
 public class While {
     public static void main(String[] args) {
         int sum = 0;
         int endNum = 3;
         int i = 1;
         while (i <= endNum) {
             sum = sum + i;
             System.out.println("i=" + i + " sum=" + sum);
             i++;
         }
         // ... 아래 더 많은 코드들이 있다고 가정
     }
}
```

**for**
```java
package loop;
 public class For {
     public static void main(String[] args) {
         int sum = 0;
         int endNum = 3;
         for (int i = 1; i <= endNum; i++) {
             sum = sum + i;
             System.out.println("i=" + i + " sum=" + sum);
         }
				//... 아래에 더 많은 코드들이 있다고 가정 
		}
}
```

스코프 관점에서 카운터 변수 `i`를 비교해보자.

- `while`문의 경우 변수 `i`의 스코프가 `main()` 메서드 전체가 된다. 반면에 `for`문의 경우 변수 `i`의 스코프가 `for`문 안으로 한정된다.
- 따라서 `for`문의 `i`와 같이 반복문 안에서만 사용되는 카운터 변수가 있다면 `while`문 보다 `for`문을 사용해서 **스코프의 범위를 제한하는 것이 메모리 사용과 유지보수 관점에서 더 좋다.**

## 정리
- 변수는 꼭 필요한 범위로 한정해서 사용하는 것이 좋다. **변수의 스코프는 꼭 필요한 곳으로 한정해서 사용하자.** **메모리를 효율적**으로 사용하고 더 **유지보수하기 좋은 코드**를 만들 수 있다.

<br>

> 김영한님 말씀
>
> 좋은 프로그램은 무한한 자유가 있는 프로그램이 아니라 적절한 제약이 있는 프로그램이다.


---
