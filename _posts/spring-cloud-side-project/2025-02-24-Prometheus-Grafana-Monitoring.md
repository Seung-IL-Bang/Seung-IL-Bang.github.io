---
title: Spring Boot + Prometheus + Grafana로 마이크로서비스 모니터링 시스템 구축
description:
categories:
- MSA
- Monitoring
- Prometheus
- Grafana

tags:
- Prometheus
- Grafana

---

> Prometheus와 Grafana를 활용하여 Spring Boot 기반 마이크로서비스를 모니터링 하는 방법에 대해 알아봅시다!

<!-- more -->

# 📌 들어가기 

현대의 마이크로서비스 아키텍처는 여러 개의 독립적인 서비스가 유기적으로 협력하여 하나의 큰 시스템을 구성합니다. 마이크로서비스 환경에서는 각 서비스가 독립적으로 배포되고 운영되기 때문에, 특정 서비스의 장애나 성능 저하가 전체 시스템에 영향을 줄 수 있습니다. 만약 모니터링 시스템이 없다면 서비스 장애나 성능 저하 발생 시 원이 파악에 오랜 시간이 소요되어 비즈니스에 심각한 영향을 미칠 수 있습니다. 이런 서비스 장애가 지속되면 사용자 경험이 악화되고, 결과적으로 서비스의 신뢰를 잃어버리는 최악의 상황이 올 수 있습니다. 따라서 MSA 환경에서는 각 서비스의 상태와 성능을 실시간으로 모니터링하는 것이 필수적입니다. 

본 포스트에서는 `Prometheus`와 `Grafana`를 활용하여 Spring Boot 기반 마이크로서비스를 모니터링하는 방법에 대해 알아보도록 하겠습니다.

# Promethus & Grafana

우선 Prometheus와 Grafana가 각각 무엇인지 알아봅시다.

## Prometheus란?

`Prometheus`는 오픈소스 모니터링 및 경고 도구로, 주로 **시계열 데이터**를 수집하여 저장, 쿼리, 분석하는 데 사용되는 저장소입니다. 특히 시계열 데이터에 특화된 쿼리 언어인 **PromQL**을 제공하여, 수집한 메트릭을 조회할 수 있습니다.

## Grafana란?

`Grafana`는 `Prometheus`에서 수집한 데이터를 직관적인 대시보드 형태로 시각화하는 시각화 도구입니다. Prometheus 뿐만 아니라 다양한 데이터 소스와의 연동도 지원합니다. 대시보드를 커스텀할 수 있을 뿐만 아니라 경고 시스템도 구축 가능합니다. 

# Spring Boot와 Prometheus 연동 

Spring Boot는 **Actuator** 모듈을 통해 애플리케이션의 다양한 상태 정보(**metrics**)를 쉽게 노출할 수 있습니다. 이를 활용하여 Prometheus가 **Pull 방식**으로 메트릭 데이터를 수집할 수 있도록 설정할 수 있습니다.

Actuator 모듈을 통해 스프링 애플리케이션의 메트릭 정보들을 Prometheus에게 노출하기 위해 다음 의존성을 추가해줘야 합니다.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```

그리고, Prometheus가 Pull 방식으로 메트릭을 조회할 Actuator 엔드포인트를 활성화시켜 줘야 합니다.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
```

마지막으로 Promethues.yml 설정 파일을 통해 Prometheus가 Pull 하고자 하는 애플리케이션의 위치 정보를 설정해주어야 합니다.

아래 설정은 동일한 도커 네트워크 상에 존재하는 `user service`, `product service`, `order service`에 대한 메트릭을 Pull을 5초 간격으로 진행하겠단 설정입니다.

```yaml
global:
  scrape_interval: 15s
  external_labels:
    monitor: 'codelab-monitor'
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'user-actuator'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['user-service:8080']
      - targets: ['product-service:8081']
      - targets: ['order-service:8082']
```

만약 로컬에 설치된 Prometheus가 아닌 도커로 실행 시, Prometheus의 `/etc/prometheus/prometheus.yml` 경로와 커스텀 설정 파일을 볼륨 마운팅 해주어야 합니다.

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
```

위 설정이 완료되면 Prometheus는 `http://<호스트>:<포트>/actuator/prometheus` 엔드포인트를 통해 Spring Boot 애플리케이션들의 메트릭 데이터를 수집할 수 있습니다.

# Prometheus 메트릭 -> Grafana 시각화 

메트릭 데이터들이 Prometheus에 저장되면 이와 연동하여 Grafana를 통해 시각화할 수 있습니다. 

Grafana 관리 페이지에서 데이터 소스로 Prometheus를 추가한 다음, URL에 Prometheus 서버의 주소(`http://<prometheus-host>:9090`)를 입력합니다.

대시보드를 개인이 직접 커스터마이징할 수도 있지만, 이미 잘 만들어진 대시보드 템플릿을 가져다 사용하면 편리합니다. 



# 🚀 결론

**Prometheus**와 **Grafana**를 활용한 모니터링 시스템은 마이크로서비스 환경에서 서비스의 상태를 실시간으로 확인하고, 장애 발생 시 빠른 대응을 가능하게 합니다. 모니터링 시스템은 필수적이며, 이를 부재할 경우 심각한 비즈니스 리스크를 초래할 수 있다는 것을 명심해야 됩니다.

# 추가로 공부하면 좋을 내용

- Grafana Alerting 기능 (특정 메트릭 값이 임계치 초과 시 알림 발송)


---
