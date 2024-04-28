---
title: 김영한 자바 입문 배열
description: 김영한 자바 입문 배열
categories:
- Java
- 김영한 자바 입문

tags:
- Java
- Array
- Data Structure

---

<!-- more -->

## 배열이 필요한 이유

같은 타입의 변수를 반복해서 선언하고 반복해서 사용하는 문제를 해결하기 위해 필요한 것이 배열이다.

```java
public class Array1 {
   public static void main(String[] args) {
       int student1 = 90;
       int student2 = 80;
       int student3 = 70;
       int student4 = 60;
       int student5 = 50;

			 System.out.println("학생1 점수: " + student1); 
			 System.out.println("학생2 점수: " + student2); 
			 System.out.println("학생3 점수: " + student3); 
			 System.out.println("학생4 점수: " + student4); 
			 System.out.println("학생5 점수: " + student5);
		}
}
```

## 배열의 선언과 생성

> 배열은 같은 타입의 변수를 사용하기 편하게 하나로 묶어둔 것이다.

```java
public class Array1Ref1 {

    public static void main(String[] args) {
        int[] students; //배열 변수 선언
        students = new int[5]; //배열 생성

        //변수 값 대입
        students[0] = 90;
        students[1] = 80;
        students[2] = 70;
        students[3] = 60;
        students[4] = 50;

        //변수 값 사용
        System.out.println("학생1 점수: " + students[0]);
        System.out.println("학생2 점수: " + students[1]);
        System.out.println("학생3 점수: " + students[2]);
        System.out.println("학생4 점수: " + students[3]);
        System.out.println("학생5 점수: " + students[4]);
    }
}
```

아주 간단해보이는 다음 두 줄은 아주 자세히 살펴봐야 한다.
```java
int[] students; //1. 배열 변수 선언 
students = new int[5]; //2. 배열 생성
```

### 배열 변수 선언

- 배열을 사용하려면 `int[] students`와 같이 배열 변수를 선언해야 한다.
- 일반적인 변수와 차이점은 `int[]` 처럼 타입 다음에 대괄호`[]`가 들어간다는 점이다.
- 배열 변수를 선언한다고 해서 사용할 수 있는 배열이 만들어진 것은 아니다!
  - `int[] students`와 같은 배열 변수에는 배열을 담을 수 있다.

### 배열 생성

- 배열을 사용하려면 배열을 생성해야 한다.
- `new int[5]`라고 입력하면 총 5개의 `int`를 저장할 수 있는 배열 변수가 생성된다.

### 배열과 초기화

- `new int[5]`라고 하면 총 5개의 `int`형 변수가 만들어진다. **자바는 배열을 생성할 때 그 내부값을 자동을 초기화 한다.**
- 숫자는 `0`, `boolean`은 `false`, `String`은 `null`로 초기화 된다.

### 배열 참조값 보관
- `new int[5]`로 배열을 생성하면 배열의 크기만큼 메모리를 확보한다.
  - `int`형 5개: `4byte * 5` => `20byte`
- **배열을 생성하고 나면 자바는 메모리 상에서 해당 배열에 접근할 수 있는 참조값(주소)을 반환한다.**
- 앞서 선언한 배열 변수인 `int[] students`에 배열의 참조값을 보관한다. 참조값을 통해 메모리에 있는 실제 배열에 접근하고 사용할 수 있다.

```java
int[] students = new int[5]; // 1. 배열 생성
int[] students = x001; // 2. new int[5]의 결과로 x001 참조값 반환
students = x001; // 3. 배열 별수에 참조값 보관
```

## 배열 사용

배열을 사용하려면 변수에 `[]`를 붙이고 대괄호 안에 배열의 위치를 나타내는 숫자인 인덱스(index)를 넣어주면 된다.

```java
//변수 값 대입
students[0] = 90; 
students[1] = 80;

//변수 값 사용
System.out.println("학생1 점수: " + students[0]);
System.out.println("학생2 점수: " + students[1]);
```

### 배열에 값 대입

