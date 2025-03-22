---
title: 성적 관리 모델링 연습 (3)
description: 데이터베이스 중급(Modeling) - 성적 관리 모델링 연습 (3)
categories:
- Database
- 데이터베이스 중급(Modeling)

tags:
- Database

---

<!-- more -->

<figure align="center">
<img src="/post_images/Database/subject-openlecture-teacher.png">
<figcaption>과목-개강-선생님 관계도</figcaption>
</figure>

- 과목과 선생님은 M:N 관계이다. 그 가운데서 비즈니스 로직을 담당하는 관계 테이블인 개강 테이블이 존재한다.
- 개강 테이블(OpenLecture)의 PK는 `TeacherID` , `SubjectId` , `OpenDate` 가 합쳐진 복합키이다. 이처럼 상속형 PK를 사용하는 이유는 한 선생님이 동일한 학년 과목을 중복으로 개강할 수 없기 때문이다.
- 개강 테이블의 `Seq` 대체키를 선언하여 조회를 편리하게 돕도록 했다.

<figure align="center">
<img src="/post_images/Database/erd(1).png">
<figcaption>현재까지 완성된 ERD</figcaption>
</figure>

<figure align="center">
<img src="/post_images/Database/SchoolGrade-OpenLecture.png">
<figcaption>학년 마스터 테이블이 개강 관계 테이블과 1:M 관계를 맺었다.</figcaption>
</figure>

- 학년별로 각 과목이 존재하기 때문에 개강 테이블 PK에 SchoolGradeId 를 추가해준다. 이렇듯 관계 테이블에는 여러 집단으로부터 참조되는 테이블이다.

<figure align="center">
<img src="/post_images/Database/teacherPK-classFK.png">
<figcaption>선생님 테이블의 PK가 반 테이블의 FK로 1:M 관계를 맺고 있다.</figcaption>
</figure>

- 선생님 테이블의 PK가 반 테이블의 FK로 1:M 관계를 맺고 있지만, 원칙적으로 1:1 관계이다. 하지만 FK를 설정해준 목적은 담임 선생님이 없는 반 데이터를 만들지 않기 위함이다. 즉, 외래키 제약 조건을 이용하여 반드시 담임 선생님이 존재하도록 FK로 `TeacherId` 를 설정한 것이다. **(관계가 1:M 이라서 설정한 것이 아닌 것을 짚고 넘어가자.)** 이렇듯 관계가 1:M은 아니지만, PK와 FK 설정을 맺어줌으로써 FK로 사용되는 컬럼의 도메인 값을 `기준 테이블(선생님 테이블)` 의 도메인으로 한정짓고 반드시 값이 입력되도록 하기 위해서 사용되기도 한다.
- `TeacherID` 가 반 테이블에 들어옴으로써 언제 담임을 맡게 됐는지에 대한 정보가 존재해야지만 테이블 해석이 가능해진다. 따라서 `AssignedDate` 라는 담임을 맡게된 날짜 컬럼을 추가해준다.
- 참고로 `StudentId` 타입을 `bigint` 로 변경해줘야 한다. ~~(초기 설계 오류)~~
    - 또는 `entrance_date` 를 추가하면 매년 신입생이 입학할 때마다 `StudentId` 가 1번부터 다시 시작하는 방법도 존재한다. 이렇게하면 `smallint` 로 그대로 사용해도 된다.
- `TearcherId` 의 타이블 `tinyint` 에서 `int` 로 바꿔주자. 새로 발령오거나 퇴직하는 상황이 발생하면 `tinyint` 만으로는 부족할 것이다. ~~(초기 설계 오류)~~

<figure align="center">
<img src="/post_images/Database/gradebook-final-erd.png">
<figcaption>최종 ERD - 자동 정렬</figcaption>
</figure>

- 학생과 개강 테이블 간의 시험 점수 관계 테이블을 생성해준다.(M:N 관계)
    - `학생ID`와 개강 테이블의 `보조키(Seq)`를 PK 복합키로 잡는다.
    - 시험 점수 테이블(TB_Score)에 시험 점수(`score`) 속성을 추가해준다.

<figure align="center">
<img src="/post_images/Database/gradebook-final-erd2.png">
<figcaption>최종 ERD - 위치 변경</figcaption>
</figure>

- ERD 테이블 위치 선정 팁
    - 마스터 테이블 또는 기준 테이블들은 바깥에 위치시킨다.
    - 관계 테이블들은 가운데에 위치하도록 한다. 관계가 많이 맞물려있을 수록 가운데에 놓도록 하자. 핵심 비즈니스 관계 테이블일 확률이 높다.
- `OpenLecture` 개강 테이블의 경우 학년, 과목, 선생님 테이블들의 **관계 테이블 역할을 수행하면서**, 동시에 시험 점수 테이블의 **마스터 테이블로도 이용되고 있다.**

> 1:M, M:N 관계를 모르는 상태에서 설계를 하면 십중팔구 조인 테이블(=결과 테이블, 엑셀)을 설계한다. 잘못된 설계를 하면 데이터 이상 현상(중복, 삭제, 갱신) 등이 발생할 수 있다.

<figure align="center">
<img src="/post_images/Database/erdcloud-studentscore.png">
<figcaption>erdcloud 툴을 이용하여 정리한 ERD</figcaption>
</figure>

~~ERDCloud 툴에는 개강 테이블의 보조키를 성적 테이블의 PK로 잡는 방법이 없는 것 같다. ~~

---
