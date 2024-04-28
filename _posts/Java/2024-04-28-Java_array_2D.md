---
title: 김영한 자바 입문 배열 - 2차원
description: 김영한 자바 입문 배열 - 2차원
categories:
- Java
- 김영한 자바 입문

tags:
- Java
- Array
- Data Structure

---

<!-- more -->

## 2차원 배열 선언, 생성 그리고 사용

2차원 배열은 `int[][] arr = new int[2][3]`와 같이 선언하고 생성한다. 그리고 `arr[1][2]`와 같이 사용하는데, 먼저 행(row) 번호를 찾고, 그 다음 열(column) 변호를 찾으면 된다.

> `arr[행][열]`, `arr[row][column]`

2차원 배열을 선언, 생성, 사용하는 예제 코드를 살펴보자.
```java
public class ArrayDi0 {

    public static void main(String[] args) {
        // 2x3 2차원 배열을 만든다.
        int[][] arr = new int[2][3]; //행2, 열3

        arr[0][0] = 1; //0행, 0열
        arr[0][1] = 2; //0행, 1열
        arr[0][2] = 3; //0행, 2열
        arr[1][0] = 4; //1행, 0열
        arr[1][1] = 5; //1행, 1열
        arr[1][2] = 6; //0행, 2열

        //0행 출력
        System.out.print(arr[0][0] + " "); //0열 출력
        System.out.print(arr[0][1] + " "); //1열 출력
        System.out.print(arr[0][2] + " "); //2열 출력
        System.out.println();//한 행이 끝나면 라인을 변경한다.

        //1행 출력
        System.out.print(arr[1][0] + " "); //0열 출력
        System.out.print(arr[1][1] + " "); //1열 출력
        System.out.print(arr[1][2] + " "); //2열 출력
        System.out.println();//한 행이 끝나면 라인을 변경한다.
    }
}
```

다음은 중첩된 for문을 사용하여 첫 번째 for문은 row를 탐색하고, 두 번째 for문은 열을 탐색하여 작업을 수행하도록 리팩토링 해보자.
```java
public class ArrayDi2 {

    public static void main(String[] args) {
        // 2x3 2차원 배열을 만든다.
        int[][] arr = new int[2][3]; //행2, 열3

        arr[0][0] = 1; //0행, 0열
        arr[0][1] = 2; //0행, 1열
        arr[0][2] = 3; //0행, 2열
        arr[1][0] = 4; //1행, 0열
        arr[1][1] = 5; //1행, 1열
        arr[1][2] = 6; //0행, 2열

        for (int row = 0; row < 2; row++) {
            for (int column = 0; column < 3; column++) {
                System.out.print(arr[row][column] + " ");
            }
            System.out.println();//한 행이 끝나면 라인을 변경한다.
        }
    }
}
```

## 2차원 배열 간단한 생성과 초기화

1차원 배열과 마찬가지로 `{}`를 사용하여 간단히 생성과 초기화를 할 수 있다.
```java
public class ArrayDi3 {

    public static void main(String[] args) {
        // 2x3 2차원 배열을 만든다.
        int[][] arr = new int[][]{
            {1,2,3},
            {4,5,6}
        }; //행2, 열3

        for (int row = 0; row < arr.length; row++) {
            for (int column = 0; column < arr[row].length; column++) {
                System.out.print(arr[row][column] + " ");
            }
            System.out.println();//한 행이 끝나면 라인을 변경한다.
        }
    }
}
```

`new int[][]`를 생략하여 한 줄로 선언과 초기화를 할 수 있다.
```java
public class ArrayDi3 {

    public static void main(String[] args) {
        // 2x3 2차원 배열을 만든다.
        int[][] arr = { // new int[][] 생략 가능
            {1,2,3},
            {4,5,6}
        }; //행2, 열3

        for (int row = 0; row < arr.length; row++) {
            for (int column = 0; column < arr[row].length; column++) {
                System.out.print(arr[row][column] + " ");
            }
            System.out.println();//한 행이 끝나면 라인을 변경한다.
        }
    }
}
```

> 라인을 적절하게 넘겨주어 행과 열이 명확해지도록 간단히 초기화하는 것이 코드의 가독성을 높인다.
```java
{
  {1, 2, 3},
  {4, 5, 6}
}
```

## 2차원 배열 길이

- `arr.length`는 행의 길이를 뜻한다.
- `arr[row].length`는 열의 길이를 뜻한다.



---