```java
students[0] = 90; //1. 배열에 값을 대입
x001[0] = 90; //2. 변수에 있는 참조값을 통해 실제 배열에 접근. 인덱스를 사용해서 해당 위치의 요소에
접근, 값 대입
```

### 배열 값 읽기

```java
//1. 변수 값 읽기
System.out.println("학생1 점수: " + students[0]);
//2. 변수에 있는 참조값을 통해 실제 배열에 접근. 인덱스를 사용해서 해당 위치의 요소에 접근
System.out.println("학생1 점수: " + x001[0]);
//3. 배열의 값을 읽어옴
System.out.println("학생1 점수: " + 90);
```

> 배열을 사용하면 이렇게 참조값을 통해서 실제 배열에 접근하고 인덱스를 통해서 원하는 요소를 찾는다.

**참고**
- 배열은 인덱스가 0부터 시작한다. 따라서 사용 가능한 인덱스 범위는 `0 ~ (n-1)`이 된다.
- 만약 인덱스 허용 범위를 넘어선다면 에러가 발생한다.

```java
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out
of bounds for length 5 at array.Array1Ref1.main(Array1Ref1.java:14)
```

## 기본형 vs 참조형

자바의 변수 데이터 타입은 크게 **기본형**과 **참조형**으로 분류할 수 있다.

- **기본형(Primitive Type)**: `int`, `long`, `double`, `boolean`처럼 변수에 사용할 값을 직접 넣을 수 있는 데이터 타입을 기본형 또는 원시형이라 한다.
- **참조형(Reference Type)**: `int[] students`와 같이 데이터에 접근하기 위한 참조(주소)를 저장하는 데이터 타입을 참조형이라 한다.

**참고**
기본형과 참조형은 각각 언제 사용할까?

기본형은 모두 사이즈가 명확하게 정해져있다.
```java
int i; // 4byte
long l; // 8byte
double d; // 8byte
```

배열은 동적으로 사이즈를 변경할 수 있다.
```java
new int[size]; // size 값에 따라 배열의 크기가 정해진다.
```

- **기본형은 선언과 동시에 크기가 정해진다.** 따라서 크기를 동적으로 바꿀 수 없다. 반면에 배열과 같은 **참조형은 크기를 동적으로 할당할 수 있다.**
- 기본형은 사용할 값을 직접 저장한다. 반면에 참조형은 메모리에 저장된 배열이나 객체의 참조를 저장한다. **이로 인해 참조형은 더 복잡한 데이터 구조를 만들고 관리할 수 있다. 반면 기본형은 더 빠르고 메모리를 효율적으로 처리한다.**


## 배열 리팩토링
맨 처음 예제 코드를 다음과 같이 리팩토링 할 수 있다. 배열 생성 코드 `new int[]` 옆에 `{}`을 가지고 초기화를 해준다. 이때 배열의 크기는 작성하지 않는다.
```java
public class Array1Ref3 {

    public static void main(String[] args) {
        int[] students; //배열 변수 선언
        students = new int[]{90, 80, 70, 60, 50}; //배열 생성과 초기화

        //변수 값 사용
        for (int i = 0; i < students.length; i++) {
            System.out.println("학생" + (i + 1) + " 점수: " + students[i]);
        }
    }
}
```

### 간단한 배열 생성

> 배열은 `{}`만 사용해서 생성과 동시에 초기화 하는 기능을 제공한다. 단, 이때는 배열 변수의 선언과 한 줄에 함께 사용할 때만 가능하다.

```java
int[] students = {90, 80, 70, 60, 50};
```

**오류**
```java
int[] students;
students = {90, 80, 70, 60, 50}; // 이렇게 사용하지 말자.
```

간단한 배열 생성을 통해 리팩토링 해보자.
```java
public class Array1Ref4 {

    public static void main(String[] args) {
        int[] students = {90, 80, 70, 60, 50};

        //변수 값 사용
        for (int i = 0; i < students.length; i++) {
            System.out.println("학생" + (i + 1) + " 점수: " + students[i]);
        }
    }
}
```
배열을 사용한 덕분에 예제 코드를 전체적으로 잘 구조화 할 수 있었다.


---
