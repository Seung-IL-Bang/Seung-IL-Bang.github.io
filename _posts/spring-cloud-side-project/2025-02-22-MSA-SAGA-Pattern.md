---
title: MSA에서 데이터 일관성 유지를 위한 Saga 패턴
description:
categories:
- Spring Cloud
- MSA

tags:
- Saga pattern

---

> Saga 패턴이 무엇이고, MSA 환겨에서 왜 필요한지 알아봅시다!

<!-- more -->

# 📌 들어가기

**마이크로서비스 아키텍처**에서는 각 서비스가 독립적인 데이터베이스를 가지고 있으며, 서비스 간 통신을 통해 비즈니스 로직이 연결되는 경우가 많습니다.

이때, 데이터 일관성을 유지하는 것이 중요한데, 전통적인 **분산 트랜잭션** 방식으로는 한계가 있습니다.

이번 포스트에서는 **Saga 패턴**의 필요성과 이를 도입하지 않을 경우 발생할 수 있는 문제점에 대해 살펴보도록 하겠습니다.


# 1. 전통적인 분산 트랜잭션 

MSA 환경에서 각 서비스는 독립적인 데이터베이스를 가지고 있기 때문에 데이터 정합성을 유지하기가 훨씬 어려워집니다.

전통적인 분산 트랜잭션은 데이터 정합성 불일치 문제를 해결하기 위한 **초기 접근 방식**으로, **여러 데이터베이스에 걸쳐 원자성(Atomicity)을 보장하려는 방식**이었습니다.

그러나 이 방식은 **한계점과 문제점을 동반**하게 되며, 결과적으로 **Saga 패턴**과 같은 대안이 등장하게 되었습니다.

## 전통적인 분산 트랜잭션 핵심: 2-Phase Commit(2PC)

MSA **다중 데이터베이스 환경**에서 하나의 비즈니스 로직이 수행될 때, 여러 데이터베이스에 걸쳐 관련 데이터들이 저장됩니다. **전통적인 분산 트랜잭션**에서 가장 널리 사용된 프로토콜로 `2-Phase Commit(2PC)`가 있습니다. 이는 `트랜잭션 관리자(Transaction Coordinator)`가 존재하여 전체 트랜잭션을 조율하는 방식입니다.

## 전통적인 분산 트랜잭션 한계점

전통적인 분산 트랜잭션 한계점은 다음과 같습니다.

- 락킹(Locking)으로 인한 성능 저하, 병목 지점이 될 수 있습니다.
- 트랜잭션은 관리하는 코디네이터가 **단일 실패 지점**이 될 수 있습니다.
- 네트워크 문제로 인해 트랜잭션 관리자의 지시를 받지 못하면, 교착 상태(Deadlock)에 빠질 수도 있습니다.
- 관리하는 노드가 많아질수록 트랜잭션을 조율하기 복잡해집니다. 이로 인해 시스템의 확장성이 저하됩니다.


💡 위와 같은 한계점들로 인해 전통적인 2PC 방식은 단일 시스템이나 적은 수의 서비스에 적합하지만, **확장성, 고가용성, 비동기성이 중요한 마이크로서비스 환경에는 적합하지 않습니다.**

❗️ 참고로, 이번 포스트의 주제는 전통적인 분산 트랜잭션의 한계점으로 인한 Saga 패턴 도입의 필요성을 다루는 포스트입니다. 따라서 전통적인 분산 트랜잭션에 대한 내용은 추후 다른 포스트에서 자세히 다루도록 하겠습니다.

# 2. 전통적인 분산 트랜잭션의 대안

전통적인 분산 트랜잭션의 대안으로 `Saga 패턴`과 `Outbox 패턴`이 있습니다.

두 패턴 중 이번 포스트에서는 Spring Cloud 사이드 프로젝트를 진행하면서 Saga 패턴을 적용한 내용을 다루겠습니다.

# 3. Saga 패턴이란?

Saga 패턴은 마이크로서비스 간 **분산 트랜잭션을 관리하기 위한 패턴**입니다. 

각 서비스의 로컬 트랜잭셩을 통해서 전체 비즈니스 로직의 일관성을 유지하는 방법이죠.

**Saga 패턴**의 핵심 아이디어는 하**나의 큰 트랜잭션을 여러 개의 트랜잭션으로 나누고**, 문제가 발생하면 **보상 트랜잭션(Compensating Transaction)**을 수행하여 일관성을 맞추는 것입니다.

