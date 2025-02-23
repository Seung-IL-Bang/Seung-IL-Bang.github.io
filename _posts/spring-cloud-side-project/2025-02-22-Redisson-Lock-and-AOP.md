---
title: Redisson을 활용한 분산 락 그리고 AOP 적용
description:
categories:
- MSA
- Redis

tags:
- AOP
- Distributed Lock
- Redisson

---

> 분산 시스템에서 Redis를 이용한 분산 락의 필요성과 분산 락 적용 부분의 AOP 활용 방법을 알아봅시다.

<!-- more -->

# 📌 들어가기

MSA와 같은 분산 시스템에서는 여러 인스턴스가 동시에 하나의 리소스에 접근하는 경우가 빈번하게 발생할 수 있습니다. 예를 들어, 주문이 동시다발적으로 발생하여 상품의 재고를 차감하기 위해 여러 인스턴스가 재고에 접근합니다.

이때 동시성 이슈가 발생하여 데이터 무결성에 문제가 발생합니다. 이를 방지하기 위해 분산 락(Distributed Lock)이 필요한 것입니다.

본 포스트에서는 Redis 기반의 **분산 락**을 구현하기 위해 **Redisson** 라이브러리를 활요하는 방법과, **스프링 AOP(Aspect Oriented Programming)**를 통해 락 처리 로직을 분리하여 코드의 응집도와 유지보수성을 높이는 방법을 살펴보도록 하겠습니다.

# Redis 라이브러리: Redisson을 활용한 분산 락 

Redis는 인메모리 데이터 저장소로 높은 성능과 빠른 응답 속도를 제공합니다. 이를 활용하여 여러 프로세스 혹은 서버 간에 분산 락을 구현할 수 있으며, 분산 환경에서의 동시성 제어와 데이터 무결성 확보에 큰 역할을 하게 됩니다.

**Redisson**은 Redis를 Java 애플리케이션에서 쉽게 사용할 수 있도록 지원하는 **Redis 클라이언트 라이브러리**입니다. Redisson은 분산 락, 세마포어 등 여러 동시성 관련 기능을 제공하며, **복잡한 락 로직을 단순화**시켜 줍니다.

# Lettuce vs Redisson

`Spring Data Redis` 의존성을 추가하면 기본적으로 Lettuce 라이브러리를 Redis 클라이언트로 사용하게 됩니다. 기본적으로 제공하는 Lettuce가 있음에도 분산 락에 Redisson을 사용하는 이유는 무엇일까요?

분산 락에 `Lettuce` 대신 `Redisson`을 사용하는 이유는 **락 획득 방식** 차이점에 있습니다.

`Lettuce`는 스레드가 락 획득 대기 상태일 경우, `spin lock` 방식으로 대기하게 됩니다. 많은 스레드가 스핀 락으로 Redis에 락을 요청하게 될 경우 Redis에 큰 부하가 생길 수 있습니다. 반면에 `Redisson`은 락 획득 방식이 pub/sub 방식으로 구현되어 있기 때문에 스핀 락 방식보다는 Redis에 부하 부담이 줄어들게 됩니다. 더불어 락 획득 재시도를 기본 로직으로 제공해주기 때문에 편리하게 Lock을 사용할 수 있게 해줍니다.

# Redisson Lock 예제 코드

