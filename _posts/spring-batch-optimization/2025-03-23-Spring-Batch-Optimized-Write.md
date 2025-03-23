---
title: Spring Batch 효율적인 대량 데이터 Write 전략
description: 
categories:
- Spring Batch

tags:
- Spring Batch
- JPA
- JDBC
- Batch Insert

---

> 스프링 배치 환경에서 **JPA 기반** `ItemWriter`의 성능 저하 요인을 분석하고, **JDBC 배치 인서트 전략**을 통한 성능 개선 방안을 알아봅니다.

<!-- more -->

# 🚀 들어가기 

대용량 데이터 처리는 엔터프라이즈 애플리케이션에서 흔히 마주하는 도전 과제입니다. 특히 **Spring Batch** 환경에서 대량의 데이터를 효율적으로 저장하는 것은 전체 배치 프로세스의 성능에 직접적인 영향을 미칩니다. 본 포스트에서는 **JPA 기반** `ItemWriter` 사용 시 발생하는 성능 제약 요인을 상세히 분석하고, **JDBC 기반 배치 인서트**를 활용한 성능 최적화 전략에 대해 살펴보겠습니다.

# JPA 기반 ItemWriter의 성능 제약 요인

먼저 배치 처리 환경에서 JPA의 적합성에 대해 고민해볼 필요가 있습니다. JPA는 **영속성 컨텍스트**를 통해 개발자에게 많은 **편의성을 제공**하지만, 이러한 추상화 계층은 내부적으로 복잡한 로직을 수행하며 대량 데이터 처리 시 **성능 오버헤드**로 작용할 수 있습니다.

# 영속성 컨텍스트와 더티 체킹 오버헤드

**JPA**는 정교한 **ORM 프레임워크**로서 엔티티의 상태 변화를 감지하고 관리하기 위해 **영속성 컨텍스트**와 **더티 체킹(Dirty Checking) 메커니즘**을 활용합니다. 이 과정에서 다음과 같은 오버헤드가 발생합니다:

1. **엔티티 상태 추적**: 모든 엔티티의 원본 상태를 메모리에 유지하고 변경사항을 감지
2. **스냅샷 비교**: 트랜잭션 커밋 시점에 원본 스냅샷과 현재 상태를 비교하는 연산 수행
3. **캐시 관리**: 1차 캐시와 2차 캐시의 동기화 및 관리 비용

**배치 처리**에서는 일반적으로 **데이터의 읽기와 쓰기 단계가 명확히 구분**되어 있어, 이러한 정교한 **상태 관리 메커니즘이 불필요한 오버헤드로 작용**할 수 있습니다.

# 불필요한 컬럼 업데이트 문제

JPA는 기본적으로 변경 감지 시 엔티티의 모든 필드를 포함한 `UPDATE` 쿼리를 생성합니다. 이는 다음과 같은 문제를 야기합니다:

1. **비효율적인 SQL 생성**: 실제로 변경된 필드만 업데이트하는 것이 아니라 모든 필드를 포함한 SQL 생성
2. **데이터베이스 부하 증가**: 불필요한 컬럼까지 업데이트함으로써 데이터베이스 리소스 낭비
3. **최적화 제한**: 특정 컬럼만 선택적으로 업데이트하는 최적화 전략 적용 어려움

이러한 **JPA의 특성**은 대규모 배치 처리 환경에서 심각한 **성능 병목** 현상을 초래할 수 있습니다.

아래는 스프링 배치에서 **JPA 기반의** `ItemWriter`를 사용하는 일반적인 코드입니다:

```java
@Bean
public JpaItemWriter<DailySettlement> aggregateDailySettlementWriter() {
    JpaItemWriter<DailySettlement> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(entityManagerFactory);
    return writer;
}
```

이 코드는 **간결하고 직관적**이지만, 내부적으로는 **영속성 컨텍스트 관리, 변경 감지, 그리고 각 엔티티별 개별 업데이트라는 오버헤드**를 수반합니다.

<figure align="center">
<img src="/post_images/spring-batch-optimization/jpa-overhead-diagram.svg">
<figcaption></figcaption>
</figure>

# JDBC 배치 인서트를 통한 성능 최적화

