---
tags: [ci-cd, test, querydsl, spring-boot]
date: 2026-07-01
status: 해결됨
---

# DataJpaTest QueryDSL 설정 누락

## 한 줄 요약
> `@DataJpaTest`는 JPA slice 테스트라 일반 설정 클래스를 전부 올리지 않는다. 생성자 주입으로 `JPAQueryFactory`를 받는 QueryDSL repository가 추가되면서 repository 테스트 컨텍스트에 `QueryDslConfig`를 명시적으로 import해야 했다.

## 배경
CI에서 repository 테스트들이 한꺼번에 실패했다.

대표 로그는 다음과 같았다.

```text
ContentRepositoryTest > 콘텐츠 목록을 평점순 오름차순으로 조회한다. FAILED
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException
```

`ContentRepositoryTest`, `ContentTagRepositoryTest`, `TagRepositoryTest`, `PlaylistRepositoryTest`가 연쇄적으로 실패했다.

## 문제
repository 테스트 공통 support는 다음처럼 되어 있었다.

```java
@Import(TestcontainersConfig.class)
@DataJpaTest
@ActiveProfiles("test")
@Tag("repository")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class RepositoryTestSupport {
}
```

그런데 QueryDSL repository 구현체는 `JPAQueryFactory` 빈을 생성자 주입으로 요구한다.

```java
@Repository
@RequiredArgsConstructor
public class PlaylistQDSLRepositoryImpl implements PlaylistQDSLRepository {

  private final JPAQueryFactory queryFactory;
}
```

`JPAQueryFactory`는 운영 코드의 `QueryDslConfig`에서 만들어진다.

```java
@Configuration
public class QueryDslConfig {

  @PersistenceContext
  private EntityManager entityManager;

  @Bean
  public JPAQueryFactory jpaQueryFactory() {
    return new JPAQueryFactory(entityManager);
  }
}
```

하지만 `@DataJpaTest`는 slice 테스트라 전체 애플리케이션 설정을 전부 올리지 않는다. 그래서 CI에서는 `JPAQueryFactory` 빈이 없어서 QueryDSL repository 생성이 실패했다.

## 해결
repository 테스트 공통 support에 `QueryDslConfig`를 명시적으로 import했다.

```java
@Import({TestcontainersConfig.class, QueryDslConfig.class})
@DataJpaTest
@ActiveProfiles("test")
@Tag("repository")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class RepositoryTestSupport {
}
```

## 검증

```bash
./gradlew compileTestJava
```

테스트 컴파일은 통과했다.

로컬 환경에서는 Docker/Testcontainers를 사용할 수 없어 repository 테스트 본문 실행까지는 확인하지 못했다. 다만 CI 로그의 핵심 원인인 `NoSuchBeanDefinitionException`은 `JPAQueryFactory` 빈 누락이므로, `QueryDslConfig` import가 직접적인 해결책이다.

## 배운 점
- `@DataJpaTest`는 빠르고 가볍지만 필요한 설정을 자동으로 모두 가져오지 않는다.
- QueryDSL repository를 slice 테스트에서 검증하려면 `JPAQueryFactory` 빈을 테스트 컨텍스트에 명시적으로 올려야 한다.
- 생성자 주입 기반 repository가 늘어나면 기존 repository 테스트 컨텍스트가 깨질 수 있다.
