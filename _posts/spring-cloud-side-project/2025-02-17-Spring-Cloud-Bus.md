---
title: Config Server와 Spring Cloud Bus를 통한 동적 설정 관리
description:
categories:
- Spring Cloud
- MSA

tags:
- Spring Cloud Config
- Spring Cloud Bus

---

> 분산 환경에서 각 마이크로서비스가 Config Server로부터 설정 파일을 읽어오는 방법을 알아봅시다!

<!-- more -->

# 📌 들어가기

이전에 Spring Cloud Config Server에 관한 내용을 다룬 포스트가 있습니다.

이번 포스트에서는 Config Server를 사용하는 환경에서, `Spring Cloud Bus`와 `Spring boot Actuator`를 이용하여 애플리케이션을 재시작하지 않고도 설정 변경을 실시간으로 반영하는 방법에 대해 탐구해보겠습니다!

## 실시간 설정 변경이 왜 중요한가?

분산된 마이크로서비스 환경에서는 각각의 서비스가 개별적으로 재시작 없이 설정을 업데이트해야 운영 효율성이 높아집니다.

동적 설정 변경을 통해 서비스 중단 없이 빠르게 환경 변화에 대응할 수 있기 때문입니다.

이로 인해 설정 값들의 일관성을 쉽게 유지할 수 있게 됩니다.

## 해당 기술이 없다면?

설정을 변경할 때마다 각 서비스마다 재시작해야 하는 번거로움이 존재합니다. 이 과정에서 오타나, 잘못된 값이 설정된다면 또 다시 재시작 과정을 반복해야 하죠.

또한 변경 사항이 즉각적으로 반영되지 않기 때문에(재시작으로 인해), 설정 불일치로 인한 오류가 발생할 수 있습니다.

마지막으로, 분산 환경에서 개별적으로 설정을 관리하게 된다면 유지보수의 비용이 크게 증가할 것입니다.

> Config Server를 사용한다면 애플리케이션을 재시작하지 않고도 설정 변경을 실시간으로 반영할 수 있도록 지원하는 Spring Cloud Bus, Spring Boot Actuator 기술 도입이 필수라고 생각합니다.


# 마이크로서비스와 Config Server 연동하기

클라이언트(마이크로서비스)는 Config Server에서 설정 파일을 읽어올 수 있어야 하기 때문에, Config Server와 연결을 진행해야 합니다.

## Dependency

```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-config'
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

- Config Server와 연결을 위해 `spring-cloud-starter-config` 의존성을 추가해줍니다.
- Config Server에게 설정 파일 조회 또는 갱신 요청을 위해 `actuator` 의존성을 추가해줍니다.
- actuator의 엔드포인트로 `/refresh`, `/busrefresh` 를 사용할 예정입니다.

## application.yml 

```yaml
spring:
  cloud:
    config:
      name: # 읽어올 파일명
  config:
    import: optional:configserver:http://localhost:8888
  profiles:
    active: # 프로파일명
    
management:
  endpoints:
    web:
      exposure:
        include: refresh, busrefresh
