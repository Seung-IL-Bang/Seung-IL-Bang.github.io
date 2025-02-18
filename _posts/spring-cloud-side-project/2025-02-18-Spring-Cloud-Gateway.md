---
title: Spring Cloud Gateway를 활용한 API Gateway 구축
description: 
categories:
- Spring Cloud
- MSA

tags:
- Spring Cloud Gateway

---

> Spring Cloud Gateway의 역할, 라우팅 및 필터 처리 방법에 대해 알아봅시다!

<!-- more -->

# 📌 들어가기

## Gateway란 무엇인가?

Gateway는 클라이언트와 내부 시스템(ex: 마이크로서비스) 간의 **단일 진입점 역할**을 하는 미들 웨어입니다.

모든 외부 요청은 먼저 Gateway를 통해 들어오며, 이곳에서 **인증, 인가, 로깅, 모니터링 등 공통 기능을 처리**한 후 적절한 내부 서비스로 요청을 전달합니다.

이를 통해 **개별 서비스는 본연의 비즈니스 로직에 집중**할 수 있게 됩니다.

## 왜 Spring Cloud Gateway를 사용해야 할까?

1. Spring Cloud Gateway는 Spring Boot, Spring Security 등과 자연스럽게 연동되므로, 기존 Spring 기반 시스템에 쉽게 도입할 수 있습니다.

2. **Netty** 기반의 `Spring Cloud Gateway`는 비동기, 논블로킹 방식으로 동작하여 높은 동시성 처리가 가능하며, 성능과 확장성 면에서 우수합니다.

3.  `application.yml` 설정 파일에 **선언적** 설정을 통해 **쉽게 라우팅 규칙을 정의하고, 전/후 처리 필터를 적용**할 수 있습니다. 이는 다양한 요구사항에 맞춰 Gateway를 유연하게 구성할 수 있게 해줍니다.

이와 같이 Spring Cloud Gateway는 Spring 생태계와의 원활한 통합을 지원하며, 마이크로서비스 환경에서 발생할 수 있는 다양한 문제(코드 중복, 보안 취약점, 성능 및 확장성 문제 등)를 효과적으로 해결해줍니다.

# 기본 라우팅 설정

