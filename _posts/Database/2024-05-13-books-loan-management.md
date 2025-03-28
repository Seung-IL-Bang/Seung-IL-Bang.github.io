---
title: 도서대출관리 모델링 연습 (1)
description: 데이터베이스 중급(Modeling) - 도서대출관리 모델링 연습 (1)
categories:
- Database
- 데이터베이스 중급(Modeling)

tags:
- Database

---

> 도서대출관리 모델링을 통해 1:M, M:N 관계 설계를 연습해보자.

<!-- more -->


## 도서관의 도서 대출 관리 요구 사항

- 도서관에는 각 서고/서가에 많은 책들이 있다
- 고객들은 인터넷을 통해서 로그인한 후 도서 목록을 조회할 수 있다
- 고객들은 원하는 책을 대출 받을 수 있다.
- 고객은 책이 있을 경우 대출 예약을 할 수 있으며 대출을 위해서는 직접 방문해서 책을 찾아서 대출을 해야 한다


### 마스터 테이블

- 서고, 서가, 책, 고객, 도서 목록

### 관계 테이블

- 로그인, 조회, 대출, 대출 예약


## 설계 순서

논리적 설계 → 물리적 설계

> 초기에는 1:M 관계를 찾는 것에 집중하자. 마스터 테이블 간 어떤 비즈니스 행위가 존재하는지 확신할 수 없는 상황에서 M:N 을 따져봤자 헛수고이다.

### 논리적 설계

- 한글로 작성하여 가독성을 높인다.
- 쓸데없는 컬럼 들어가지 않는다.
- PK, FK 및 없어서는 안되는 속성 정도로만 구성한다.

<figure align="center">
<img src="/post_images/Database/logical-theory.png">
<figcaption></figcaption>
</figure>


### 물리적 설계

- 영어로 실제 컬럼 이름을 짓는다.
- 데이터 형식을 제대로 잡아준다.
- NOT NULL, UNIQUE 등 제약을 설정해준다.
- 구체적인 속성들을 작성해준다.


## 서고와 서가
<figure align="center">
<img src="/post_images/Database/libraries-shelves.png">
<figcaption>PK를 어떻게 잡는지에 따라 서가ID의 생성체계가 달라진다. 서가2는 서고ID가 null 허용이다.</figcaption>
</figure>

### 서가1

- 서고를 반드시 입력해야 하는 경우 상속형 PK를 사용한다.(에러가 없지만, 융통성이 없다.)
- 서가 ID가 각 서고별로 1번부터 시작한다.

### 서가2

- 서가를 먼저 등록해야되는 경우에는 독립형 PK를 설정해줘야 한다.(융통성은 있지만 에러가 있을 수 있다. 나중에 업데이트를 잊는 경우 문제가 발생할 수 있다. 가비지 데이터가 된다.)
- 서고에 상관없이 서가ID는 계속 증가한다.

> DMBS 가 보기에는 독립형 PK를 사용하는 서가2의 FK가 NULL인 데이터는 전혀 문제가 없다. 하지만 사람들의 비즈니스 로직으로 봤을 때는 서고 ID(FK)가 NULL인 것은 오류로 해석되는 것이다.

**상속형 PK**를 사용하는 것이 안전하니 상속형 PK를 사용한다고 가정해보자.

<figure align="center">
<img src="/post_images/Database/bookshelves.png">
<figcaption>서가와 책 1:M 관계</figcaption>
</figure>

<figure align="center">
<img src="/post_images/Database/book-lists.png">
<figcaption>책과 목차 1:M 관계</figcaption>
</figure>

> 상속형 PK 테이블의 특징
>
> 손자격인 책 테이블은 서가의 PK를 FK로 물려받는다. 즉, 서가의 PK는 복합키(서고ID + 서가ID)이므로 책의 FK로 그대로 계승된다.

**상속형 PK**는 안전하지만 그 만큼 융통성이 없다고 볼 수 있다.

서가 - 책 - 목차를 살펴보면, 복합키가 계속 눈덩이처럼 커져서 계승되고 있는 것을 볼 수 있다.

즉, **상속형 PK**는 계승될 수록 PK에 참여하는 컬럼 개수가 늘어난다는 단점이 있다.

<figure align="center">
<img src="/post_images/Database/books-loan-customer.png">
<figcaption>(책 - 대출 - 고객) M:N 관계 테이블</figcaption>
</figure>

- `책Seq(AK, Alternate Key)`를 선언해줘서 책 조회와 대출 테이블의 FK로 편리하게 활용 가능하다. AK를 사용하지 않는다면 조회시 PK(책, 서고, 서가 ID)를 모두 작성해야 한다. 마찬가지로 대체키가 없다면 대출 테이블에도 전부 작성해야 한다.
- `대출일(REG_DATE)`은 관계 테이블의 필수 요소이다. `고객이 어떤 책을 언제 빌렸다.` 를 대출 테이블이 표현하고 있고, 이 한 문장을 표현할 수 있도록 `책Seq` , `고객ID` , `대출일` 을 복합키로 잡았다.

<figure align="center">
<img src="/post_images/Database/Draft.png">
<figcaption>1:1 관계를 완성하기 전 초안 ERD</figcaption>
</figure>

책과 도서목록은 **<u>1:1 관계</u>**이다.

1:1 관계를 완성해야지 설계가 최종적으로 완성이 된다.

1:1 관계 없이는 설계 완성이 안 될 수도 있다.

1:1 관계는 깊이를 요구하는 고난이도 기술이 요구 되기도 한다.

---
