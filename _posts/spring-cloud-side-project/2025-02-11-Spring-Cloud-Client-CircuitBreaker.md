---
title: Spring Cloud CircuitBreaker
description: 
categories:
- Spring Cloud
- MSA

tags:
- Circuit Breaker
- Resilience4j

---

> 마이크로 서비스 간 통신 장애 대처 방식 중 CircuitBreaker 사용 방법에 대해 알아보자!

<!-- more -->

# 들어가기 

MSA 통신 중 장애가 발생한 경우, 장애 대처 방식 중 하나인 CircuitBreaker 에 대해 알아보겠습니다.

CircuitBreaker 를 직역하자면, 회로 차단기로 볼 수 있습니다.

회로 차단기를 전류의 흐름을 막거나 흐르게 하는 흐름 제어기입니다.

CircuitBreaker는 API 요청의 흐름을 막거나 흐르게 하는 API 요청 제어기로 볼 수 있습니다.

만약 어떤 서버가 다운되버렸다면, 그 서버에 API 호출을 하더라도 응답을 받지 못할 것입니다.

단순히 응답을 받지 못하는 것에 그치지 않고, 이 에러는 MSA 환경에서 전파가 됩니다.

예를 들어, 사용자가 자기 자신의 주문 정보를 조회하는 API가 User Service에 있다고 가정해보겠습니다.

클라이언트의 요청은 `User Service -> Order Service` 서비스를 거치면서 필요한 정보들을 조회하는 API를 연달아 호출합니다. 이때 Order Service에 장애가 발생한다면, User Service는 응답을 받지 못 할 것이고 결국 User Service를 호출한 클라이언트도 에러를 응답받게 될 것 입니다.

이런 상황의 경우, Order Service가 회복되기 전까지 반복적으로 요청을 해봤자 에러 응답만 받을 뿐입니다.

그리고 주문 관련 데이터를 조회 못 했을 뿐, 사용자의 정보는 User Service에서 조회가 가능합니다. 단지, 주문 데이터를 조회 못 했다는 이유로 사용자에게 에러 응답을 보내는 것은 부자연스런 처리라고 생각합니다.

이런 문제점들을 해결하고자, Spring Cloud 는 CircuitBreaker의 추상화 계층을 제공하며 CircuitBreak의 구현체 클래스로 Resilience4j가 널리 사용되고 있습니다.