**JDBC 배치 인서트**는 여러 SQL 명령을 그룹화하여 **단일 네트워크 요청**으로 데이터베이스에 전송하는 기법입니다. 이 방식은 다음과 같은 이점을 제공합니다:

1. **네트워크 오버헤드 감소**: 여러 쿼리를 하나의 요청으로 묶어 네트워크 왕복 시간 최소화
2. **데이터베이스 최적화**: 데이터베이스가 여러 연산을 일괄 처리하여 내부 최적화 적용 가능
3. **리소스 효율성**: 커넥션 관리 및 트랜잭션 오버헤드 감소


다음은 사이드 프로젝트에서 일일 정산 집계 데이터를 업데이트 하기 위해 **JDBC 배치 인서트**를 활용한 최적화된 `ItemWriter` 구현 예시입니다:

```java
@Bean
public ItemWriter<SettlementAggregationResult> optimizedAggregateDailySettlementJdbcWriter() {
    return new ItemWriter<SettlementAggregationResult>() {
        @Autowired
        private JdbcTemplate jdbcTemplate;

        @Override
        public void write(Chunk<? extends SettlementAggregationResult> items) throws Exception {
            if (items.isEmpty()) {
                return;
            }

            log.info("Processing {} settlement aggregation results with JDBC batch update", items.size());

            String updateSql = """
                        UPDATE daily_settlement SET
                            total_order_count = ?,
                            total_claim_count = ?,
                            total_quantity = ?,
                            sales_amount = ?,
                            tax_amount = ?,
                            promotion_discount_amount = ?,
                            coupon_discount_amount = ?,
                            point_used_amount = ?,
                            shipping_fee = ?,
                            claim_shipping_fee = ?,
                            commission_amount = ?,
                            total_settlement_amount = ?,
                            updated_at = ?
                        WHERE settlement_id = ?
                    """;

            jdbcTemplate.batchUpdate(updateSql, new BatchPreparedStatementSetter() {
                @Override
                public void setValues(PreparedStatement ps, int i) throws SQLException {
                    SettlementAggregationResult result = items.getItems().get(i);
                    LocalDateTime now = LocalDateTime.now();
                    int idx = 1;

                    // SET clause parameters
                    ps.setInt(idx++, result.getTotalOrderCount());
                    ps.setInt(idx++, result.getTotalClaimCount());
                    ps.setInt(idx++, result.getTotalQuantity());
                    ps.setBigDecimal(idx++, result.getTotalSalesAmount());
                    ps.setBigDecimal(idx++, result.getTotalTaxAmount());
                    ps.setBigDecimal(idx++, result.getTotalPromotionDiscountAmount());
                    ps.setBigDecimal(idx++, result.getTotalCouponDiscountAmount());
                    ps.setBigDecimal(idx++, result.getTotalPointUsedAmount());
                    ps.setBigDecimal(idx++, result.getTotalShippingFee());
                    ps.setBigDecimal(idx++, result.getTotalClaimShippingFee());
                    ps.setBigDecimal(idx++, result.getTotalCommissionAmount());
                    ps.setBigDecimal(idx++, result.getTotalSettlementAmount());
                    ps.setTimestamp(idx++, Timestamp.valueOf(now));

                    // WHERE clause parameter
                    ps.setLong(idx++, result.getSettlementId());
                }

                @Override
                public int getBatchSize() {
                    return items.size();
                }
            });

            log.info("Successfully updated {} daily settlements with batch update", items.size());
        }
    };
}
```

이 구현에서는 **JPA의 불필요한 오버헤드를 제거**하고, 주어진 청크 내의 모든 항목을 단일 배치 연산으로 처리함으로써 **네트워크 왕복 시간을 크게 감소**시킵니다.

# ID 생성 전략과 JPA 배치 인서트 제약

**JPA**에서도 `hibernate.jdbc.batch_size` 속성을 통해 배치 인서트를 지원하지만, **ID 생성 전략**에 따라 중요한 **제약**이 존재합니다:

