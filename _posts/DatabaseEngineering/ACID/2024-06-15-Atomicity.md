---
title: Atomicity 원자성
description: 트랜잭션의 ACID 속성중 원자성에 대해 알아보자.
categories:
- Database
- DB Engineering
- Transaction

tags:
- Actomicity
- Database
- Transaction

---

<!-- more -->

## 정의

트랜잭션 내 모든 쿼리는 성공해야 한다는 규칙이다.

어떠한 이유로든 단 하나의 쿼리라도 실패한다면 **롤백**되어야 한다. 이것이 **원자성**의 규칙이다.

마찬가지로, 트랜잭션이 실행 도중 데이터베이스가 중단되더라도 다시 시작할 때 **롤백**되어야 한다.

원자성은 **<u>일관된</u>** 데이터베이스 상태를 위해 반드시 필요하다.

---
