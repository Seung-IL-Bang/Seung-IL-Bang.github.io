---
title: Zipkin을 활용한 마이크로서비스 분산 트레이싱
description:
categories:
- Spring Cloud
- Zipkin
- MSA

tags:
- Distributed Tracing
- b3 propagation
- Micrometer

---

> 분산 환경에서 Zipkin을 활용한 요청의 전체 흐름을 추적해봅시다!

<!-- more -->

# 📌 들어가기

마이크로서비스 아키텍처는 독립적으로 배포되고 관리되는 작은 서비스들이 모여 애플리케이션을 구성하게 됩니다. 이러한 환경에서는 서비스 간의 통신이 빈번하게 발생하며, 각 서비스가 독자적인 로그를 남기기 때문에 전체 트랜잭션의 흐름을 파악하기가 어렵습니다. 예를 들어, 주문 서비스 -> 결제 서비스 -> 배송 서비스 등의 여러 서비스가 순차적으로 호출되는 과정에서 각 서비스는 별도의 로그를 각자 남기기 때문에 어떤 서비스에서 문제가 발생했는지 확인하기 어렵습니다.

이러한 문제점 때문에 **분산 트레이싱**은 문제 발생 시 원인을 신속하게 파악하고, 성능 병목을 찾아내는 데 필수적인 도구로 등장합니다.

본 포스트에서는 분산 트레이싱 도구 중 `Zipkin`을 활용한 분산 트레이싱을 다루도록 하겠습니다.

# 마이크로서비스 환경의 복잡성 

마이크로서비스 아키텍처에서는 **하나의 사용자 요청이 여러 서비스를 거치며 처리**됩니다. 주문, 결제, 배송 서비스 등 여러 서비스를 거치게 되는 것이죠. 이러한 **분산된 호출 구조**는 여러 **문제점**을 유발할 수 있습니다.

각 서비스마다 **별도의 로그**를 남기기 때문에 하나의 트랜잭션이 어디에서 지연되거나 장애가 발생했는지 확인하기 어렵습니다. 그리고 전체 흐름에서 어느 부분이 응답 시간을 지연시키는지도 파악하기 어렵습니다.

**모놀리식(Monolihic)**의 경우 하나의 서비스로 애플리케이션이 구동되기 때문에, 전체 흐름에 대한 로그를 하나의 서비스에서 디버깅 할 수 있습니다. 하지만 **MSA**에서는 문제를 파악하기 위해 관련된 모든 서비스들의 로그를 하나하나 다 살펴봐야 하는 번거로움이 존재합니다.

만약 분산 트레이싱 도구가 없다면, 위와 같은 문제를 해결하기 위해 각 서비스에서 개별적으로 로그를 분석해야 하며, 이는 시간 소모적이이고 비효율적일 뿐만 아니라, 문제 진단의 어려움이 따르기 때문에 신속하게 대응하지 못할 위험이 높아집니다.

# Zipkin을 활용한 요청 추적 

`Zipkin`은 **오픈 소스 분산 트레이싱 시스템**으로, 마이크로서비스 환경에서 서비스 간 호출 흐름을 **시각화하고 분석할 수 있도록 도와주는 도구**입니다. `Zipkin`을 활용하면 각 서비스의 호출 데이터를 중앙에서 수집하여 아래 문제점들을 해결해줍니다.

1. **전체 호출 흐름 시각화**: 사용자 요청이 어떤 경로로 전달되는지 시각화하여 한 눈에 파악 할수 있도록 해줍니다.
2. **Latency 분석**: 각 서비스 간 호출의 응답 시간 정보를 제공하여 어떤 서비스의 어떤 로직에서 성능 병목이 일어나는지 쉽게 식별할 수 있습니다.
3. **장애 원인 분석**: 트랜잭션 중 발생한 예외나 오류를 신속하게 탐지하고, 어느 서비스에서 문제가 발생했는지 분석 할 수 있습니다.

# Zipkin 적용 방법

예전에는 Spring Boot 애플리케이션에 쉽게 분산 트레이싱 기능을 추가할 수 있도록 도와주는 `Spring Cloud Sleuth`라는 라이브러리를 사용했었습니다. 하지만 최근 버전으로 업데이트 되면서 `Spring Cloud Sleuth`는 Deprecated 될 라이브러리가 되었습니다. 따라서 최근 버전에서는 `Micrometer`로 분산 트레이싱 기능을 추가하도록 변경되었습니다.

## Dependency

