---
title: Spring Batch 효과적인 대량 데이터 리드(Read) 전략 
description: 
categories:
- Spring Batch

tags:
- Spring Batch
- Zero Offset
- Limit-Offset
- Cursor-Based

---

> 스프링 배치를 이용할 때 효과적인 대량 데이터를 읽기 위한 전략에 대해 알아봅시다!

<!-- more -->

# 🚀 들어가기

본 포스트에서는 스프링 배치의 핵심 개념인 청크 프로세싱(Chunk Processing)에 대해 알아보고, 대량의 데이터를 효율적으로 읽기 위해 사이드 프로젝트에서 실제 적용한 전략과 그 성능 개선 결과를 공유하고자 합니다.

# 청크 프로세싱 개념과 중요성 

1,000만 개의 데이터를 배치 처리한다고 가정해 보겠습니다. 이러한 대량의 데이터를 한 번에 메모리에 로드하는 것은 물리적으로 불가능하거나 심각한 성능 저하를 초래합니다. 따라서 1,000개씩 나누어 총 10,000번에 걸쳐 처리하는 방식이 필요합니다.

<figure align="center">
<img src="/post_images/spring-batch-optimization/chunk-processing-diagram.svg">
<figcaption></figcaption>
</figure>

이렇게 메모리 제약을 고려하여 데이터를 일정 크기로 나누어 순차적으로 처리하는 방식을 **청크 프로세싱(Chunk Processing)**이라고 합니다. 스프링 배치는 이러한 청크 단위 처리를 기본 아키텍처로 채택하고 있어 대용량 데이터 처리에 최적화되어 있습니다.

# ItemReader

스프링 배치에서 청크 프로세싱을 구현할 때, 데이터 읽기(Read) 역할은 `ItemReader` 인터페이스를 구현한 구현체가 담당합니다. 저는 초기에 JPA의 개발 편의성과 페이징 기능을 동시에 활용할 수 있는 `JpaPagingItemReader`를 사용했습니다. 하지만 실제 대용량 데이터 처리 과정에서 다음과 같은 심각한 한계점을 경험했습니다.

`JpaPagingItemReader`의 주요 문제점:

1. **데이터 일관성 문제**: 조건절에 사용하는 컬럼이 배치 작업 중 업데이트될 경우 페이지네이션 불일치 발생
2. **JPA 오버헤드**: 불필요한 JPA 기능으로 인한 성능 오버헤드
3. **Limit-Offset 방식의 구조적 한계**: 오프셋이 증가할수록 기하급수적으로 성능 저하

# 페이지네이션 오류

`JpaPagingItemReader`를 사용하면서 가장 먼저 직면한 문제는 페이지네이션이 정확하게 동작하지 않는 현상이었습니다. 예를 들어, 예상했던 10만 건의 처리 대상 데이터 중 약 2만 건만 처리되고 작업이 종료되는 문제가 발생했습니다.

이 문제의 근본 원인은 `JpaPagingItemReader`의 내부 동작 방식과 **OFFSET 기반 페이징**의 한계에 있었습니다. 스프링 배치는 각 청크 단위로 읽기(`ItemReader`) → 처리(`ItemProcessor`) → 쓰기(`ItemWriter`) 사이클을 반복합니다. `JpaPagingItemReader`가 동작할 때 발생하는 주요 이슈는 다음과 같습니다:

1. `JpaPagingItemReader`는 자체적으로 `EntityManager`를 생성하고 관리합니다.
2. 트랜잭션 내에서 처리될 때, 읽어온 엔티티들은 영속 상태로 유지됩니다.
3. `Processor`나 `Writer`에서 엔티티를 업데이트하면, 이 변경사항은 여전히 `Reader`의 `EntityManager`에 의해 관리됩니다.
4. 다음 청크를 읽기 위해 ItemReader가 실행될 때, 변경된 내용이 데이터베이스에 반영됩니다.
5. **LIMIT-OFFSET** 방식의 페이징은 데이터 변경 후에도 원래 개수만큼 **OFFSET**을 증가시키므로 데이터 누락이 발생합니다.

초기 작성했던 다음 코드는 이러한 문제 상황을 보여줍니다:

```java
@Override
@NonNull
public Query createQuery() {
    return this.getEntityManager()
            .createQuery(
                      "SELECT op " +
                            "FROM OrderProduct op " +
                            "LEFT JOIN Claim cl ON op.orderProductId = cl.orderProductId " +
                            "WHERE op.deliveryCompletedAt BETWEEN :startTime AND :endTime " +
                            "AND op.deliveryStatus = 'DELIVERED' " +
                            "AND op.purchaseConfirmedAt IS NULL " +
                            "AND (cl.claimId IS NULL OR cl.completedAt IS NOT NULL) " +
                            "ORDER BY op.orderProductId ASC", OrderProduct.class)
            .setParameter("startTime", startTime)
            .setParameter("endTime", endTime);
}
```