application.yml 파일에 선언적 라우팅 설정을 통해, 요청 URL에 따라 내부 서비스로 요청을 전달합니다.

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://localhost:8080 # lb://USER-SERVICE
          predicates:
            - Path=/user-service/test/**
            - Method=GET
```

- 위 게이트웨이 라우팅 설정은 HTTP 메서드가 GET 요청이고, 경로가 `/user-service/test/**`에 매칭되는 경우 `http://localhost:8080` 에서 실행중인 `user-service` 로 라우팅해줍니다.
- 만약 디스커버리 서비스인 Eureka를 사용한다면, 라우팅 URI를 `lb://USER-SERVICE` 처럼 서비스 이름으로 작성할 수 있습니다. (`lb://{application.name}`)

# 전역 필터

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      default-filters:
        - name: MyGlobalFilter
          args:
            preFilter: true
            postFilter: true
```

- `spring.cloud.gateway.default-filters`: 전역 필터 클래스를 지정해줄 수 있습니다. 이때 필터 객체에 인수(`args`)를 전달해줄 수 있습니다.
  - `name`: 전역 필터 클래스명
  - `args`: 명시된 인수값들이 전역 필터 내부의 중첩 클래스인 `Config`와 매핑된 후, 필터 기능을 하는 `apply` 메서드의 인자로 전달됩니다. (코드를 보면 쉽게 이해하실 수 있을 겁니다.)
- 💡 참고로 필터에 인자 값을 전달하려면 필터에 `name` 속성값을 적용해야 합니다. 그 다음 아랫줄에 `args` 를 작성하면 됩니다. 

## MyGlobalFilter

Spring Cloud Gateway에서 동작하는 필터 클래스는 만들기 위해서는 `AbstractGatewayFilterFactory`를 상속하고 `apply` 메서드를 오버라이딩 해줘야 합니다.

그리고, `args`를 전달 받을 중첩 클래스(`MyGlobalFilter.Config`)를 부모 클래스의 제네릭으로 전달 해줍니다.

```java
@Slf4j
@Component
public class MyGlobalFilter extends AbstractGatewayFilterFactory<MyGlobalFilter.Config> {

    public MyGlobalFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // Pre Filter
            if (config.isPreFilter()) log.info("Global Pre Filter executed");

            // Post Filter
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                if (config.isPostFilter()) log.info("Global Post Filter executed");
            }));
        };
    }

    @Data
    public static class Config {
        private boolean preFilter;
        private boolean postFilter;
    }
}
```

- `MyGlobalFilter` 클래스 내부의 중첩 클래스인 `Config` 객체에 `application.yml` 에서 설정한 `args` 를 전달받게 됩니다.
- `apply` 메서드 내부에서 `return` **전 영역은 PreFilter** 이고, `return` **구문은 PostFilter 영역**이다.
- 요청을 라우팅 하기 전에 PreFilter가 동작하고, 요청된 라우팅이 마이크로서비스에서 처리를 마친 후 응답이 돌아오면 그제서야 PostFilter가 동작합니다.


# Custom Filter

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      default-filters:
        - name: MyGlobalFilter
          args:
            preFilter: true
            postFilter: true
      routes:
        - id: user-service
          uri: http://localhost:8080
          predicates:
            - Path=/user-service/**
          filters:
            - RemoveRequestHeader=Cookie
            - RewritePath=/user-service/(?<segment>.*), /$\{segment}
            - `MyCustomFilter`
            - name: MyLoggingFilter
              args:
                baseMessage: MyLoggingFilter start
                preLogger: true
                postLogger: true
```

- `spring.cloud.gateway.routes.filters`: 해당 속성에 선언된 필터들은 라우팅 조건을 만족하는 요청에 대해서만 필터가 작동합니다. 즉, 다른 라우팅 조건을 만족하는 요청에 대해선 필터가 동작하지 않습니다.
- `RemoveRequestHeader`, `RewritePath`와 같이 기본적으로 스프링에서 지원하는 선언적 설정도 존재하고, `MyCustomFilter`, `MyLoggingFilter`와 같이 개발자가 정의한 커스텀 필터를 지정할 수도 있습니다.

> ✅ 설정 파일(application.yml) 내에서 나열된 라우트(route)나 각 라우트에 정의된 필터(filters)는 선언한 순서(위에서 아래)대로 처리되는 것이 일반적입니다. 즉, 별도의 order 속성을 지정하지 않는 경우에는 위에서 아래로 우선순위가 적용됩니다.

# Discovery Service와 Spring Cloud Gateway

Spring Cloud Netflix Eureka를 사용 환경이라면, Spring Cloud Gateway는 **로드밸런서**의 역할도 수행할 수 있습니다.

앞서 살펴봤던 `user-service` 의 Uri 경로를 `lb://USER-SERVICE` 처럼 애플리케이션 네임을 적어주면 디스커버리 서비스로부터 해당 서비스의 위치를 탐색하여 로드밸런싱 할 수 있습니다.

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/user-service/test/**
            - Method=GET
```

이때, 만약 user-service 의 인스턴스가 다중화 되어 있다면, Spring Cloud Gateway는 라운드 로빈(Round Robin) 알고리즘으로 로드밸런싱을 처리하게 됩니다.

이와 같이 Spring Cloud Gateway는 단순한 API 게이트웨이 기능을 넘어, 서비스 디스커버리와 통합되어 여러 인스턴스 간의 로드밸런싱 기능을 제공합니다.

따라서 시스템의 부하를 효과적으로 분산시켜, 서비스의 가용성과 안정성을 높여주고 마이크로서비스 아키텍처에서의 운영 효율성을 크게 향상시켜 줍니다.


# 🎯 요약

- 클라이언트와 내부 시스템 간의 단일 진입점 역할을 수행하며, 인증, 인가, 로깅, 모니터링 등 공통 기능을 중앙집중적으로 처리.
- Spring 생태계와의 원활한 통합, Netty 기반의 비동기 논블로킹 처리, 선언적 설정을 통한 유연한 라우팅 및 필터 관리 등으로 높은 성능과 확장성을 제공.
- Spring Cloud Netflix Eureka와 연동하여, lb:// 접두사를 사용함으로써 동적 로드밸런싱(라운드 로빈 알고리즘)을 수행, 여러 인스턴스 간의 부하 분산과 시스템 안정성을 확보


# 📂 참고 자료

- [Spring Cloud Gateway Docs](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/how-it-works.html)
- [Netty](https://en.wikipedia.org/wiki/Netty_(software))



---