그러면 [Resilience4j](https://github.com/resilience4j/resilience4j)의 많은 기능 중 CircuitBreaker에 대해 알아보도록 하겠습니다.

# Spring Cloud CircuitBreaker

**스프링 클라우드는 CircuitBreaker의 추상화 계층을 제공합니다.**

아래 코드는 스프링 클라우드가 제공하는 CircuitBreaker의 인터페이스 입니다.

```java
package org.springframework.cloud.client.circuitbreaker;

import java.util.function.Function;
import java.util.function.Supplier;

public interface CircuitBreaker {
    default <T> T run(Supplier<T> toRun) {
        return this.run(toRun, (throwable) -> {
            throw new NoFallbackAvailableException("No fallback available.", throwable);
        });
    }

    <T> T run(Supplier<T> toRun, Function<Throwable, T> fallback);
}
```
CircuitBreaker의 사용법에 대해 간단히 알아보겠습니다.

- 우선, CircuitBreaker 인터페이스를 구현한 객체를 생성합니다.
- 구현체를 생성하고 `run` 메서드에 `Supplier`와 `Function`을 인자로 전달합니다.
- `Supplier` 은 실제로 처리하고자 하는 비즈니스 로직을 작성해주면 됩니다.
- `Function` 은 만약 `Supplier`가 실패하거나 회로가 차단된 경우에 실행하는 `fallback` 메서드를 작성해주면 됩니다. 즉, 에러를 응답하는 대신 별도의 로직을 처리하는 로직을 전달합니다.

# CircuitBreakerFactory

스프링 클라우드는 CircuitBreaker의 Factory 패턴의 인터페이스도 제공합니다.

Resilience4j는 해당 Factory 인터페이스의 구현체를 제공합니다.

```java
package org.springframework.cloud.client.circuitbreaker;

public abstract class CircuitBreakerFactory<CONF, CONFB extends ConfigBuilder<CONF>> extends AbstractCircuitBreakerFactory<CONF, CONFB> {
    public CircuitBreakerFactory() {
    }

    public abstract CircuitBreaker create(String id);

    public CircuitBreaker create(String id, String groupName) {
        return this.create(id);
    }
}
```

Resilience4j의 의존성을 추가하면 CircuitBreakerFactory 구현체를 사용할 수 있게 됩니다.

# Resilience4j CircuitBreaker

우선 Resilience4j 의존성을 추가해줍니다.

```groovy
// gradle
dependencies {
  implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.2.0'  
}
```

그 다음으로 CircuitBreaker의 설정을 구성해줍니다. 

마지막으로 해당 설정을 가진 커스텀 CircuitBreakerFactory를 `Bean` 으로 등록하여 사용하면 됩니다.

아래 코드는 CircuitBreaker의 설정을 커스텀한 CircuitBreakerFactory를 Bean 으로 등록하는 코드입니다.

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(3)).build())
            .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
            .build());
}
```
- 기본적으로 설정값으로 제공하는 CircuitBreakerConfig 기본 베이스로 설정해줍니다.
- 그 외 커스텀마이징 하고 싶은 값들은 빌더(Builder) 패턴으로 각각 설정할 수 있습니다.
- 위 코드에서는 커스텀마이징한 `timeLimiterConfig` 설정은 지정한 시간 내에 작업이 완료되지 않으면 타임아웃을 발생시키는 설정입니다. 타임아웃이 계속 발생하다가 임계치를 넘어가게 되면, 회로가 차단(Closed)됩니다.

기본 설정 값들은 `CircuitBreakerConfig.ofDefault()`는 아래와 같은 설정  값들로 구성되어 있습니다.

각 설정들에 대한 의미는 [공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)에 잘 정리되어 있습니다.

```java
public Builder() {
    this.currentTimestampFunction = CircuitBreakerConfig.DEFAULT_TIMESTAMP_FUNCTION;
    this.timestampUnit = CircuitBreakerConfig.DEFAULT_TIMESTAMP_UNIT;
    this.recordExceptions = new Class[0];
    this.ignoreExceptions = new Class[0];
    this.failureRateThreshold = 50.0F;
    this.minimumNumberOfCalls = 100;
    this.writableStackTraceEnabled = true;
    this.permittedNumberOfCallsInHalfOpenState = 10;
    this.slidingWindowSize = 100;
    this.recordResultPredicate = CircuitBreakerConfig.DEFAULT_RECORD_RESULT_PREDICATE;
    this.waitIntervalFunctionInOpenState = IntervalFunction.of(Duration.ofSeconds(60L));
    this.transitionOnResult = CircuitBreakerConfig.DEFAULT_TRANSITION_ON_RESULT;
    this.automaticTransitionFromOpenToHalfOpenEnabled = false;
    this.slidingWindowType = CircuitBreakerConfig.DEFAULT_SLIDING_WINDOW_TYPE;
    this.slowCallRateThreshold = 100.0F;
    this.slowCallDurationThreshold = Duration.ofSeconds(60L);
    this.maxWaitDurationInHalfOpenState = Duration.ofSeconds(0L);
    this.createWaitIntervalFunctionCounter = 0;
}
```


# 사이드 프로젝트 적용 코드

사이드 프로젝트에서 CircuitBreaker를 적용한 코드를 간단히 살펴 보도록 하겠습니다.

아래 코드는 `userId` 를 통해 사용자 정보와 주문 정보를 합쳐서 응답하는 간단한 메서드입니다.

```java
@Override
public UserDto getUserByUserId(String userId) {
    log.info("user-service: 회원ID로 회원정보 및 주문 목록 조회");

    Optional<Users> findUser = userRepository.findByUserId(userId);

    if (findUser.isEmpty()) {
        throw new NoSuchElementException("User not found");
    }

    UserDto userDto = modelMapper.map(findUser.get(), UserDto.class);

    List<ResponseOrder> orders = orderClient.getOrdersByUserId(userId);

    userDto.setOrders(orders);

    return userDto;
}
```

1. `userId` 로 user 엔티티를 조회합니다.
2. 만약 `userId`에 해당하는 User가 없다면 에러가 발생합니다.
3. user 엔티티를 DTO로 매핑합니다.
4. CircuitBreaker를 적용한 OrderClient를 통해 user의 주문 목록을 조회해옵니다.
5. 주문 목록을 DTO에 설정해주고 반환합니다.

여기서 UserService -> OrderService 는 에러가 발생할 수 있기 때문에, 주문 목록을 조회해오지 못 하고 에러가 전파될 가능성이 있습니다.

아래 코드는 CircuitBreakerFactory 로 부터 CircuitBreaker 객체를 생성하고, 주문 목록을 조회해오는 API 호출 로직(Supplier)과 호출 실패 또는 회로 차단의 경우 대신할 로직(Fallback Function)을 CircuitBreaker 의 인자로 전달하는 코드입니다.

```java
@Service
@RequiredArgsConstructor
public class OrderClient {

