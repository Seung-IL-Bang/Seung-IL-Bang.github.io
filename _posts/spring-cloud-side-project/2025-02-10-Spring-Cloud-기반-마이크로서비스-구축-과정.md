---
title: Spring Cloud 기반 마이크로 서비스 간 통신
description: 
categories:
- Spring Cloud
- MSA

tags:
- RestTemplate
- OpenFeign
- Kafka

---

> Spring Cloud를 활용한 마이크로 서비스 간 통신 방식에는 무엇이 있는지 살펴보겠습니다!

<!-- more -->

# 📌 들어가기

e-commerce 플랫폼을 주제로 MSA와 Spring Cloud 기반의 사이드 프로젝트를 진행했습니다.

모놀리식이 아닌 사용자, 주문, 상품, 결제, 배달 등 각 도메인 별로 MSA 기반으로 개발했습니다.

모놀리식의 경우 각 도메인 서비스별로 소통을 위해 네트워크를 거칠 필요가 없습니다.

이에 반해, MSA 기반의 마이크로 서비스들은 도메인 서비스끼리 통신을 하기 위해서 네트워크 통신이 필수이다.

다들 아시다시피, 대표적인 네트워크 통신에는 HTTP 통신이 있습니다.

현대 대부분의 통신은 HTTP을 이용한 REST API로 이뤄진다고 봐도 무리가 아니죠.

그 외에도 TCP 통신, gRPC 등 다양한 방식이 존재합니다.

저는 사이드 프로젝트를 진행하면서 마이크로 서비스 간 통신 방식을 여러 번 수정해왔습니다.

처음에는 `RestTemplate` 을 이용한 API 통신, 그 다음으로 Spring Cloud에서 지원하는 `OpenFeign` 을 통한 통신.

마지막 종착지로, 마이크로 서비스 간 종속성을 없애기 위한 `Kafka` 이벤트 기반 비동기 통신을 적용했습니다.

각 방식에는 장단점이 있으므로, 장단점을 이해하고 해결하고자 하는 문제에 어떤 방식이 최선일 지 결정하는 것이 중요합니다. **(개발자의 덕목)**

---

# API 소개

우선, 사이드 프로젝트에서 개발한 API들에 대해 먼저 소개 해드리겠습니다.

각각의 도메인 서비스들은 서로 API 통신을 통해 유기적으로 비즈니스 로직을 처리하게 됩니다.

그리고 각 서비스 별로 발생하는 데이터들은 각각 고유한 데이터베이스에 데이터를 저장하게 됩니다.

참고로, 해당 프로젝트의 핵심 목표는 분산 환경에서의 중앙 집중식 로깅 관리 및 트레이싱이기 때문에 각 서비스 별로 필요한 비즈니스 로직들에 비약한 부분이 존재할 수 있습니다.

## User Service

- 회원 가입
- 로그인
- 유저 정보 조회
- 유저 목록 조회

## Product Service

- 상품 목록 조회
- 재고 차감 
- 재고 증가

## Order Service

- 주문
- 주문 조회 
- 주문 목록 조회

## Payment Service

- 결제 
- 결제 취소

## Delivery Service

- 배송 접수
- 배송 

---

# RestTemplate 

`RestTemplate`은 Spring 에서 기본으로 제공하는 HTTP 요청을 보낼 때 사용하는 도구입니다.

## 장점

- 간단하고 직관적이다.
- 동기 방식이기 때문에 흐름을 따라가기 쉽다.

## 단점

- 요청을 보낸 후 응답을 받을 때까지 대기한다. (try-catch로 예외 처리 + 타임아웃 설정이 필수 ⚠️)
- 부하가 큰 시스템에서는 병목 현상이 발생할 수 있다.
- 예외 발생 시 장애가 전파됨.

## 예제 코드

```java
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {
    private final RestTemplate restTemplate;
    ...

    @Override
    public UserDto getUserByUserId(String userId) {
      ...
      ResponseEntity<List<ResponseOrder>> ordersResponse = restTemplate.exchange(
        "http://example.com/orders/" + userId,
        HttpMethod.GET,
        null,
        new ParameterizedTypeReference<>() {}
      );
    }
}
```

- RestTemplate 을 사용한 코드 일부입니다. (사용자의 주문 목록을 조회하는 요청)
- 몇 개의 인자를 전달함으로써 간단하게 HTTP 요청을 보낼 수 있습니다.
  - 요청 URL
  - HTTP 메서드
  - HttpEntity (요청 본문 및 헤더)
  - Class<?> responseType (응답 타입)
- 🚨 다만, 동기 방식(Synchronous)으로 동작하기 때문에, 특정 서비스가 중단되거나 응답이 지연되면 요청을 보낸 서비스도 영향을 받는다는 것을 주의해야 합니다.
---

# OpenFeign

`OpenFeign`은 HTTP 요청을 보내는 작업을 자동으로 해주는 도구입니다. `RestTemplate`과 비슷하지만, 인터페이스만 만들면 요청 코드가 자동으로 생성되는 편의성을 제공합니다.

## 장점

- 코드가 간결함 (인터페이스 정의만 하면 됨)
- 내부적으로 로드 밸런싱 가능 (Eureka와 연동 시 URL 필요 없음)
- Feign + Resilience4j로 Circuit Breaker(회로 차단기) 적용 가능. (Fallback 기능 포함)
- Spring Cloud 생태계와 연동이 쉬움.

## 단점

- 기본적으로 동기 방식이므로 응답을 기다려야 함.
- 추가적인 설정이 필요할 수도 있음 (Feign 인터셉터, 로깅 등)
- 예외 발생 시 장애가 전파됨.

## 예제 코드

```java
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {
    private final OrderServiceClient orderServiceClient;
    ...

    @Override
    public UserDto getUserByUserId(String userId) {
      ...
      List<ResponseOrder> orderList = orderServiceClient.getOrders(userId);
    }
}
```

```java
@FeignClient(name = "order-service")
public interface OrderServiceClient {

    @GetMapping("/orders/{userId}")
    List<ResponseOrder> getOrders(@PathVariable("userId") String userId);

}
```
- OpenFeign 을 사용할 경우 인터페이스만 구현하면 되기 때문에 RestTemplate 보다 더 간결하고 직관적이다.
- Eureka(디스커버리 서비스)와 연동 시, 서비스의 이름만으로 요청을 보낼 수 있다.


---

# Kafka 

---