아래는 사이드 프로젝트에서 `Redisson`을 활용하여 분산 락을 구현한 코드입니다. 동시성 이슈가 발생할 수 있는 작업을 수행하기 전에 분산 락을 획득하고, 작업을 마쳤다면 락을 반납하면 됩니다. 이로써 여러 스레드가 동시에 접근하더라도 동시성 문제가 발생하지 않게 됩니다.

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class RedissonLockService {

    private final RedissonClient redissonClient;

    public boolean tryLock(String key, long waitTime, long leaseTime) {
        RLock lock = redissonClient.getLock(key);
        try {
            return lock.tryLock(waitTime, leaseTime, TimeUnit.SECONDS);
            // waitTime: 락을 기다리는 시간
            // leaseTime: 락이 자동으로 해제되기까지 유지되는 시간
        } catch (InterruptedException e) {
            log.error("Failed to acquire lock", e);
            return false;
        }
    }

    public void unlock(String key) {
        RLock lock = redissonClient.getLock(key);
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

- `tryLock` 메서드를 통해 락 획득 시도를 하게 됩니다. 획득에 성공한다면 `true`를 반환하고, `waitTime`동안 락을 획득하지 못한다면 예외가 발생하고 `false`를 반환하게 됩니다.
- `tryLock` 메서드는 블락킹(blocking) 메서드로, 락을 획득하지 못 한 경우 스레드가 대기 상태에 놓이게 됩니다.
- 락 획득 key는 동시성 이슈를 예방하고자 하는 대상의 고유한 식별자가 되어야 합니다. `ex: productId`
- `waitTime`동안 락 획득 대기 상태에 놓이게 되며, 락을 사용하던 스레드가 락을 해제하면 `pub/sub` 방식을 통해 대기 상태에 놓여있던 스레드에게 락 획득 시도를 하라고 알림을 보내게 됩니다.
- 만약 락을 획득하고 작업 시간이 `leaseTime`을 넘어가게 되면 자동으로 락이 해제됩니다. 따라서 `leaseTime` 이내로 작업이 끝내는 것을 보장해야 동시성 이슈가 발생하지 않습니다.
- 작업을 마친 스레드는 `unlock`을 통해 락을 해제하게 됩니다.

# 분산 락 활용 예제 코드 (재고 차감)

상품의 재고 수량을 변경시키는 작업은 동시성 이슈가 발생하는 대표적인 예시로 볼 수 있습니다. 여러 스레드가 동시에 상품 재고에 접근할 수 있기 때문입니다.

앞서 살펴본 Redisson을 활용한 분산 락을 재고 수량을 변경하는 로직 전후로 락을 획득/해제를 한다면 동시성 이슈를 예방할 수 있을 겁니다.

```java
@Transactional
@Override
public Product decreaseStock(String productId, Integer quantity) {

    long leaseTime = 5;
    long waitTime = 10;
    String lockKey = "lock:product:stock:" + productId;

    try {
      if (redissonLockService.tryLock(lockKey, waitTime, leaseTime)) { // 락 획득 시도
        Optional<Product> findProduct = productRepository.findByProductId(productId);
        if (findProduct.isEmpty()) {
            throw new IllegalArgumentException("Product not found");
        }

        Product product = findProduct.get();
        product.decreaseStock(quantity); // 재고 차감
        return product;
      }
    } finally {
      redissonLockService.unlock(lockKey); // 락 해제 
    }
}
```

- 상품 고유 식별자(productId)를 사용하여 상품 별로 분산 락을 제어할 수 있도록 했습니다.
- 락을 획득한 경우에만 재고 수량을 차감시킬 수 있게 됩니다.
- 락 획득에 실패하게 되면 예외가 발생하여, 재고 수량 차감 로직은 수행하지 못 하게 됩니다.
- 락 획득 후 작업을 완료했다면 락을 해제해줍니다.

# 분산 락 적용 부분의 AOP 분리 

애플리케이션에서 분산 락과 같은 횡**단 관심사(cross-cutting concern)**는 여러 서비스 메서드에서 반복적으로 구현되기 때문에, 이를 개별 비즈니스 로직과 분리하면 코드의 가독성 및 유지보수성이 크게 향상됩니다. **스프링 AOP**를 활용하면 메서드 호출 전후에 자동으로 락을 획득하고 해제하는 로직을 삽입할 수 있습니다.

> 아래 내용은 AOP에 대한 이해가 필요합니다!

## AOP 적용 방법

스프링 AOP를 적용하는 방법에 대해 단계별로 알아보겠습니다.

### 어노테이션 정의

락이 필요한 메서드에 적용할 커스텀 어노테이션(`@StockLock`)을 정의합니다.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface StockLock {
    long waitTime() default 5;
    long leaseTime() default 10;
}
```

### Aspect 클래스 작성

`@Around` 어드바이스를 활용하여 어노테이션이 붙은 메서드의 **실행 전후에 분산 락 획득 및 해제 로직을 삽입**합니다.

```java
@Aspect
@Slf4j
@Component
@Order(value = Integer.MAX_VALUE - 1) // AOP 우선순위를 지정합니다.
@RequiredArgsConstructor
public class StockAspect {

    private final RedissonLockService redissonLockService;

    @Around("@annotation(stockLock) && args(productId,..)")
    public Object doLock(ProceedingJoinPoint joinPoint, StockLock stockLock, String productId) throws Throwable {
        log.info("StockLockAspect.doLock {}", joinPoint.getSignature());

        long leaseTime = stockLock.leaseTime();
        long waitTime = stockLock.waitTime();
        String lockKey = "lock:product:stock:" + productId;

        if (redissonLockService.tryLock(lockKey, waitTime, leaseTime)) {
            try {
                return joinPoint.proceed();
            } finally {
                redissonLockService.unlock(lockKey);
            }
        } else {
            log.error("락을 획득하지 못하여 종료합니다. productId={}", productId);
            return false;
        }
    }
}
```

> 🚨 AOP 우선 순위 지정
>
> 스프링의 트랜잭션 처리도 AOP를 활용합니다. 이때 분산 락의 AOP와 트랜잭션의 AOP의 순서가 굉장히 중요합니다. 만약 트랜잭션 AOP가 먼저 실행된다면, 여전히 동시성 이슈가 발생할 가능성이 존재하게 됩니다. 따라서 분산 락 AOP를 먼저 적용하기 위해 @Order를 통해 순서를 지정해줍니다. 예제 코드에서는 Integer.MAX_VALUE - 1 값을 지정해주었는데, 이는 트랜잭션의 우선순위는 기본적으로 제일 후순위(INTEGER.MAX_VALUE)이기 때문입니다.


### 비즈니스 로직과 분리

실제 서비스 메서드는 락 관련 코드를 포함하지 않고, AOP가 이를 대신 처리하게 됩니다.

```java
    @Transactional
    @Override
    @StockLock
    public Product decreaseStock(String productId, Integer quantity) {
        log.info("product-service: 재고 차감");
        Optional<Product> findProduct = productRepository.findByProductId(productId);

        if (findProduct.isEmpty()) {
            throw new IllegalArgumentException("Product not found");
        }

        Product product = findProduct.get();
        product.decreaseStock(quantity);
        return product;
    }
```

## AOP 적용의 장점 

락 획득 및 해제 로직을 한 곳에서 관리하여 여러 서비스 메서드에 **중복된 코드를 제거**할 수 있습니다. 만약 락 처리 로직을 변경할 경우, Aspect 클래스만 수정하면 되므로 **관리가 용이**합니다. 그리고 서비스 메서드에서는 비즈니스 로직에만 집중할 수 있어 코드의 **가독성**이 좋아집니다.

# 결론 

**분산 시스템**에서 동시성 문제를 해결하기 위한 **분산 락은 필수**적인 요소입니다. Redis의 Redisson을 활용하면 높은 성능과 편리한 분산 락 구현이 가능합니다. 또한 AOP를 적용함으로써 락 관련 로직을 분리하면 코드의 유지보수성과 가독성이 크게 향상됩니다. 본 포스트를 참고하여 분산 락을 실제 구현하는 데 도움이 되면 좋겠습니다. 감사합니다.


# 참고 자료 

- [인프런 - 재고시스템으로 알아보는 동시성이슈 해결방법](https://www.inflearn.com/course/%EB%8F%99%EC%8B%9C%EC%84%B1%EC%9D%B4%EC%8A%88-%EC%9E%AC%EA%B3%A0%EC%8B%9C%EC%8A%A4%ED%85%9C)
- [카카오페이는 어떻게 수천만 결제를 처리할까? 우아한 결제 분산락 노하우 / if(kakaoAI)2024](https://www.youtube.com/watch?v=4wGTavSyLxE)

---
