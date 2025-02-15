---
title: Spring Cloud FeignClient 로깅 및 에러 핸들링 
description:
categories:
- Spring Cloud
- MSA

tags:
- Feign Logger
- ErrorDecoder

---

> Feign Logger와 ErrorDecoder로 API 통신 로깅 및 에러 핸들링 방법에 대해 알아봅시다!

<!-- more -->

# 들어가기

Spring Cloud의 FeignClient는 REST API 호출을 간편하게 구현할 수 있는 강력한 도구입니다. 

하지만 API 통신 시 발생할 수 있는 문제들을 효과적으로 디버깅하고, 사용자 정의 에러 핸들링을 구현하기 위해서는 추가적인 설정이 필요합니다. 

이번 포스트에서는 `Feign Logger`와 `ErrorDecoder`를 통해 API 통신의 로깅과 에러 핸들링을 강화하는 방법에 대해 살펴보겠습니다.

# Feign Logger 

Feign Logger는 FeignClient가 호출하는 API의 요청과 응답을 로깅할 수 있는 기능을 제공합니다. 

이를 통해 API 통신 시 발생하는 문제를 보다 쉽게 파악할 수 있습니다.

로그 레벨에 따라 출력되는 정보가 다릅니다.

# Feign Logger.Level

기본적으로 아무것도 설정하지 않은 경우에는 `NONE` 상태입니다.

- `NONE`: 로깅을 하지 않음
- `BASIC`: 요청 메서드, URL, 응답 상태 코드 등 기본 정보만 로깅
- `HEADERS`: BASIC 정보에 HTTP 헤더도 로깅
- `FULL`: 모든 요청과 응답 바디를 포함한 상세 정보 로깅

# Feign Logger 설정하기

아래 코드는 Logger.Level 을 FULL 로 설정하여 모든 요청과 응답 바디를 로깅하도록 하는 설정 코드입니다.

```java
@Configuration
public class FeignConfiguration {

    @Bean
    Logger.Level feignLoggerLevel() {
        // FULL 로그 레벨 설정
        return Logger.Level.FULL;
    }
}
```

# ErrorDecoder

`ErrorDecoder`는 `FeignClient` 호출 시 에러 응답을 커스터마이징하여 처리할 수 있게 해주는 인터페이스입니다. 

기본적인 예외 처리 외에, 사용자 정의 예외로 변환하여 보다 의미 있는 에러 처리를 할 수 있습니다.

# Custom ErrorDecoder 구현

아래 코드는 사이드 프로젝트에서 `ErrorDecoder` 인터페이스를 구현한 클래스입니다.

🚨 ErrorDecoder 가 동작하기 위해서는 스프링 Bean 으로 등록되어 있어야 합니다.

ErrorDecoder 가 빈으로 등록되면 FeignClient 를 통해 발생하는 모든 예외는 ErrorDecoder를 거치게 됩니다.

```java
@Component
public class FeignErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        switch (response.status()) {
            case 400:
                // 400 예외의 경우, 별도의 사용자 정의 예외 처리 가능
                break;
            case 404:
                // 404 예외의 경우, 사용자 정의 예외 메시지 응답
                if (methodKey.contains("getOrders")) {
                    return new ResponseStatusException(HttpStatus.valueOf(response.status()), "User's orders are not found");
                }
                break;
            default:
                return new Exception(response.reason());
        }
        return null;
    }
}
```

### `String methodKey` 

이 파라미터는 FeignClient 인터페이스의 특정 메서드를 식별하는 고유한 키입니다.

호출한 메서드의 이름과 관련 정보를 포함하여, 에러 발생 시 어떤 메서드 호출에서 문제가 발생했는지 식별할 수 있도록 도와줍니다.

커스터마이징된 에러 처리 로직에서 특정 메서드에 따른 에러 핸들링을 다르게 적용할 수도 있습니다.

예시: `MyFeignClient#getUser` 처럼 인터페이스 이름과 메서드 이름을 조합한 형태로 전달됩니다.

### `Response response`

이 객체는 FeignClient가 API 호출을 통해 받은 HTTP 응답을 나타냅니다.

HTTP 상태 코드, 헤더, 바디 등의 정보를 확인할 수 있습니다.