    private final OrderApiService orderApiService;
    private final CircuitBreakerFactory circuitBreakerFactory;

    public List<ResponseOrder> getOrdersByUserId(String userId) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory.create("circuitbreaker");
        ResponseEntity<List<ResponseOrder>> orderResponse = circuitBreaker.run(
                () -> orderApiService.getOrdersByUserId(userId),
                throwable -> new ResponseEntity<>(new ArrayList<>(), null, HttpStatus.INTERNAL_SERVER_ERROR));
        return orderResponse.getBody();
    }

}
```
- 회로가 차단되어 있지 않은 상태에서 정상적으로 API 응답을 받으면 기존 흐름대로 정상 응답합니다.
- 만약 Order Service가 응답에 실패하거나 회로가 차단되어 있는 상태라면 fallback 메서드를 수행하여 에러를 응답하는 대신 위 코드에서는 빈 리스트(Empty List)를 반환하도록 구성했습니다.
- 회로의 상태 전이, 실패 임계치 등 더 자세한 정보는 [공식 문서](https://resilience4j.readme.io/docs/circuitbreaker)를 통해 확인하실 수 있습니다.

# 스프링 AOP와 CircuitBreaker

CircuitBreaker 의 역할은 API 요청에 대한 흐름 제어입니다.

기존 흐름대로 흘러가게 할 지, 아니면 기존 요청을 차단하고 다른 응답을 대신 처리 할 지를 결정합니다.

만약 CircuitBreaker의 역할이 담긴 코드가 핵심 비즈니스 로직과 섞여있다면 가독성이 떨어지고 유지보수가 어려워질 것입니다.

따라서 스프링은 애너테이션을 활용한 `@CircuitBreaker` 를 제공합니다.

**이는 프록시 패턴을 활용한 스프링 AOP를 적용한 사례입니다.**

아래 코드는 `@CircuitBreaker`를 활용한 간단한 예제 코드입니다.

```java
@RestController
public class SampleController {

    // name: 회로 차단기 이름, fallbackMethod: 실패 시 호출할 메서드명
    @GetMapping("/process")
    @CircuitBreaker(name = "myCircuitBreaker", fallbackMethod = "fallbackProcess")
    public String process() {
        
        // 핵심 비즈니스 로직
        
        // 임의로 예외 발생
        if (Math.random() < 0.5) {
            throw new RuntimeException("Simulated exception");
        }
        return "Operation succeeded!";
    }

    // 폴백 메서드: 원래 메서드와 같은 반환 타입, 동일 파라미터 목록에 Throwable 추가
    public String fallbackProcess(Throwable throwable) {
        // 예외 메시지를 로깅하거나 별도의 처리 가능
        return "Fallback response: " + throwable.getMessage();
    }
}
```

- 위 코드를 보면 CircuitBreaker의 역할인 회로 차단에 관한 코드가 전혀 보이질 않습니다. 즉, 스프링 AOP에 의해 관심사가 완전히 분리된 것입니다.
- 따라서 CircuitBreaker를 적용하면서도 비즈니스 로직에 더 집중 할 수 있게 됩니다.

# 🚀 추가 개발 및 고려하면 좋을 내용
- Resilience4j의 메트릭 및 모니터링 기능(prometheus, micrometer와의 연동)
- fallback 로직 외에도, 재시도(Retry)나 백오프(Backoff) 전략 조합

---

# 📌 요약

## CircuitBreaker 개념:
서비스 장애 시, 지속적인 요청으로 인한 오류 전파를 막고 대체 로직(fallback)을 실행하여 시스템 안정성을 높이는 역할을 합니다.

## Spring Cloud CircuitBreaker:
다양한 CircuitBreaker 구현체(예: Resilience4j)를 추상화하여 사용하기 쉽고, Spring 생태계와 통합되어 있으며, AOP를 활용해 비즈니스 로직과 관심사를 분리합니다.

## Resilience4j CircuitBreaker:
세밀한 설정과 최적화된 구현체를 제공하며, 커스텀마이징한 설정과 함께 사용할 수 있습니다.























---