## 3-1. Saga 패턴 예시

Saga 패턴이 무엇인지 이해하기 쉽도록 예시를 들어보겠습니다.

예를 들어, `Order Service -> Payment Service -> Product Service` 의 순서로 주문과 결제 그리고 재고 차감이 이뤄지는 비즈니스 로직이 있다고 하겠습니다.

1. `Order Service`: 주문 생성 -> `ORDER_CREATED` 이벤트 발행.
2. `Payment Service`: 결제 요청 -> 결제 성공 시 `PAYMENT_COMPLETED` 이벤트 발행.
3. `Product Service`: 재고 차감 -> 만약 재고 부족 시, **실패 이벤트** 발행.

🎯 만약 `Product Service`에서 재고 부족으로 인해 실패 이벤트가 발생하면, 보상 트랜잭션으로 **주문 취소 및 결제 취소**를 진행해주게 됩니다. 이처럼 실패에 대한 보상 처리를 해주는 것이 Saga 패턴입니다.


# 4. Saga 패턴의 종류

Saga 패턴에는 종류가 존재합니다. 크게 Choreography(코레오그래피) 방식과 Orchestration(오케스트레이션) 방식으로 나뉘게 됩니다.

각 방식에는 장단점이 존재하며, 프로젝트 환경에 따라 적절한 패턴을 적용하면 될 것 같습니다.

그럼, 각 패턴 종류의 장단점을 살펴보겠습니다.

## 4-1. Saga pattern - Choreography(코레오그래피)

**이벤트 기반**으로 **각 서비스가 직접 이벤트를 구독하고 다음 작업을 수행**하는 패턴입니다. 이는 **중앙 조정자가 없다**는 것이 특징입니다.

- **장점**: 단순한 구현, 낮은 복잡성
- **단점**: 서비스 간 의존성 증가, 복잡한 이벤트 플로우 관리 어려움

👉 **예시**

1. `Order Service` -> `ORDER_CREATED` 주문 생성 이벤트 발행
2. `Payment Service` 가 주문 생성 구독 후 결제 처리 -> `PAYMENT_COMPLETED` 결제 성공 이벤트 발행
3. `Delivery Service`가 결제 성공 구독 후 배송 요청 처리.

## 4-2. Saga Pattern - Orchestration(오케스트레이션)

**중앙 조정자(Service Orchestration)**가 각 서비스에 요청을 보내고 **전체 트랜잭션을 관리 및 조율**하는 패턴입니다.

- **장점**: 서비스 간 의존성 감소, 중앙 집중식으로 전체 플로우 관리 가능.
- **단점**: Orchestrator(중앙 조정자)를 추가 구현해야 하며, 단일 실패 지점(SPOF)이 될 수 있음.

👉 **예시**

1. **Orchestrator**가 Order Service 에 주문 생성 요청
2. 주문 생성 -> 결제 요청 -> 재고 차감 -> 배송 요청 순으로 전체 이벤트 발행 및 트랜잭션 관리
3. 실패 시 보상 트랜잭션을 직접 호출

## ✅ 4-3 Saga Pattern 종류 정리

Saga 패턴에는 Choreography와 Orchestration 두 방식이 존재합니다.

두 방식은 각각의 장단점이 상반되며, 프로젝트 환경에 알맞게 적절한 패턴 종류를 선택하면 됩니다.

참고로 저는 사이드 프로젝트를 진행할 때 단순한 구현과 낮은 복잡성으로 구성되도록 원했고, 중앙 조정자 도입이 오히려 서비스가 많아질 수록 중앙 조정자를 관리하기 어려워 질 것이란 판단하에 **Choreography(코레오그래피)** 방식을 적용하였습니다.

# 5. Saga 패턴 적용 예제 코드 

사이드 프로젝트에서 Saga 패턴(Choreography)을 적용한 예제 코드를 보여드리겠습니다. 

참고로 실제 코드 전부가 아닌 Saga 패턴 이해를 위한 간소화된 코드로 보여드리겠습니다.

## Order Service - `ORDER_CREATED` 이벤트 발행 
```java
@Transactional
@Override
public OrderDto createOrder(OrderDto orderDto) {
    log.info("Order-service: 주문 생성");
    
    // 주문 생성 로직 ...

    // ORDER_CREATED 이벤트 발행
    OrderCreatedEvent orderCreatedEvent = new OrderCreatedEvent(
            order.getOrderId(),
            BigDecimal.valueOf(totalPrice),
            orderDto.getPaymentInfos(),
            orderDto.getDeliveryInfo());
    orderEventProducer.send(ORDER_CREATED, orderCreatedEvent);
}
```

