---
title: 김영한 자바 입문 메서드
description: 김영한 자바 입문 메서드
categories:
- Java
- 김영한 자바 입문

tags:
- Java
- Method

---

> 자바 메서드에 관해 알아보자.

<!-- more -->

## 메서드 정의
```java
public static int add(int a, int b) {
  // 메서드 본문, 실행 코드
}

제어자 반환타입 메서드이름(매개변수 목록) {
  메서드 본문
}
```
자바에서는 함수를 메서드(Method)라 한다. 메서드도 함수의 한 종류라고 생각하면 된다. 지금은 둘을 구분하지 않고, 이정도만 알아두자.

> 메서드는 크게 메서드 선언과 메서드 본문으로 나눌 수 있다.

### 메서드 선언(Method Declaration)
`public static int add(int a, int b)` 부분에서 반환타입, 메서드 이름, 매개변수 목록들을 메서드 선언부분이라 한다. 메서드 선언 정보를 통해 다른 곳에서 해당 메서드를 호출할 수 있다.

### 메서드 본문(Method Body)
```java
{
  int sum = a + b;
  return sum;
}
```
- 메서드가 수행해야 하는 코드블록이다.
- 메서드 본문은 **블랙박스이다.** 메서드를 호출하는 곳에서는 메서드 선언은 알지만 본문은 모른다.
- 메서드 실행 결과를 반환하려면 `return`문을 사용해야 한다.

## 메서드 호출과 용어 정리
메서드를 호출할 때는 메서드에 넘기는 매개변수(파라미터)의 타입과 순서, 개수도 일치해야 한다.
```java
호출: call("hello", 20);
메서드 정의: int call(String str, int age)
```

### 인수(argument)
여기서 `"hello"`, `20`처럼 넘기는 값을 영어로 **Argument**라고 한다. 실무에서는 아규먼트, 인수, 인자라는 용어를 모두 사용한다.


### 매개변수(parameter)
메서드를 정의할 때 선언한 변수인 `String str`, `int age`를 매개변수 또는 파라미터라 한다. 메서드를 호출할 때 인수를 넘기면, 그 인수가 매개변수에 대입된다.

> IntelliJ 단축키 tip
>
> 메서드명에 커서를 올려놓고 `cmd + B`를 입력하면 해당 메서드 선언 부분으로 이동한다.


## 메서드 호출과 값 전달
> ⭐️ 자바는 원시형과 참조형 관계없이 항상 변수의 값을 복사해서 대입한다.
이 대원칙은 반드시 숙지해야 복잡한 상황에도 코드를 단순하게 이해할 수 있다.

```java
public class MethodValue1 {

    public static void main(String[] args) {
        int number = 5;
        System.out.println("1. changeNumber 호출 전, number: " + number);
        changeNumber(number);
        System.out.println("4. changeNumber 호출 후, number: " + number);
    }

    public static void changeNumber(int number) {
        System.out.println("2. changeNumber 변경 전, number: " + number);
        number = number * 2;
        System.out.println("3. changeNumber 변경 후, number: " + number);
    }
}
```
```java
1. changeNumber 호출 전, number: 5
2. changeNumber 변경 전, number: 5
3. changeNumber 변경 후, number: 10
4. changeNumber 호출 후, number: 5
```
`main()` 내부의 `number`변수의 값을 읽고 복사해서 `changeNumber` 메서드의 매개변수에 전달해준다. 복사해서 전달해주었기 때문에, `changeNumber` 내부의 `number`변화는 `main()` 내부의 `number` 에 영향을 주지 않는다. 즉, 각 메서드 안에서 사용하는 `number`변수는 서로 완전히 분리된 다른 변수이다. 이름이 같아도 완전히 다른 변수다.

## 메서드 형변환
메서드를 호출할 때 인수와 매개변수 사이에서 명시적 형변환 또는 자동 형변환이 일어난다.

### 명시적 형변환
큰 범위에서 작은 범위로 인수를 전달하여 메서드를 호출할 때는 **명시적 형변환**이 필요하다. 그렇지 않으면 컴파일 오류가 발생한다.
```java
public class MethodCasting1 {

    public static void main(String[] args) {
        double number = 1.5;
        //printNumber(number); //double을 int에 대입하므로 컴파일 오류
        printNumber((int) number); //명시적 형변환을 사용해 double을 int로 변환
    }

    public static void printNumber(int n) {
        System.out.println("숫자: " + n); // 숫자: 1
    }
}
```

### 자동 형변환
작은 범위에서 큰 범위로 인수를 전달하여 메서드를 호출할 때는 자동 형변환이 일어난다. **단, 타입이 달라도 자동 형변환이 가능한 경우에 호출할 수 있다.**
```java
public class MethodCasting2 {

    public static void main(String[] args) {
        int number = 100;
        printNumber(number);
    }

    public static void printNumber(double n) {
        System.out.println("숫자: " + n); // 숫자: 100.0
    }
}
```

## 메서드 오버로딩

> 메서드 오버로딩이란?
>
> 같은 이름의 메서드를 여러 개 정의하되 매개변수의 유형, 개수 또는 순서가 다른 경우를 가리킨다.

```java
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }
}
```

> 참고로 반환 타입은 오버로딩 규칙으로 인정하지 않는다.

**오버로딩 실패**
```java
int add(int a, int b)
double add(int a, int b)
```

### 메서드 시그니처(method signature)
`메서드 시그니처 = 메서드 이름 + 매개변수 타입(순서)`
메서든 시그니처는 자바에서 메서드를 구분할 수 있는 고유한 식별자나 서명을 뜻한다. 메서드 시그니처는 메서드의 이름과 매개변수 타입(순서 포함)으로 구성되어 있다.

자바 입장에서 각각의 메서드를 고유하게 식별할 수 있어야 어떤 메서드를 호출할 지 결정할 수 있다.

> 먼저 전달된 인수 타입에 최대한 맞는 메서드를 찾아서 실행하고, 그래도 없으면 형 변환 가능한 타입의 메서드를 찾아서 실행한다.

---