1. `IDENTITY` **전략 제약**:
  - IDENTITY 전략을 사용할 경우 Hibernate는 배치 인서트를 비활성화합니다.
  - 이는 데이터베이스가 자동 증가 ID 값을 생성하고, Hibernate는 이 값을 즉시 조회해야 하기 때문입니다.
2. `SEQUENCE` **전략 활용**: 
  - SEQUENCE 전략을 사용하면 Hibernate가 사전에 ID 값을 할당하여 배치 인서트를 수행할 수 있습니다.
  - `SEQUENCE` 전략은 Hibernate가 미리 여러 개의 ID 값을 가져올 수 있습니다(`allocationSize` 활용).
  - 즉, Hibernate는 ID 값을 미리 할당한 후, 여러 개의 `INSERT` 문을 한 번의 JDBC batch로 실행할 수 있습니다.
3. `TABLE` **전략 오버헤드**:
   - TABLE 전략은 별도의 테이블을 사용하여 ID를 생성하는 방식이며, 배치 인서트 자체는 가능하지만 시퀀스 테이블에 대한 동시 접근으로 인해 **잠금 경합(lock contention)**이 발생할 가능성이 높습니다.
   - 따라서 대량 삽입 작업 시 성능 저하가 발생할 수 있습니다.

실무 환경에서는 종종 데이터베이스의 자동 증가 기능에 의존하는 `IDENTITY` 전략을 사용하기 때문에, JPA의 배치 인서트 기능을 활용하기에 현실적으로 어려운 경우가 많습니다. 이런 상황에서는 **순수 JDBC**를 직접 사용하는 배치 인서트가 효과적인 대안이 될 것입니다.

# 추가 개선 방안

**JDBC** 방식을 활용하면서 발생할 수 있는 **SQL문의 타입 안전성 부재**와 **유지보수성 저하 문제**를 해결하기 위해 다음과 같은 개선 방안을 고려할 수 있습니다:

## QueryDSL 도입

**QueryDSL**을 활용하면 타입 안전한 방식으로 SQL 쿼리를 작성하면서도 배치 인서트의 성능 이점을 유지할 수 있습니다.

아래 코드는 QueryDSL을 적용한 예시 코드입니다. 기존 문자열로 SQL를 다룰 때보다 깔끔하고, 컴파일 시점에 오류를 잡아낼 수 있어서 안정적입니다:

```java
@Bean
public ItemWriter<SettlementAggregationResult> querydslBatchWriter() {
    return new ItemWriter<SettlementAggregationResult>() {
        @Autowired
        private SQLQueryFactory queryFactory;
        
        @Override
        public void write(Chunk<? extends SettlementAggregationResult> items) {
            // QueryDSL을 사용한 배치 업데이트 구현
            List<SQLInsertBatch> batch = items.getItems().stream()
                .map(this::createBatchUpdate)
                .collect(Collectors.toList());
                
            queryFactory.batch(batch.toArray(new SQLInsertBatch[0])).execute();
        }
        
        private SQLInsertBatch createBatchUpdate(SettlementAggregationResult result) {
            // 타입 안전한 업데이트 구문 작성
            QDailySettlement settlement = QDailySettlement.dailySettlement;
            return queryFactory.update(settlement)
                .set(settlement.totalOrderCount, result.getTotalOrderCount())
                // 기타 필드 설정
                .where(settlement.settlementId.eq(result.getSettlementId()));
        }
    };
}
```

# 성능 모니터링 및 최적화 - 앞으로의 계획

현재 사이드 프로젝트에서는 기본적인 배치 인서트 구현에 집중하고 있지만, 향후 성능을 더욱 개선하기 위해 다음과 같은 추가 전략을 적용할 계획입니다:

