---
title: 김영한 자바 입문 형변환
description: 김영한 자바 입문 형변환
categories:
- Java

tags:
- Java
- Casting

---

> Java 형변환이 언제 어떻게 발생하는지 알아보자.

<!-- more -->

## 대입과 형변환

- 작은 범위에서 큰 범위로는 당연히 값을 넣을 수 있다.
  - `int` -> `long` -> `double`
- 큰 범위에서 작은 범위로는 다음과 같은 문제가 발생한다.
  - 오버플로우
  - 소수점 버림


> Java는 작은 범위에서 큰 범위로 대입할 때 **자동 형변환**이 일어난다.

### 자동 형변환

작은 범위 숫자 타입에서 큰 범위 숫자 타입으로 대입할 때 개발자가 직접 형변환을 하지 않아도 자동으로 형변환이 일어난다. 이것을 **자동 형변환** 또는 **묵시적 형변환**이라 한다.

```java
int intValue = 10;

double doubleValue = intValue;
// double doubleValue = (double) intValue // 자동 형변환

System.out.println(doubleValue); // 10.0
```


### 명시적 형변환

큰 범위에서 작은 범위로 대입할 때는 반드시 **명시적 형변환**이 필요하다.

> 🚨 주의
>
> 큰 범위에서 작은 범위로 대입할 경우 명시적 형변환을 하지 않으면 컴파일 오류가 발생한다. 컴파일 오류는 문제를 가장 빨리 발견할 수 있는 좋은 오류이다.

```java
java: incompatible types: possible lossy conversion from double to int 
```

**명시적 형변환**은 변경하고 싶은 데이터 타입을 `(int)` 와 같이 괄호를 함께 사용하여 변수앞에 입력하면 된다. 영어로는 **캐스팅(casting)**이라고 한다.
```java
public class Casting {
   public static void main(String[] args) {
      double doubleValue = 1.5;
      int intValue = 0;
			//intValue = doubleValue; // 컴파일 오류 발생 
			intValue = (int) doubleValue; // 명시적 형변환 
			System.out.println(intValue); // 출력:1
	 } 
}
```

## 형변환과 오버플로우

형변환을 할 때 만약 작은 숫자가 표현할 수 있는 범위를 넘어서면 어떻게 될까?
```java
package casting;
public class Casting {
	public static void main(String[] args) {
		long maxIntValue = 2147483647; //int 최고값
		long maxIntOver = 2147483648L; //int 최고값 + 1(초과) 
		int intValue = 0;
		
		intValue = (int) maxIntValue; //형변환
		System.out.println("maxIntValue casting=" + intValue); //출력:2147483647
		
		intValue = (int) maxIntOver; //형변환
		System.out.println("maxIntOver casting=" + intValue); //출력:-2147483648 
	}
}
```
**출력결과**
```java
 maxIntValue casting=2147483647
 maxIntOver  casting=-2147483648
```
- `long maxIntValue = 2147483647` 정수 범위의 최대값을 가진 `long` 타입을 `int`로 캐스팅할 때는 아무런 문제가 없다.
- `long maxIntOver = 2147483648L` 정수 범위 최대값을 넘긴 리터럴은 `L`을 붙여서 `long` 타입으로 받고 있다. 이 경우 `int`로 표현할 수 있는 범위를 초과했기 때문에 `long` -> `int` 캐스팅시 오버플로우 문제가 발생한다.


> 💡 중요한 것은 오버플로우 자체가 발생하지 않도록 막아야 한다는 것이다. 오버플로우가 발생한 결과를 가지고 어떻게 할 생각을 하지 말자. 변수 타입을 `int` -> `long`으로 사이즈를 늘리면 오버플로우 문제를 간단히 해결할 수 있다.


## 계산과 형변환

대입뿐만 아니라 계산식에서도 형변환이 일어난다.
자바에서 계산에서의 형변환은 다음 2가지를 꼭 기억하자.

1. 같은 타입끼리의 계산은 같은 타입의 결과를 낸다.
   - `int` + `int` -> `int` 
2. 서로 다른 타입의 계산은 큰 범위로 자동 형변환이 일어난다.
   - `int` + `long` -> `long` + `long`
   - `int` + `double` -> `double` + `double`

---
