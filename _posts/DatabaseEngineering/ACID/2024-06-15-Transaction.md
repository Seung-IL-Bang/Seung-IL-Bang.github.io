---
title: DB에서 트랜잭션이란 무엇인가?
description: DB의 트랜잭션에 대해 알아보자.
categories:
- Database
- Transaction
- DB Engineering

tags:
- Transaction
- Database

---

<!-- more -->

## 정의

하나의 작업 단위로 취급되는 SQL 쿼리들의 모음.

## 트랜잭션 Lifecycle
```sql
Transaction BEGIN

Transaction COMMIT

Transaction ROLLBACK

Transaction unexpected ending  = ROLLBACK (e.g. crash)
```

만약 1000개의 쿼리, 1000개의 변경을 커밋한다면, 이 모든 변경을 실제로 디스크에 모두 쓰는 걸까?

아니면 기다렸다가 모든 것을 메모리에 넣은 다음 커밋할 때 한꺼번에 디스크에 쓰는 걸까?

각각의 방법에는 **장단점**이 있다.

첫 번째 방법은 커밋을 빠르게 만든다.

두 번째 방법은 커밋을 느리게 만든다.

결국 이러한 장단점을 파악하면서 개발을 해야한다.

## 트랜잭션 역할
- 트랜잭션은 보통 데이터를 수정하거나 업데이트할 때 사용된다.
- 읽기 전용 트랜잭션도 존재한다.

---