위 쿼리와 함께 발생하는 구체적인 문제 시나리오:

1. 조건절에 `purchaseConfirmedAt IS NULL` 조건이 포함되어 있습니다.
2. 청크 크기를 1,000으로 설정하여 첫 번째 1,000개 레코드(ID 1~1000)를 읽고 처리합니다.
3. `ItemWriter` 단계에서 처리된 레코드의 `purchaseConfirmedAt` 필드가 현재 시간으로 업데이트됩니다.
4. 두 번째 청크 작업 시, `JpaPagingItemReader`는 다음과 같은 내부 쿼리를 생성합니다:

```sql
SELECT op FROM OrderProduct op
WHERE op.deliveryCompletedAt BETWEEN :startTime AND :endTime
AND op.deliveryStatus = 'DELIVERED'
AND op.purchaseConfirmedAt IS NULL
AND (cl.claimId IS NULL OR cl.completedAt IS NOT NULL)
ORDER BY op.orderProductId ASC
OFFSET 1000 LIMIT 1000
```

- 그러나 이 시점에서 처음 1,000개 레코드는 이미 `purchaseConfirmedAt IS NULL` 조건을 더 이상 만족하지 않게 됩니다.
- 따라서 원래 데이터베이스에서 1001~2000 ID를 가진 레코드들이 아닌, 새롭게 조건을 만족하는 처음 1,000개 중에서 OFFSET 1000을 적용하게 됩니다.
- 이로 인해 원래 처리되어야 할 데이터 중 일부(OFFSET으로 인해 건너뛰는 데이터)가 누락되는 현상이 발생합니다.

이러한 문제의 근본적인 원인은 `JpaPagingItemReader`가 내부적으로 채택하고 있는 **LIMIT-OFFSET** 방식의 페이징과 `EntityManager`의 영속성 관리 방식이 조합되어 발생하는 구조적 한계에 있습니다. 

따라서 `JpaPagingItemReader`은 **처리 과정에서 조건절의 컬럼이 변경될 경우, 페이지네이션 로직이 정상적으로 동작하지 않아 데이터 누락이 발생**합니다.

# Limit-Offset 방식의 한계

`JpaPagingItemReader`가 사용하는 `Limit-Offset` 페이지네이션 방식은 대용량 데이터 처리 시 치명적인 성능 저하를 초래합니다. 이 방식의 핵심적인 문제는 **오프셋이 커질수록 데이터베이스가 처리해야 하는 작업량이 기하급수적으로 증가**한다는 점입니다.

예를 들어, `LIMIT 1000 OFFSET 9000`과 같은 쿼리를 실행할 경우:

1. 데이터베이스는 처음부터 9,000번째 레코드까지 모두 스캔합니다.
2. 그 후 9,001번째부터 10,000번째까지의 1,000개 레코드만 실제로 반환합니다.

즉, 오프셋이 커질수록 실제로 **필요한 데이터는 일정한데 반해, 스캔해야 하는 데이터는 계속 증가**하게 됩니다. 수천만 건의 데이터를 처리해야 하는 배치 작업에서 이러한 방식은 심각한 성능 병목을 야기합니다.

# Limit-Offset vs Zero-Offset 비교 다이어그램

<figure align="center">
<img src="/post_images/spring-batch-optimization/offset-comparison-diagram.svg">
<figcaption></figcaption>
</figure>

위 다이어그램은 두 방식의 차이점을 명확하게 보여주는 자료입니다. Limit-Offset의 구조적 한계점 때문에 왜 **Zero-Offset** 방식이 필요한지를 이해할 수 있습니다.

# Zero-Offset 아이템 리더 구현과 성능 개선

`JpaPagingItemReader`의 한계를 극복하기 위해, 저는 `Zero-Offset` 방식(또는 **키셋 페이징**)을 구현한 커스텀 ItemReader를 개발했습니다.

**Zero-Offset** 방식의 핵심 아이디어는 다음과 같습니다:

1. **항상 오프셋을 0으로 유지**: 페이지 이동 시에도 항상 OFFSET 0을 사용
2. **ID 기반 필터링**: 이전 페이지에서 처리한 마지막 레코드의 ID를 기준으로 다음 데이터를 조회
3. **PK 인덱스 활용**: 기본키(PK) 인덱스를 최대한 활용하여 조회 성능 최적화