## Payment Service - `ORDER_CREATED` 이벤트 구독 후 결제 처리

```java
@KafkaListener(topics = "ORDER_CREATED", groupId = "${spring.kafka.consumer.group-id:payment-service-group}")
public void consume(ConsumerRecord<String, String> record) throws JsonProcessingException {
    try {
        String message = record.value();
        log.info("Consumed message: {}", message);
        OrderCreatedEvent orderCreatedEvent = objectMapper.readValue(message, OrderCreatedEvent.class);
        log.info("Order created event: {}", orderCreatedEvent);

        paymentHandler.handle(orderCreatedEvent); // 결제 처리 
    } catch (Exception e) {
        log.error("Error Consume OrderCreatedEvent", e);
    }
}
```

만약, 결제 처리 실패 시 보상 트랜잭션을 수행합니다.

`PAYMENT_FAILED` 이벤트 발행 -> `Order Service`에서 결제 실패를 구독하여 주문 취소를 진행합니다.

```java
public void handle(Event event) {
    try {
        handleEvent(event);
    } catch (Exception e) {
        handleException(event, e);
    }
}

private void handleException(Event event, Exception e) {
    log.error("Error handling payment", e);
    if (event instanceof OrderCreatedEvent orderCreatedEvent) {
        paymentEventProducer.send(PAYMENT_FAILED, new PaymentFailedEvent(
                orderCreatedEvent.getOrderId(),
                null,
                FAILED, e.getMessage()));
    }
    ...
}
```

# 6. Saga 패턴 도입 시 장점

분산 트랜잭션 처리를 위해 Saga 패턴을 도입하면 다음과 같은 장점이 있습니다.

## 6-1. 데이터 정합성 보장

서비스 간 비동기 처리에도 Eventually Consistent(최종 일관성) 상태를 유지할 수 있습니다. 예를 들어, 주문 생성 후 결제를 실패한다면, 주문을 취소하여 데이터 불일치를 방지할 수 있습니다.

이처럼, 보상 트랜잭션을 통해 데이터 일관성이 유지됩니다.

## 6-2. 마이크로서비스 독립성 유지

각 서비스는 로컬 트랜잭션만 관리하면 되므로, 강한 결합(Tightly Coupled)을 피할 수 있습니다.

## 6-3. 확장성 향상

서비스 간의 직접적인 의존성이 줄어들어, 새로운 서비스 추가 및 확장이 쉬워지게 됩니다.

# 7. Saga 패턴을 도입하지 않으면 발생하는 문제점

## 7-1. 데이터 불일치

실패 시 적절한 보상 트랜잭션이 없다면, **결제 실패**나 **재고 부족** 등의 시나리오에서, 주문 상태는 **"완료"**이지만 실제로는 결제가 되지 않은 주문이 발생할 수 있습니다.

## 7-2. 사용자 경험 저하

주문은 **완료 상태**이고 **결제는 실패**했을 때, 제대로된 보상 트랜잭션 처리를 하지 않는다면 **"주문 정상 처리"**라는 잘못된 정보를 사용자에게 전달할 수도 있습니다.


# 🎯 결론

- Saga 패턴은 마이크로서비스 환경에서 **데이터 정합성을 유지하기 위한 필수 설계 패턴**으로 볼 수 있습니다.
- 보상 트랜잭션을 통해 실패 시에도 전체 시스템의 일관성을 유지할 수 있으며, 서비스 간 결합도를 낮춰 **확장성**과 **유연성**을 확보할 수 있도록 도와줍니다.

Kafka와 같은 이벤트 드리븐 기반의 MSA 환경에서 Saga 패턴을 도입하지 않으면, 데이터 불일치 및 비즈니스 오류가 발생할 가능성이 높습니다. 안정적인 시스템을 만들기 위해서는 반드시 **Saga 패턴과 같은 분산 트랜잭션 관리 기법**이 필요합니다.


# 💡 추가로 공부하면 좋을 내용

- [CAP 이론](https://en.wikipedia.org/wiki/CAP_theorem)
- [2-Phase Commit](https://ko.wikipedia.org/wiki/2%EB%8B%A8%EA%B3%84_%EC%BB%A4%EB%B0%8B_%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C)
- Outbox 패턴


---