```

- `spring.cloud.config.name`: Config Server로 부터 읽어올 파일명을 지정해줍니다.
- `spring.config.import`: Config Server 연결 URL을 지정해줍니다. (로컬 테스트: Config Server가 port 번호 8888로 실행 가정)
- `spring.profiles.active`: 읽어올 파일의 프로파일을 지정할 수 있습니다. (기본 프로파일: `default`)
- `management.endpoints.web.exposure.include`: actuator에서 활성화시킬 엔드포인트 입니다.
  - `/refresh`: 개별 클라이언트만 설정을 갱신합니다.
  - `/busrefresh`: Bus(RabbitMQ 또는 Kafka)에 연결된 모든 클라이언트의 설정을 갱신합니다.

여기까지 설정을 마치면, `/refresh` 엔드포인트 요청으로 Config Server로 부터 설정 파일을 읽어와서 클라이언트의 설정을 동적으로 실시간 변경할 수 있게 됩니다.

하지만, 이것은 개별 클라이언트에만 변화가 생깁니다. 

만약 변경된 설정 값을 여러 마이크로서비스들에게 공통적으로 동시에 적용하고 싶다면 어떡해야 할까요?

개별 클라이언트마다 `/refresh` 요청을 날려야 할까요? (아마도 매우 번거롭고 귀찮은 작업일 것입니다.)

이럴 때 사용하는 것이 `Spring Cloud Bus` 입니다.

# Spring Cloud Bus란?

`Spring Cloud Bus`는 **메시지 브로커(RabbitMQ, Kafka 등)를 이용해 여러 마이크로서비스 간에 설정 변경 이벤트를 실시간으로 전파**합니다.

이제 `Spring Cloud Bus` 도 추가 설정하여 동시에 여러 마이크로서비스들의 설정 값들을 실시간으로 반영하도록 해봅시다.

Bus 작동 테스트를 위해 **RabbitMQ**와 **Kafka** 두 개를 이용하여 테스트를 진행해봤습니다.

두 메시지 브로커 모두 Bus를 쉽게 구성할 수 있었습니다.

하지만 백엔드 개발자라면, 기술을 채택하는 데 있어 근거를 가지고 있어야겠죠?

결과적으로 저는 개인 사이드 프로젝트를 위해 Kafka를 적용하긴 했습니다만, 두 메시지 브로커 설정 방식에 대해 모두 설명 해드리겠습니다.

## Bus로 사용하기 적합한 메시지 브로커?

설정 방식을 알아보기 전에, Spring Cloud Bus의 메시지 브로커로 RabbitMQ와 Kafka를 선택할 때 고려할 수 있는 장단점을 비교해보겠습니다.

### RabbitMQ

**장점**

- 비교적 간단하게 설치 및 운영할 수 있으며, 관리 UI도 직관적입니다.
- AMQP를 기반으로 하여 Pub/Sub, Work Queue, RPC 등 다양한 패턴을 쉽게 구현할 수 있는 **유연한 메시징 패턴**을 제공합니다.
- 일반적인 애플리케이션 환경에서는 충분한 성능을 제공하며, **소규모 시스템에 적합합니다.**

**단점**

- 대용량 데이터 처리에는 적합하지 않습니다.
- Kafka에 비해 수평 클러스터 확장성이 떨어집니다.

### Kafka

**장점**

- **대용량 메시지를 빠르게 처리**할 수 잇으며, **클러스터 확장이 용이**합니다.
- 메시지를 디스크에 지속적으로 저장하여 **내구성을 제공**합니다.
- **스트리밍 데이터 처리에 최적화**되어 있습니다.

**단점**

- 초기 설정과 운영 및 모니터링이 RabbitMQ보다 복잡합니다.
- 메시지 시스템의 개념과 운영 전략에 대한 Kafka 학습 비용이 필요합니다. 
- 단순한 시스템에 도입하기에는 오버엔지니어링이 될 수 있습니다.

### 선택 기준 정리

- 소규모 시스템이나 간단한 메시징 패턴이 필요하다면 **RabbitMQ**가 빠르고 쉽게 도입할 수 있는 선택입니다.
- 대규모 분산 환경이나 높은 처리량, 내구성이 요구되는 경우에는 **Kafka**가 적합합니다.

두 메시지 브로커 모두 Spring Cloud Bus를 활용한 동적 설정 변경을 지원하므로, 시스템 규모와 메시지 처리량, 확장성을 고려하여 적절한 브로커를 선택하시면 됩니다.

---

# Spring Cloud Bus 설정

## Kafka를 이용한 Bus

### Dependency
```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-bus-kafka'
```
- 클라이언트가 Kafka를 통해 Bus 이벤트를 수신 및 발신할 수 있도록 의존성 추가해줍니다.

### 클라이언트 application.yml
```yaml
spring:
  kafka:
    bootstrap-servers: http://localhost:9092 # Kafka URL
  cloud:
    bus:
      destination: bus-topic # Bus 이벤트를 주고받을 토픽
