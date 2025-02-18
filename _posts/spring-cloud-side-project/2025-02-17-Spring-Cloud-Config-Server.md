---
title: Spring Cloud Config를 이용한 환경 변수 관리
description: 
categories:
- Spring Cloud
- MSA

tags:
- Spring Cloud Config
- Git

---

> Spring Cloud Config로 분산 환경에서 환경 변수 설정을 관리하는 방법에 대해 알아봅시다!

<!-- more -->

# Spring Cloud Config란?

Spring Cloud Config는 마이크로서비스 아키텍처 환경에서 각 서비스의 설정을 중앙 집중식으로 관리할 수 있게 도와주는 도구입니다.

설정 정보를 별도의 Config Server에서 관리하고, 각 서비스는 Config Server로부터 설정을 전달받아 애플리케이션 구성을 하게 됩니다.

## 왜 Spring Config Server가 필요한가?

### 일관성
MSA 환경에서는 각 서비스가 개별적으로 설정을 관리할 경우 설정의 중복 및 불일치 문제가 발생할 수 있습니다.

Config Server를 통해 중앙 집중식으로 설정을 관리하게 되면 **설정 정보의 일관성을 유지**할 수 있게 됩니다.

그러므로 분산 시스템의 설정 복잡성을 해소할 수 있게 되는 것이죠.

### 동적 대응

애플리케이션의 새로운 설정을 적용하려면, 보통 애프리케이션을 재시작해야만 합니다.

하지만, Config Server는 각 서비스 애플리케이션들이 **재시작하지 않고도 동적으로 설정을 변경**할 수 있도록 해줍니다.

이로써 운영 중 발생할 수 있는 이슈에 빠르게 대응할 수 있다는 장점이 있습니다.

### 버전 관리 용이

Config Server가 중앙 집중식으로 관리하는 설정 파일들의 저장소로 Git을 사용할 수 있습니다.

Git과 같은 버전관리 시스템과 연동하여 변경 이력을 관리하게 되면, 추후 필요에 따라 원하는 버전으로 롤백이 용이합니다.


## 만약 Config Server가 없다면?

### 설정 관리 

각 서비스마다 개별적으로 설정을 관리하면, 환경별로 일관된 설정 유지가 어려워집니다.

### 운영 효율성 저하

분산된 설정 파일을 하나하나 수정하고 배포해야 하므로, 관리 비용이 대폭 증가하여 운영 효율성이 떨어집니다.

### 동적 반영 불가

설정을 변경하려면 애플리케이션을 재시작해야 합니다.

그리고 변경 사항이 즉각 반영되지 않기 때문에 서비스 중단이나 오류에 대해 대응이 느려질 수 밖에 없습니다.


> 이와 같이 Spring Cloud Config는 분산 환경에서의 설정 관리를 단순화하고, 운영 안정성을 높여주는 핵심 기술이라고 생각합니다.

# Config Server 설정하기

## Dependency 
```groovy
dependencies {
	implementation 'org.springframework.cloud:spring-cloud-config-server'
}
```
- Config Server 사용을 위해 의존성을 추가해줍니다.

## @EnableConfigServer 적용하기
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServiceApplication.class, args);
	}
}
```
- Config Server의 자동 구성을 위해 `@EnableConfigServer` 애너테이션을 적용해줍니다.

## application.yml

```yaml
spring:
  application:
    name: config-service
  cloud:
    config:
      server:
        git:
          uri: https://github.com/Seung-IL-Bang/spring-cloud-ecosystem-remote-config.git
#          username: # private 저장소인 경우
#          password: # private 저장소인 경우

server:
  port: 8888
```
- 위와 같이 설정하면 `spring.cloud.config.server.git.uri` 경로의 레포지토리로부터 설정 파일들을 Config Server에서 읽어올 수 있습니다.
- private 저장소인 경우 접근 가능한 Git 계정의 `username`과 `password`를 입력해줍니다. (암호화해서 입력해야 안전합니다.)

# Config Server 설정 파일 조회

<figure align="center">
<img src="/post_images/spring-cloud-side-project/config-server.png">
<figcaption></figcaption>
</figure>

- public git 저장소에 위와 같이 `test.yml` 파일 하나를 업로드 해놨습니다. 파일 내용은 다음과 같습니다.

```yaml
test:
  value: 123
```
- 위 파일의 설정 값을 Config Server 에서 조회할 수 있습니다.

# {HOST:PORT}/{filename}/{profile}

로컬 환경에서 테스트하는 경우 `http://localhost:8888/test/default` 경로로 Config Server에 `GET` 요청을 하시면 됩니다.

배포 환경에 따라 **프로파일(profile)**별로 설정 파일들을 다르게 조회할 수 있습니다.

파일명에 프로파일을 명시하지 않은 경우 기본 값으로 `default` 가 적용 됩니다.

### 예시
- 운영 환경: `{HOST:PORT}/test/prod`
- 개발 환경: `{HOST:PORT}/test/dev`
- 로컬 환경: `{HOST:PORT}/test/local`

<figure align="center">
<img src="/post_images/spring-cloud-side-project/config-server2.png">
<figcaption></figcaption>
</figure>


- 위 이미지는 Config Service로부터 test.yml 설정 파일을 읽어들인 화면입니다.
- Config Server는 저장소(Git)로부터 설정 파일을 중앙 집중 관리하여 다른 서비스들에게 설정값들을 제공하는 역할을 합니다.

# 📌 요약

- Spring Cloud Config는 분산 환경에서 각 서비스의 설정을 중앙 집중식으로 관리해 일관성과 동적 반영을 가능하게 합니다.
- Git 연동을 통해 버전 관리와 롤백이 용이하여 운영 효율성을 크게 향상시킵니다.

# 🚀 정리

이번 포스트에서는 Spring Cloud Config Server의 개념과 필요성에 대해 알아보았습니다.

다음 포스트에서는 Spring Cloud 기반 MSA 환경에서 각 마이크로 서비스들이 Config Server로부터 파일을 읽어오는 방법과,

`Spring Boot Actuator` 와 `Spring Cloud Bus`를 이용한 실시간 설정 반영 방법에 대해서도 살펴보도록 하겠습니다.



---