또한 응답 데이터를 기반으로 에러 상황에 맞는 예외를 생성하거나 추가 로깅을 할 때 활용됩니다.


# Spring cloud Retry와 ErrorDecoder

FeignClient를 통해 API 호출 시 만약 예외가 발생한다면, 기본 또는 커스터마이징된 ErrorDecoder를 거치게 됩니다.

예외가 발생한 경우 재시도 처리를 고려해볼 수 있을 것입니다. Spring Cloud는 재시도 처리에 대한 인터페이스를 제공하고, Resilience4j 는 Retry 의 구현체를 제공합니다.

만약 Spring Cloud Retry 설정도 적용되어 있다면 재시도 처리와 ErrorDecoder 중 어떤 것이 더 먼저 처리되는지 살펴보도록 하겠습니다.

FeignClient에서 요청이 실패할 경우의 동작 순서는 다음과 같습니다

### 1. 오류 발생

API 호출 결과 HTTP 오류(예: 4xx, 5xx)가 발생하면 FeignClient은 먼저 해당 응답을 받습니다.

### 2. ErrorDecoder 처리

받은 응답은 ErrorDecoder의 decode 메서드를 통해 처리됩니다.

이때 ErrorDecoder는 응답에 따른 적절한 예외(Exception)를 생성합니다.

만약 커스터마이징된 ErrorDecoder 빈이 존재한다면, ErrorDecoder 인터페이스의 기본 구현체(Default)를 대체합니다.

### 3. RetryableException 여부에 따른 처리

- 만약 ErrorDecoder가 반환하는 예외가 RetryableException인 경우, Feign에 설정된 Retryer가 해당 예외를 감지하여 재시도 로직을 실행합니다.
- 반면, RetryableException이 아닌 일반 Exception이라면, 재시도 없이 호출이 실패한 것으로 처리됩니다.

### ErrorDecoder.Default

ErrorDecoder 인터페이스 내부에 중첩 클래스로 기본 구현체인 Default 클래스가 존재합니다.

Default 의 decode 메서드를 살펴보겠습니다.

```java
public static class Default implements ErrorDecoder {
    private final RetryAfterDecoder retryAfterDecoder = new RetryAfterDecoder();
    
    ...

    public Exception decode(String methodKey, Response response) {
        FeignException exception = FeignException.errorStatus(methodKey, response, this.maxBodyBytesLength, this.maxBodyCharsLength);
        Long retryAfter = this.retryAfterDecoder.apply((String)this.firstOrNull(response.headers(), "Retry-After"));
        return (Exception)(retryAfter != null ? new RetryableException(response.status(), exception.getMessage(), response.request().httpMethod(), exception, retryAfter, response.request()) : exception);
    }

    private <T> T firstOrNull(Map<String, Collection<T>> map, String key) {
        return map.containsKey(key) && !((Collection)map.get(key)).isEmpty() ? ((Collection)map.get(key)).iterator().next() : null;
    }
}
```

- `decode` 메서드의 return 구문을 살펴보면 최종적으로 `RetryableException` 예외를 반환하는 지, 아니면 다른 Exception 을 반환하는지에 따라 재시도 처리가 결정됩니다.
- 만약 `Retry` 가 적용된 상태에서 `RetryableException` 예외가 발생하면, 재시도 처리가 수행됩니다.
- `Long` 타입의 `retryAfter` 변수는 클라이언트에게 재시도를 하기 전에 얼마나 기다려야 하는지(예: 초 단위)를 알려주는 정보입니다. 해당 정보는 응답 헤더에 담겨서 전달됩니다.


# 📌 요약 

- Feign Logger로 상세 로그를 기록하고, Custom ErrorDecoder로 사용자 정의 예외 처리를 구현함으로써 서비스의 신뢰성과 유지보수성을 향상시킬 수 있음.
- FeignClient 요청 실패 시 먼저 ErrorDecoder를 거쳐 예외가 생성되고, 그 예외가 RetryableException인 경우에만 재시도 메커니즘이 동작.

# 참고 문헌
- [Spring Cloud OpenFeign](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/)
- [Feign Logging Configuration](https://www.baeldung.com/java-feign-logging)

---