```

- `spring.kafka.bootstrap-servers`: Bus로 이용할 Kafka 브로커 서버의 URL (클러스터링이 된 경우 `,`를 기준으로 여러 브로커 URL을 명시하면 됩니다.)
- `spring.cloud.bus.destination`: Bus 이벤트를 Pub/Sub 할 토픽을 명시해줍니다.
- ⚠️ 위 설정은 **Config Server도 마찬가지로 동일하게 설정**되어 있어야 합니다!

```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh, busrefresh
```
- 클라이언트의 `actuator` 엔드포인트 `/busrefresh`를 이용하면 메시지 브로커의 설정된 토픽으로 이벤트가 발송됩니다.
- 해당 토픽을 Consume(소비/구독)하는 클라이언트들은 모두 Config Server로 부터 설정 파일의 갱신을 요청하게 됩니다.
- 이로써 동시에 여러 마이크로서비스들의 설정 값을 실시간으로 업데이트 할 수 있게 됩니다.

## /busrefresh

<figure align="center">
<img src="/post_images/spring-cloud-side-project/config-server3.png">
<figcaption>/busrefresh 성공 응답으로 204(no-content) 응답이 옵니다.</figcaption>
</figure>

## bus topic Log

`/busrefresh` 이벤트가 발생하여 Kafka 토픽에 메시지가 도달하면 아래와 같은 **JSON 포맷**으로 기록되는 것을 확인하실 수 있습니다.

`type` 필드와 `originService` 를 살펴보면 어떤 클라이언트에서 이벤트를 발송했고, 어떤 클라이언트가 이벤트를 수신했는지 확인할 수 있습니다.

- `type`: 어떤 클라이언트에서 `/busrefresh` 요청이 발생했고, 어디서 그 이벤트를 수신했는지 확인할 수 있습니다.
  - `RefreshRemoteApplicationEvent`: `/busrefresh`를 호출했다는 이벤트입니다.
  - `AckRemoteApplicationEvent`: `/busrefresh`를 요청에 대한 확인 응답 이벤트입니다.
- `originService`: 이벤트를 처리한 클라이언트(마이크로서비스)의 애플리케이션 이름 정보가 담겨있습니다.

```json
[
  {
    "type": "RefreshRemoteApplicationEvent",
    "timestamp": 1738394015516,
    "originService": "order-service:default:8082:48ea6afcb8f6479b592a8cf041d2927b",
    "destinationService": "**",
    "id": "d52e5139-9137-4371-8da9-f5152a3ed8f0"
  },
  {
    "type": "AckRemoteApplicationEvent",
    "timestamp": 1738394015600,
    "originService": "order-service:default:8082:48ea6afcb8f6479b592a8cf041d2927b",
    "destinationService": "**",
    "id": "f7c31aa1-a1df-4952-9e6c-a2ddd51c65f9",
    "ackId": "d52e5139-9137-4371-8da9-f5152a3ed8f0",
    "ackDestinationService": "**",
    "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"
  },
  {
    "type": "AckRemoteApplicationEvent",
    "timestamp": 1738394018002,
    "originService": "config-service:8888:5cadfc3cecb50346571c4e594886d273",
    "destinationService": "**",
    "id": "ce0d38d8-bd5a-43ce-b9cf-ea96456265d1",
    "ackId": "d52e5139-9137-4371-8da9-f5152a3ed8f0",
    "ackDestinationService": "**",
    "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"
  },
  {
    "type": "AckRemoteApplicationEvent",
    "timestamp": 1738394021382,
    "originService": "user-service:default:8080:4938cf08c833b263a04aa56c247721d3",
    "destinationService": "**",
    "id": "5bb5898f-a570-475d-b6c6-3efa7516b4b6",
    "ackId": "d52e5139-9137-4371-8da9-f5152a3ed8f0",
    "ackDestinationService": "**",
    "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"
  },
  {
    "type": "AckRemoteApplicationEvent",
    "timestamp": 1738394022065,
    "originService": "product-service:8081:2dc75521f12081519f46be8f3441ee00",
    "destinationService": "**",
    "id": "949d77da-9861-4066-8aca-b9b3aaacb678",
    "ackId": "d52e5139-9137-4371-8da9-f5152a3ed8f0",
    "ackDestinationService": "**",
    "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"
  }
]
```

> 참고로 Kafka는 도커 이미지를 이용하여 구동하였습니다. 이번 포스트는 Kafka에 대한 포스트가 아니므로, Kafka 도커 실행에 대한 내용은 생략하도록 하겠습니다.

여기까지 Kafka를 이용한 Spring Cloud Bus 를 알아보았습니다.

## RabbitMQ를 이용한 Bus

이번에는 RabbitMQ를 이용한 Spring Cloud Bus 설정 방법을 알아보겠습니다.

동작 흐름은 Kafka와 동일하기 때문에, RabbitMQ는 설정법만 간단히 짚고 넘어가도록 하겠습니다.


```yaml
spring:
  application:
    name: user-service
  config:
    import: optional:configserver:http://localhost:8888
  profiles:
    active: default
  cloud:
    config:
      name: test
  rabbitmq:
    host: localhost # RabbitMQ Host
    port: 5672 # RabbitMQ Port
    username: guest 
    password: guest

management:
  endpoints:
    web:
      exposure:
        include: refresh, busrefresh
```

- Spring Cloud Bus를 위해서 Config Server와 연결되어 있어야 합니다.
- `/busrefresh` 를 호출하면 RabbitMQ에 메시지를 발송하여, RabbitMQ(Bus 역할)에 연결된 서비스들의 설정 파일들을 갱신시키게 됩니다.

> RabbitMQ는 도커 이미지를 이용하여 구동하였습니다.

# 🚀 정리

이번 포스트에서는 Spring Cloud Bus와 Spring Boot Actuator를 활용하여 Config Server에서 읽어온 설정 파일을 애플리케이션 재시작 없이 동적으로 갱신하는 방법을 다뤘습니다.

Kafka와 RabbitMQ 두 가지 메시지 브로커를 통한 Bus 설정 방법과 각각의 장단점에 대해서도 알아봤습니다.

`/refresh`와 `/busrefresh` 엔드포인트를 활용해 개별 서비스와 전체 마이크로서비스에 동시에 설정 변경 이벤트를 전파하는 과정도 살펴봤습니다.

# 🎯 요약

- Spring Cloud Bus와 Actuator를 사용하면 Config Server의 설정 변경을 각 마이크로서비스에 실시간으로 적용할 수 있습니다.
- Kafka는 대용량 처리와 확장성이 뛰어나고, RabbitMQ는 설치와 운영이 간편하다는 장점을 제공합니다.

---
