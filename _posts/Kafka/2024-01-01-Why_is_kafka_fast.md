---
title: Why is kafka fast?
description: What is Zero Copy?
categories:
- Kafka

tags:
- Kafka

---

> Kafka를 빠르게 하는 Zero Copy에 대해 알아보자.

<!-- more -->

---

## Kafka는 왜 빠른 것일까?
아래와 같은 이유 등으로 Kafka의 성능이 좋다.

- 파티션 파일은 OS 페이지캐시 사용
- Zero Copy
- 브로커가 하는 일이 비교적 단순
- 묶어서 보내기, 묶어서 받기 (batch) 가능

---

오늘은 Zero Copy 에 대해서 자세히 알아보자.

> Zero Copy란?
>
> Zero Copy는 데이터를 복사하지 않고 메모리를 공유하여 데이터 전송을 최적화하는 기술이다. 


<figure align="center">
<img src="/post_images/zero_copy.png">
<figcaption>what zero-copy means / Ref: ByteByteGo</figcaption>
</figure>

위 이미지에서, Zero Copy를 이용한다면 불필요한 데이터 copy가 발생하지 않고 바로 NIC Buffer를 통해 message를 consuming 하는 것을 확인할 수 있다.

 > **Zero copy**는 애플리케이션 컨텍스트와 커널 컨텍스트 사이에 여러 데이터 복사본을 저장하는 지름길이다. 이 접근 방식을 사용하면 시간이 약 65% 단축된다고 한다.

 ---
Reference

- [System Design Interview](https://www.amazon.com/System-Design-Interview-Insiders-Guide/dp/1736049119?utm_source=li_posts)
- [kafka 조금 아는 척하기 1(개발자용), 최범균](https://www.youtube.com/watch?v=0Ssx7jJJADI)
 


