---
title: Spring Cloud Netflix - Service Discovery Eureka
description: 
categories:
- Spring Cloud
- MSA

tags:
- Service Discovery
- Eureka


---

> 서비스 디스커버리가 무엇이고, Eureka가 무엇인지 알아봅시다!

<!-- more -->

# 📌 들어가기

서비스 간 통신을 하기 위해서는 서로의 위치(IP Address 혹은 도메인)를 알고 있어야 합니다.

MSA 환경에는 수 많은 서비스들이 존재할 수 있습니다.

그렇다면 한 서비스에서 다른 서비스들과 통신 하기 위해 다른 서비스들의 모든 주소들을 알아야 할까요?

Netflix 가 Spring Cloud 재단에 기증한 서비스 디스커버리(탐색) 기술인 `Eureka`는 모든 주소들을 기억해야 하는 문제점을 깔끔히 해결해 줍니다.

이번 포스트에서는 마이크로서비스 환경에서 서비스 등록과 탐색이 왜 중요한지, 그리고 동적 서비스 등록 및 로드밸런싱의 필요성에 대해 알아보겠습니다.

# 서비스 디스커버리란?

마이크로서비스 환경에서는 개별 서비스들이 독립적으로 배포되고 실행이 됩니다.

이때 서비스 인스턴스들의 위치(Host, Port etc)가 동적으로 변경되기 때문에, 클라이언트가 서비스의 위치를 고정된 주소로 참조하기 어려운 문제가 발생합니다.

**서비스 디스커버리는 클라이언트가 서비스의 실제 위치를 동적으로 탐색하고, 최신 정보를 반영할 수 있도록 하는 역할을 합니다.**

## 서비스 디스커버리 패턴

서비스를 동적으로 탐색하는 패턴에는 두 가지가 있습니다.

### Client Side Discovery

클라이언트가 직접 서비스 레지스트리에서 정보를 가져와 로드밸런싱을 수행합니다.

### Server Side Discovery

클라리언트의 요청이 로드 밸런서를 통해 전달되며, 이 로드 밸런서가 **서비스 레지스트리(Eureka Server)**에서 실제 서비스 인스턴스를 선택하게 됩니다.

`Spring Cloud Gateway` 는 로드 밸런싱 기능을 지원하기 때문에 서버 사이드 디스커버리 패턴을 사용할 경우 `Spring Cloud Gateway`를 활용할 수 있습니다.

게이트웨이는 유레카 서버로부터 서비스 인스턴스의 위치를 가지고와 직접 로드밸런싱을 수행합니다.

# Eureka Server 와 Client

`Eureka`는 **서버**와 **클라이언트** 두 개로 나뉘어 역할이 각각 다릅니다.

## Eureka Server

유레카 서버의 역할은 다음과 같습니다.

- 모든 서비스 인스턴스들의 등록 정보를 관리합니다.
- 주기적으로 서비스 인스턴스의 상태를 확인하여, 정상 동작하지 않는 인스턴스는 레지스트리에서 제거합니다.
- 클라이언트나 다른 서비스가 현재 사용 가능한 인스턴스 정보를 조회할 수 있도록 API를 제공합니다.

## Eureka Client

유레카 클라이언트의 역할은 다음과 같습니다.

- 애플리케이션이 기동되면서 자동으로 Eureka 서버에 자신의 정보를 등록합니다.
- 등록된 정보를 주기적으로 갱신하여, 서버 측에서 정상 동작 여부를 판단할 수 있도록 합니다.
- 필요한 경우 Eureka 서버에서 다른 서비스의 정보를 가져와, 로드 밸런싱 및 동적 호출에 활용합니다.

# Eureka 특장점

- 서비스 인스턴스가 늘어나거나 줄어들 때 자동으로 반영됩니다.
- 유레카 서버 이중화 구성을 통해 장애 발생 시에도 서비스 디스커버리 지속성이 보장됩니다.
- 등록 주기와 타임아웃을 활용하여, 정상 상태를 유지하지 않는 인스턴스들은 자동으로 제거되고, 다시 회복된 인스턴스는 다시 등록되는 복구 메커니즘이 있습니다.


# 동적 서비스 등록 메커니즘

서비스가 기동 시 Eureka Client가 Eureka Server에 HTTP 요청을 보내어 자신을 등록합니다.

이후 주기적으로 등록 정보를 갱신하며, 만약 갱신이 실패하면 일정 시간이 지난 후 자동으로 레지스트리에서 제거됩니다.

Eureka 대시보드(보통 `http://localhost:8761`)를 통해 현재 등록된 인스턴스 목록과 상태 정보를 확인하실 수 있습니다.


# Eureke와 결합되어 동작하는 Spring Cloud Gateway

`Spring Cloud Gateway`는 `Eureka`와 결합되어 로드 밸런서의 역할을 수행합니다.

클라이언트의 요청이 게이트웨이에 도달하게 되면, Eureka에서 조회한 인스턴스 목록을 바탕으로 **라운드 로빈** 방식으로 요청을 분산 처리합니다.


# Eureka 서비스 클러스터링 방법과 장점

단일 Eureka 서버로 운영하면 장애 시 전체 서비스 디스커버리 시스템이 마비될 수 있습니다.

따라서 여러 대의 Eureka 서버를 클러스터링하여 운영할 필요가 있습니다.

Eureka 서버를 클러스터링하게 되면 여러 Eureka 서버가 협력하여 서비스를 관리하기 때문에 특정 Eureka 서버의 장애가 전체 시스템에 미치는 영향을 최소화할 수 있습니다.

실제 운영 환경에서는 두 대 이상의 서버로 구성하면, 리버스 프록시 또는 로드 밸런서를 통해 어느 서버에 접근하든지 간에 서비스 디스커버리 기능을 수행할 수 있게 됩니다.

또한 다수의 Eureka 서버가 요청을 분산 처리함으로써, 높은 트래픽 환경에서도 안정적인 서비스를 제공할 수 있습니다.

## Eureka 서버 클러스터링 구성 예시

`application.yml` 설정 파일에 Eureka 서버 간의 동기화 방식 설정(`Peer Awareness`)을 `true` 값으로 설정하고, Eureka 서버들이 호스팅되고 있는 위치들을 `defaultZone` 에 작성해주면 됩니다.

```yaml
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false
  peer:
    awareness:
      enabled: true
    eureka:
      serviceUrl:
        defaultZone: http://localhost:8761/eureka/,http://localhost:8762/eureka/
```
- 동기화 방식 설정과 관련하여 [공식 문서](https://cloud.spring.io/spring-cloud-netflix/reference/html/#spring-cloud-eureka-server-peer-awareness)에서 더 자세히 확인하실 수 있습니다.


# 정리

마이크로서비스 환경에서 서비스 디스커버리의 필요성과 `Netflix Eureka`를 통한 인스턴스들의 동적 위치 탐색 방안에 대해 알아보았습니다. 

Eureka를 사용하면 서비스 인스턴스가 동적으로 등록되고 자동으로 관리됩니다. 

이로써 서비스 디스커버리는 클라이언트가 서비스의 실제 위치를 동적으로 탐색할 수 있게 해주어 서비스 통신이 원활하게 이루어지도록 해줍니다.

그리고 클러스터링을 통해 고가용성을 확보하는 방법에 대해서도 살펴보았습니다.

# 참고 자료

- [Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-netflix/reference/html/)

---