구현을 위한 필수 조건:

1. PK를 기준으로 데이터를 명시적으로 정렬 (예: `ORDER BY id ASC`)
2. 각 페이지 조회 시, 이전 페이지의 마지막 ID 값을 기준으로 필터링 조건 추가 (예: `WHERE id > :lastId`)

예를 들어, 첫 번째 페이지에서 1,000건을 조회한 후 마지막 레코드의 ID가 1000이라면, 두 번째 페이지 조회 시에는 다음과 같은 쿼리에 `:lastId`에 1000을 설정해주면 됩니다.

```sql
SELECT * FROM order_product 
WHERE ... AND order_product_id > :lastId
ORDER BY order_product_id ASC 
LIMIT 1000
```

이 방식의 장점은 페이지 번호가 아무리 뒤로 가더라도 항상 인덱스를 타는 효율적인 쿼리가 실행된다는 점입니다. 결과적으로 일관된 성능을 유지할 수 있으며, 데이터 일관성 문제도 해결할 수 있습니다.


# 커서 기반 아이템 리더 활용과 성능 개선 

**Zero-Offset** 방식 외에도, **커서 기반(Cursor-based)** 아이템 리더를 활용하는 방법도 효과적입니다. 스프링 배치는 **JdbcCursorItemReader**와 같은 커서 기반 구현체를 제공합니다.

커서 기반 방식의 주요 특징:

1. **스트리밍 방식**: 데이터베이스 커서를 통해 필요한 만큼만 데이터를 스트리밍 방식으로 가져옴
2. **메모리 효율성**: 전체 결과셋을 메모리에 로드하지 않고 필요한 만큼만 가져오므로 메모리 효율적
3. **성능 일관성**: Limit-Offset 방식의 성능 저하 없이 대량 데이터 처리 가능

🚨 다만, 커서 기반 접근법은 다음과 같은 고려사항이 있습니다:

1. **데이터베이스 연결 유지**: 처리가 완료될 때까지 데이터베이스 연결을 유지해야 함. (충분한 `Connection-time` 설정 필요.)
2. **트랜잭션 범위**: 장시간 실행되는 작업의 경우 트랜잭션 관리에 주의 필요.
3. **리소스 관리**: 커서를 명시적으로 닫아야 하므로 리소스 관리에 신경 써야 함. (`JdbcCursorItemReader`는 리소스를 자동으로 정리.)

# 최종 선택: 커서 기반 접근법 JdbcCursorItemReader

이론적으로는 **Zero-Offset** 방식이 매우 효율적이지만, 실제 프로젝트에서는 **커서 기반 접근법(Cursor-based approach)**을 최종적으로 채택했습니다. 이러한 결정을 내린 주요 이유는 다음과 같습니다:

- **개발 리소스 효율성**: Zero-Offset 방식은 매번 새로운 Reader를 구현할 때마다 상당한 개발 공수가 필요합니다. 반면 Spring Batch에서 기본 제공하는 `JdbcCursorItemReader`를 사용하면 즉시 활용 가능합니다.
- **유지보수 용이성**: 커스텀 구현체보다 Spring Batch의 공식 컴포넌트를 사용함으로써 유지보수가 용이하고 버전 업그레이드 시 호환성 문제가 적습니다.
- **성능 대비 비용**: Zero-Offset 방식이 이론적으로 약간 더 뛰어난 성능을 보일 수 있지만, 커서 기반 접근법도 충분히 우수한 성능을 제공하면서 개발 비용은 크게 절감할 수 있었습니다.

⭐️ 따라서, 성능상 크게 차이가 안 나는 **Zero-Offset** 방식과 **Cursor-Based** 방식 중 실용적인 측면을 생각하여 Cursor-based인 `JdbcCursorItemReader` 를 프로젝트에 적용하였습니다.

다음은 실제 구현에 사용한 `JdbcCursorItemReader` 설정의 코드입니다:

