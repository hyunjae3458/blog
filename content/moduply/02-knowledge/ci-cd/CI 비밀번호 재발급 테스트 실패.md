# CI 비밀번호 재발급 테스트 실패

## 문제
- PR #189 CI의 `./gradlew test` 단계에서 테스트가 실패했다.
- 컴파일은 성공했지만 `134 tests completed, 12 failed`로 종료됐다.
- 주요 실패 테스트:
  - `Sb10ModuplyTeam6ApplicationTests.contextLoads`
  - `WebSocketConfigTest`
  - `WatchingSessionIntegrationTest`
  - `UserControllerMethodSecurityTest`

## 원인 1: PasswordEncoder 빈 순환 참조
- CI 로그에 `BeanCurrentlyInCreationException`이 발생했다.
- `SecurityConfig`가 `JwtAuthenticationFilter` 등 보안 컴포넌트를 생성하는 과정에서 `PasswordEncoder`가 필요했다.
- 그런데 `PasswordEncoder` 빈이 `SecurityConfig` 내부에 선언되어 있어 보안 설정 생성 중 다시 `SecurityConfig` 빈 생성을 요구하는 순환 참조가 생겼다.

흐름:
```text
SecurityConfig
-> JwtAuthenticationFilter
-> AuthService / UserDetailsService
-> PasswordEncoder
-> SecurityConfig.passwordEncoder()
```

## 해결 1
- `SecurityConfig`에서 `PasswordEncoder` 빈 선언을 제거했다.
- 별도 설정 클래스 `PasswordEncoderConfig`를 추가해 `PasswordEncoder` 빈을 독립적으로 등록했다.

관련 파일:
- `src/main/java/com/team6/moduply/common/config/SecurityConfig.java`
- `src/main/java/com/team6/moduply/common/config/PasswordEncoderConfig.java`

## 원인 2: WebMvc slice 테스트의 의존성 mock 누락
- `UserControllerMethodSecurityTest`에서 `NoSuchBeanDefinitionException`이 발생했다.
- 이 테스트는 `@WebMvcTest`에 `UserService`를 직접 `@Import`해서 메서드 시큐리티를 검증한다.
- `UserService`에 새 의존성이 추가되면 slice 테스트에도 해당 의존성을 `@MockitoBean`으로 제공해야 한다.
- 비밀번호 변경 흐름에 Redis 임시 비밀번호 파기 처리가 추가되면서 관련 의존성이 테스트 컨텍스트에 누락됐다.

## 해결 2
- `UserControllerMethodSecurityTest`에 필요한 의존성을 mock으로 등록했다.
- 이후 비밀번호 변경 로직이 Redis 직접 삭제에서 `PasswordChangeEvent` 발행으로 변경되면서 테스트도 이벤트 발행 검증으로 맞췄다.

관련 파일:
- `src/test/java/com/team6/moduply/user/UserControllerMethodSecurityTest.java`
- `src/test/java/com/team6/moduply/user/UserServiceTest.java`
- `src/main/java/com/team6/moduply/user/event/PasswordChangeEvent.java`
- `src/main/java/com/team6/moduply/user/event/listener/PasswordChangeListener.java`

## 확인
- 관련 테스트 묶음은 로컬에서 통과했다.

```bash
./gradlew test --tests '*AuthServiceTest' --tests '*ModuPlyUserDetailsServiceTest' --tests '*UserServiceTest' --tests '*UserControllerMethodSecurityTest'
```

## 재발 방지
- 보안 설정 클래스에는 인증 흐름에서 다시 필요해질 수 있는 공용 빈을 두지 않는다.
- `PasswordEncoder`, mapper, util 같은 공용 빈은 독립된 config나 component로 분리한다.
- `@WebMvcTest`에 실제 service를 `@Import`할 때는 service 생성자 의존성이 바뀔 때마다 테스트 mock도 같이 갱신한다.
- 비밀번호 변경처럼 트랜잭션 후처리가 필요한 로직은 서비스 직접 Redis 접근보다 도메인 이벤트로 분리하고, 테스트도 이벤트 발행을 검증한다.

## 관련
- [[03-decisions/auth/임시 비밀번호 인증 정책]]
- [[02-knowledge/test/spring-security/Spring Security 테스트 slice와 WebSocket]]