```groovy
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'io.micrometer:micrometer-observation'
	implementation 'io.micrometer:micrometer-tracing-bridge-brave'
	implementation 'io.zipkin.brave:brave-instrumentation-spring-web'
	implementation 'io.zipkin.reporter2:zipkin-reporter-brave'
```

## application.yml

```yaml
spring:
  application:
    name: product-service
management:
  tracing:
    sampling:
      probability: 1.0
    propagation:
      type: b3
  zipkin:
    tracing:
      endpoint: ${MANAGEMENT_ZIPKIN_TRACING_ENDPOINT:http://localhost:9411/api/v2/spans}
```

- 여기서 `spring.application.name`은 서비스 태깅에 활용되며, 각 서비스의 이름이 **Zipkin UI**에 표시되어 호출 흐름을 쉽게 파악할 수 있습니다.
- 위와 같이 의존성과 설정을 마치면, 자동으로 **각 요청에 고유한 Trace ID와 Span ID를 부여하여, 서비스 간의 호출 관계를 추적**하게 됩니다.

# Zipkin UI를 활용한 트레이스 분석 

## 서비스 간 호출 흐름 시각화

**Zipkin UI**는 수집된 트레이스 데이터를 기반으로, 서비스 간 호출 흐름을 직관적으로 시각화해줍니다. 개발자는 UI를 통해 각 서비스가 호출한 순서와 관계를 그래픽으로 확인할 수 있습니다. 또한 특정 트랜잭션의 세부적인 스팬 정보와 타임라인을 분석할 수도 있습니다.

## Latency 분석 및 장애 감지 

Zipkin UI에서는 각 스팬의 응답 시간을 시각적으로 표시하여, 어느 부분에서 지연이 발생했는지 쉽게 식별할 수 있습니다. 이를 통해 성능 병목 구간을 찾아내고, 장애 발생 시 원인 분석에 필요한 정보를 제공합니다. 만약 이러한 시각화 도구가 없으면, 로그 파일만으로 각 서비스 간의 호출 관계와 지연 시간을 분석해야 하므로, 문제 발생 시 즉각적인 대응이 어려워질 것입니다.

# Spring AOP를 활용한 미들웨어 간의 요청 추적

마이크로서비스에서는 HTTP 호출 외에도 Kafka, Redis와 같은 미들웨어를 활용하는 경우가 많습니다. Zipkin과 Micrometer의 기본 의존성과 설정만으로는 미들웨어 통신 경로를 자동 추적하진 않습니다. 미들웨어간의 호출 흐름까지 추적하기 위해, 추적에 필요한 로직을 **스프링 AOP**를 활용하여 **사용자 정의 트레이싱** 코드를 삽입할 수 있었습니다.

예를 들어, 스프링 AOP를 활용하여 Kafka를 통한 메시지 발행/구독 흐름도 추적할 수 있습니다. 이를 통해 메시지 큐를 통한 비동기 호출도 명확하게 추적할 수 있으며, 메시지 손실이나 지연 문제를 쉽게 추적할 수 있습니다.

만약 이러한 미들웨어 추적 메커니즘이 없다면, 각 미들웨어 간의 호출 흐름이 단절되어 문제 발생 시 원인 파악에 큰 어려움이 따를 것입니다. 이는 디버깅 하는 시간이 급격히 증가할 수 있습니다.

# 🚀 결론 

마이크로서비스 아키텍처에서는 서비스 간 호출 흐름이 복잡해짐에 따라, 단순 로그 분석만으로는 문제의 원인을 파악하기 어렵습니다. `Zipkin`과 `Micrometer`를 활용하면, 전체 트랜잭션의 흐름을 시각화하고 각 서비스의 성능을 모니터링할 수 있어 문제 발생 시 신속하게 대응할 수 있습니다. 이러한 도구들이 없다면, 시스템 장애나 성능 병목을 찾아내는 데에 많은 시간과 리소스가 소모되며, 이는 비즈니스에 심각한 영향을 미칠 것입니다. MSA 환경에서 `Zipkin`을 통한 **분산 트레이싱**은 단순히 로그를 남기는 것을 넘어, 시스템의 **전체적인 건강 상태를 관리**하고, 장애 발생 시 **신속한 원인 분석 및 대응** 전략을 마련하는 데 **필수적인 요소**라 볼 수 있습니다.

# 참고 자료

- [LINE 광고 플랫폼의 MSA 환경에서 Zipkin을 활용해 로그 트레이싱하기](https://engineering.linecorp.com/ko/blog/line-ads-msa-opentracing-zipkin)
- [b3-propagation](https://github.com/openzipkin/b3-propagation)



---