```java
@Bean
@StepScope
public JdbcCursorItemReader<OrderProduct> deliveryCompletedJdbcItemReader(
        @Value("#{jobParameters['settlementDate']}") String settlementDateStr) {

    LocalDate date = JobParameterUtils.parseSettlementDate(settlementDateStr);
    LocalDateTime startTime = date.atStartOfDay();
    LocalDateTime endTime = date.plusDays(1).atStartOfDay();

    // SQL 쿼리 작성
    String sql = """
        SELECT op.*
        FROM order_product op
        LEFT JOIN claim cl ON op.order_product_id = cl.order_product_id
        WHERE op.delivery_completed_at BETWEEN ? AND ?
        AND op.delivery_status = 'DELIVERED'
        AND op.purchase_confirmed_at IS NULL
        AND (cl.claim_id IS NULL OR cl.completed_at IS NOT NULL)
        ORDER BY op.order_product_id ASC
        """;

    // PreparedStatement 파라미터 설정
    Object[] parameters = new Object[] {
            startTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")),
            endTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
    };

    return new JdbcCursorItemReaderBuilder<OrderProduct>()
            .name("deliveryCompletedJdbcItemReader")
            .dataSource(dataSource)
            .sql(sql)
            .preparedStatementSetter(new ArgumentPreparedStatementSetter(parameters))
            .rowMapper(new BeanPropertyRowMapper<>(OrderProduct.class))
            .build();
}
```

- **Zero-Offset** 방식의 `ItemReader`를 직접 개발하는 것보다 Spring Batch에서 지원해주는 `JdbcCursorItemReader`를 사용함으로써 매번 `:lastId`를 신경쓰지 않고도 편리하게 개발할 수 있습니다.

# 성능 비교: JPA Paging vs Cursor-based

실제 동일한 데이터셋(최대 100만 건의 주문 데이터)에 대해 두 가지 방식을 적용한 성능 비교 결과는 다음과 같습니다:

<figure align="center">
<img src="/post_images/spring-batch-optimization/realistic-performance-comparison.svg">
<figcaption></figcaption>
</figure>

참고로, `JpaPagingItemReader` 읽기가 2만건 처리에 6분이 걸렸고, 그 이상의 데이터는 테스트하기 어려웠던 현실적인 상황을 반영하여 추정치를 표시했습니다.

성능 테스트 결과 분석:

- **JPA Paging**: 데이터량이 증가할수록 처리 시간이 **기하급수적으로 증가**함.
- **Cursor-based**: 빠른 처리 속도를 보이며, 데이터량이 증가하더라도 처리 시간이 **선형적으로 증가**함.

# 보완할 점

현재 구현에서 가장 큰 개선점은 타입 안전성입니다. 문자열 기반 쿼리를 사용하는 현재 방식은 오타나 컬럼명 변경 시 컴파일 타임에 오류를 잡아내기 어렵습니다.

향후 개선 방향:

- **QueryDSL 도입**: Java 코드로 타입 안전한 쿼리를 작성하여 컴파일 타임에 오류 감지
- **테스트 커버리지 강화**: 다양한 데이터 패턴에 대한 테스트 케이스 추가
- **모니터링 기능 개선**: 배치 작업의 진행 상황 및 성능 지표 실시간 모니터링

Kotlin 기반 프로젝트의 경우, **Exposed 라이브러리**를 활용하면 더욱 간결하고 타입 안전한 방식으로 쿼리를 작성하실 수 있으니 참고하시면 좋을 것 같습니다.

# ⭐️ 결론

대용량 데이터 처리 시 Spring Batch의 `ItemReader` 전략은 전체 배치 프로세스의 성능과 안정성에 결정적인 영향을 미칩니다. 특히 `Limit-Offset` 방식의 페이징 대신 **Zero-Offset** 또는 **커서 기반 접근법**을 활용함으로써 다음과 같은 이점을 얻을 수 있습니다:

- **일관된 성능**: 처리 데이터량에 관계없이 예측 가능한 성능 유지
- **데이터 일관성**: 배치 처리 중 데이터 변경에도 페이지네이션 정확성 보장
- **리소스 효율성**: 데이터베이스와 애플리케이션 서버의 리소스 효율적 활용

대규모 배치 작업을 설계할 때는 단순히 **코드 작성의 편의성보다 성능과 확장성을 우선적으로 고려하는 것이 중요**합니다. 본 포스트에서 소개한 전략들이 효율적인 배치 시스템 구축에 도움이 되길 바랍니다.

# 참고 자료

- <a href="https://www.youtube.com/watch?v=2IIwQDIi3ys&t=1534s" target="_blank">Batch Performance 극한으로 끌어올리기: 1억 건 데이터 처리를 위한 노력 / if(kakao)dev2022</a>
- <a href="https://www.youtube.com/watch?v=VSwWHHkdQI4&t=1369s" target="_blank">Spring Batch 애플리케이션 성능 향상을 위한 주요 팁 (kakao tech)</a>
- <a href="https://docs.spring.io/spring-batch/reference/5.2-SNAPSHOT/readers-and-writers/database.html#JdbcCursorItemReader" target="_blank">Spring Batch Documentation: Cursor-based ItemReader Implementations</a>


---
