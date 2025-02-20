---
title: Spring Cloud Gateway와 JWT: 분산 시스템의 중앙집중식 인증 구현
description: 
categories:
- Spring Cloud
- MSA

tags:
- JWT
- Spring Cloud Gateway

---

> Spring Cloud Gateway와 JWT를 활용한 중앙 집중식 인증에 대해 알아봅시다!

<!-- more -->

# 들어가기 

분산 시스템에서 `JWT`와 같은 토큰 기반 인증을 사용하지 않는다면 어떻게 될까요? 

이번 포스트에서는 분산 시스템에서 Spring Cloud Gateway와 JWT 토큰 기반 인증을 사용의 이점에 대해 알아보는 시간을 갖도록 하겠습니다.

## 분산 시스템(No Token & No Gateway)

우선, 분산 환경에서 API Gateway와 토큰 기반 인증 기술을 결합한 중앙집중식 인증 시스템을 도입하지 않는 경우 발생하는 문제점들에 대해 먼저 살펴보도록 하겠습니다.

1. 중앙집중식 인증 시스템이 없다면, 각 서비스마다 별도의 인증/인가 로직을 구현해야 합니다. 이는 코드의 중복과 유지보수의 어려움이 발생할 수 있습니다.

2. 토큰 대신 사용하는 세션 기반 인증 방식은 서버에 상태 정보를 저장해야 하기 때문에, 분산 환경에서의 서버 간 세션 동기화 문제가 발생할 수 있습니다. 이는 시스템 확장성에 어려움을 증가시킵니다.

## 분산 시스템(Gateway + Token 기반)

그렇다면 Spring Cloud Gateway와 JWT 결합의 중앙 집중식 인증 시스템은 어떠한 장점이 있는지 살펴보겠습니다.

1. 모든 요청에 대한 검사를 API Gateway에서 한 번에 수행할 수 있어, 개별 서비스마다 중복된 인증 로직을 구현할 필요가 없습니다. 이를 통해 보안 설정의 일관성이 확보되고, 유지보수가 용이해집니다.

2. JWT는 토큰 기반 인증 방식으로, 별도의 세션 관리를 하지 않고도 사용자 인증 정보를 안전하게 전달 할 수 있습니다. 이는 분산 시스템 환경에서 효율적인 리소스 사용과 확장성을 보장합니다.

# 예제 코드

이번에는 사이드 프로젝트에서 직접 Spring Cloud Gateway와 JWT를 결합한 인증 필터를 살펴보겠습니다.

다음 코드는 `Spring Cloud Gateway` 에서 사용자 요청의 담겨 있는 `Authorization` 헤더의 `JWT` 토큰이 유효한지 검증하는 코드입니다.

```java
@Component
@Slf4j
public class AuthorizationHeaderFilter extends AbstractGatewayFilterFactory<AuthorizationHeaderFilter.Config> {

    public static class Config {
    }

    private Environment env;

    public AuthorizationHeaderFilter(Environment env) {
        super(Config.class);
        this.env = env;
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {

            ServerHttpRequest request = exchange.getRequest();

            if (!request.getHeaders().containsKey(HttpHeaders.AUTHORIZATION)) {
                return onError(exchange, "Authorization header is missing", HttpStatus.UNAUTHORIZED);
            }

            String authorizationHeader = request.getHeaders().get(HttpHeaders.AUTHORIZATION).get(0);
            String jwt = authorizationHeader.replace("Bearer", "").trim();

            if (isNotValidJwt(jwt)) {
                return onError(exchange, "JWT is not valid", HttpStatus.UNAUTHORIZED);
            }

            return chain.filter(exchange);
        };
    }

    // WebFlux method to handle errors
    private Mono<Void> onError(ServerWebExchange exchange, String err, HttpStatus httpStatus) {
        exchange.getResponse().setStatusCode(httpStatus);
        log.error(err);
        return exchange.getResponse().setComplete();
    }

    private boolean isNotValidJwt(String jwt) {
        boolean returnValue = false;

        String secretKey = env.getProperty("token.secret");
        byte[] keyBytes = Base64.getDecoder().decode(secretKey);
        SecretKey key = Keys.hmacShaKeyFor(keyBytes);

        String subject = Jwts.parser()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(jwt)
                .getBody()
                .getSubject();

        if (subject == null || subject.isEmpty()) {
            returnValue = true;
        }
        return returnValue;
    }
}
```

- 위 코드처럼 Gateway에서 모든 요청에 대한 인증을 한 번에 처리함으로써 보안 관리가 용이해집니다.
- 그리고 각 서비스에서 별도의 세션 관리를 하지 않아도 됩니다.
- 인증/인가 로직을 게이트웨이에서 처리하므로 각 백엔드 서비스의 중복된 코드를 줄일 수 있습니다.

이처럼 Spring Cloud Gateway와 JWT 인증의 결합은 복잡한 분산 환경에서 안정적이고 효율적인 인증/인가를 가능하게 해줍니다.

# 추가로 고려해볼 만한 내용

- JWT 사용 시 발생할 수 있는 보안 이슈와 이에 대한 대응 방법

# 요약
Spring Cloud Gateway와 JWT 결합 사용의 이유 및 기대 효과는 다음과 같습니다.

- 중앙 집중식 보안 관리로 유지보수가 용이
- 분산 환경에 적합한 `stateless` 인증 방식
- 확장성과 유연성 확보
- 효율적인 리소스 사용

---