1. **청크 크기 최적화**: 메모리 사용량과 처리 속도의 균형을 고려한 최적의 청크 크기를 찾는 실험을 진행할 예정입니다. 현재는 고정값(size: 1000)을 사용 중이지만, 실제 데이터 특성에 맞는 최적화가 필요합니다.
2. **실행 계획 분석**: 배치 처리 시 생성되는 SQL의 실행 계획을 분석하여 인덱스 활용을 최적화하는 작업도 진행하면 좋을 것 같습니다. 현재는 기본 인덱스만 활용 중이지만, 실행 패턴에 따른, 좀 더 세밀한 인덱스 전략이 필요해 보입니다.
3. **Rewrite 배치 전략**: 대량 데이터 갱신 시 임시 테이블을 활용한 Rewrite 전략도 검토 중입니다. 이 전략은 다음과 같은 이유로 특히 대규모 업데이트가 필요한 월별 정산 처리에서 효과적일 것으로 예상됩니다:

   - **테이블 잠금 최소화**: 개별 UPDATE 문은 해당 레코드에 대한 잠금을 발생시켜 동시성을 저하시킬 수 있지만, 임시 테이블에 데이터를 먼저 준비한 후 한 번에 교체하면 잠금 시간을 최소화할 수 있습니다.
   - **트랜잭션 로그 감소**: 수많은 개별 UPDATE 문은 대량의 트랜잭션 로그를 생성하지만, INSERT-SELECT 또는 MERGE 패턴을 사용하면 로그량을 크게 줄일 수 있습니다.
   - **인덱스 부하 감소**: 반복적인 UPDATE는 인덱스 재구성 부하를 지속적으로 발생시키지만, 임시 테이블 방식은 인덱스 유지 비용을 일괄 처리할 수 있습니다.
   - **I/O 패턴 최적화**: 순차적 I/O는 랜덤 I/O보다 훨씬 효율적인데, 임시 테이블을 생성하고 대량 데이터를 삽입하는 경우는 주로 순차적 I/O 패턴을 따르게 됩니다.

이러한 고급 최적화 전략들은 현재 기본 구현을 완료한 후 성능 테스트를 통해 점진적으로 도입할 예정입니다. 실제 적용 결과와 성능 개선 지표는 후속 포스팅에서 공유하겠습니다.

# 결론

**Spring Batch** 환경에서 대용량 데이터를 효율적으로 처리하기 위해서는 사용 사례에 적합한 기술을 선택하는 것이 중요합니다. **JPA**는 개발 생산성과 객체 지향적 접근을 제공하지만, 대량 데이터 처리 시에는 성능 제약이 존재합니다. 반면 **JDBC 배치 인서트**는 로우 레벨 접근을 통해 최적의 성능을 제공하지만 타입 안전성과 유지보수성 측면에서 추가적인 노력이 필요합니다.

실무에서는 이러한 **트레이드오프**를 충분히 이해하고, 처리해야 할 **데이터의 볼륨과 성능 요구사항에 따라 적절한 전략을 선택**해야 합니다. 특히 대규모 데이터 처리가 필요한 배치 작업에서는 **JDBC 배치 인서트**와 같은 최적화 기법을 적극적으로 활용하되, **QueryDSL**과 같은 도구를 함께 도입하여 **유지보수성**도 함께 확보하는 것이 좋을 것 같습니다.


# 마무리 

본 포스트가 Spring Batch 환경에서 대용량 데이터 처리 시 발생할 수 있는 성능 병목 현상을 이해하고, 실질적인 최적화 전략을 수립하는 데 도움이 되었으면 좋겠습니다. 해당 사이드 프로젝트를 진행하면서 기술적 편의성과 성능 사이의 균형을 찾는 과정이 엔지니어링의 핵심이라는 것을 다시 한 번 느낄 수 있었습니다. 앞으로도 실무에서 얻은 경험과 학습을 통해 더 나은 기술적 해결책을 모색하고 공유하겠습니다.

감사합니다.

# 참고 자료

- <a href="https://docs.jboss.org/hibernate/orm/6.6/introduction/html_single/Hibernate_Introduction.html#statement-batching" target="_blank">Hibernate - Enabling statement batching</a>
- <a href="https://www.baeldung.com/jpa-hibernate-batch-insert-update" target="_blank">Batch Insert/Update with Hibernate/JPA</a>
- <a href="https://docs.spring.io/spring-framework/reference/data-access/jdbc/advanced.html#jdbc-batch-classic" target="_blank">Basic Batch Operations with JdbcTemplate</a>
- <a href="https://docs.oracle.com/cd/E19455-01/806-3204/6jccb3gac/index.html" target="_blank">Optimizing for Random I/O and Sequential I/O</a>

---
