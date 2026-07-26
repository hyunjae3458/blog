# Spring Security 메서드 시큐리티 테스트

## 배경
사용자 수정과 권한 수정 API에 메서드 시큐리티를 적용했다.

```java
@PreAuthorize("hasRole('ADMIN')")
public void updateUserRole(UUID userId, UserRoleUpdateRequest request)
```

```java
@PreAuthorize("#userId.equals(authentication.principal.userDto.id)")
public UserDto updateUser(UUID userId, UserUpdateRequest request, MultipartFile profileImg)
```

여기서 검증해야 하는 핵심은 일반 CRUD 로직이 아니라 인증/인가 규칙이다.

- 권한 수정은 ADMIN만 가능해야 한다.
- 프로필 수정은 본인만 가능해야 한다.
- PATCH 같은 상태 변경 요청은 CSRF 토큰이 없으면 차단되어야 한다.

## 왜 @InjectMocks 테스트로는 부족한가
`@PreAuthorize`는 메서드 내부에서 직접 실행되는 코드가 아니다.

Spring AOP가 만든 프록시가 메서드 호출 전에 인증/인가를 검사한다.

따라서 아래처럼 Mockito로 만든 서비스 객체를 직접 호출하면 메서드 시큐리티가 실행되지 않는다.

```java
@InjectMocks
private UserService userService;
```

이 테스트는 서비스 내부 로직 검증에는 적합하다.

예를 들면 다음은 단위 테스트로 충분하다.

- 사용자 조회
- 이름 수정
- 프로필 이미지 저장 실패 시 커스텀 예외 변환
- role 필드 변경
- repository 호출 여부

하지만 다음은 단위 테스트로 검증할 수 없다.

- `@PreAuthorize("hasRole('ADMIN')")`
- `@PreAuthorize("#userId.equals(authentication.principal.userDto.id)")`
- CSRF 토큰 누락 차단

## 왜 컨트롤러 slice 테스트에 실제 UserService를 import했는가
이번 테스트는 `@WebMvcTest`를 사용했다.

```java
@WebMvcTest(controllers = UserController.class)
@Import({
    UserService.class,
    UserControllerMethodSecurityTest.MethodSecurityTestConfig.class
})
class UserControllerMethodSecurityTest {
}
```

일반적인 컨트롤러 slice 테스트라면 `UserService`를 mock으로 둔다.

하지만 이번 목적은 컨트롤러 응답만 보는 것이 아니라 `UserService`에 붙은 `@PreAuthorize` 실행 여부를 확인하는 것이다.

그래서 `UserService`를 mock으로 만들면 안 된다.

필요한 구조는 다음과 같다.

```text
MockMvc 요청
→ UserController
→ Spring Bean으로 등록된 UserService 프록시
→ @PreAuthorize 검사
→ 통과하면 서비스 로직 실행
→ 실패하면 403
```

그래서 하위 의존성만 mock 처리했다.

```java
@MockitoBean
private UserRepository userRepository;

@MockitoBean
private UserMapper userMapper;

@MockitoBean
private BinaryContentService binaryContentService;
```

이렇게 하면 DB나 S3 없이도 실제 메서드 시큐리티 프록시를 검증할 수 있다.

## MethodSecurity 설정
테스트에서 메서드 시큐리티를 활성화해야 한다.

```java
@EnableMethodSecurity
static class MethodSecurityTestConfig {
}
```

운영 코드에서는 `SecurityConfig`에 `@EnableMethodSecurity`가 있어야 한다.

테스트 slice에서는 필요한 설정만 가져오므로 테스트 내부 설정으로 명시해도 된다.

## 커스텀 principal이 필요한 이유
권한 수정 테스트는 단순 role만 필요하므로 Spring Security 테스트의 기본 user로 충분하다.

```java
.with(user("admin@example.com").roles("ADMIN"))
```

하지만 프로필 수정 테스트는 SpEL에서 아래 값을 사용한다.

```java
authentication.principal.userDto.id
```

기본 `@WithMockUser`나 `.with(user("..."))`의 principal은 프로젝트의 `ModuPlyUserDetails`가 아니다.

그래서 `userDto` 필드가 없다.

따라서 프로젝트에서 실제로 사용하는 principal 타입을 직접 넣어야 한다.

```java
private ModuPlyUserDetails userDetails(UUID userId) {
    UserDto userDto = new UserDto(
        userId,
        null,
        "user@example.com",
        "user",
        null,
        Role.USER,
        false
    );
    return new ModuPlyUserDetails(userDto, "encoded-password");
}
```

요청에는 이렇게 사용한다.

```java
.with(user(userDetails(userId)))
```

이렇게 해야 `authentication.principal.userDto.id`가 실제로 평가된다.

## 테스트해야 할 시나리오
보안 테스트는 거부 케이스만 있으면 부족하다.

전부 막히는 잘못된 설정이어도 거부 테스트는 통과할 수 있기 때문이다.

상태 변경 API는 허용/거부/CSRF 누락을 한 세트로 둔다.

### 권한 수정 API
대상 API:

```text
PATCH /api/users/{userId}/role
```

검증한 시나리오:

- `ADMIN + csrf` 요청이면 204
- `USER + csrf` 요청이면 403
- `ADMIN + csrf 없음` 요청이면 403

핵심 코드:

```java
mockMvc.perform(patch("/api/users/{userId}/role", userId)
        .with(user("admin@example.com").roles("ADMIN"))
        .with(csrf())
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(request)))
    .andExpect(status().isNoContent());
```

CSRF 누락 테스트는 일부러 `.with(csrf())`를 빼서 검증한다.

```java
mockMvc.perform(patch("/api/users/{userId}/role", userId)
        .with(user("admin@example.com").roles("ADMIN"))
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(request)))
    .andExpect(status().isForbidden());
```

### 프로필 수정 API
대상 API:

```text
PATCH /api/users/{userId}
```

검증한 시나리오:

- `본인 + csrf` 요청이면 200
- `타인 + csrf` 요청이면 403
- `본인 + csrf 없음` 요청이면 403

multipart PATCH 요청은 `multipart()`가 기본 POST로 동작하므로 method를 PATCH로 바꿔야 한다.

```java
mockMvc.perform(multipart("/api/users/{userId}", userId)
        .file(requestPart)
        .with(httpRequest -> {
            httpRequest.setMethod("PATCH");
            return httpRequest;
        })
        .with(user(userDetails(userId)))
        .with(csrf())
        .contentType(MediaType.MULTIPART_FORM_DATA))
    .andExpect(status().isOk());
```

## 인가 실패 예외 처리
`@PreAuthorize` 실패는 필터의 `AccessDeniedHandler`가 처리하지 않을 수 있다.

서비스 메서드 호출 중 `AuthorizationDeniedException`이 발생해 MVC 예외 흐름으로 올라오기 때문이다.

기존에는 공통 `Exception` 핸들러에 잡혀 500으로 응답되는 문제가 있었다.

그래서 `GlobalExceptionHandler`에서 403으로 변환했다.

```java
@ExceptionHandler({AuthorizationDeniedException.class, AccessDeniedException.class})
public ResponseEntity<ErrorResponse> handleAccessDenied(Exception ex) {
    return ResponseEntity.status(HttpStatus.FORBIDDEN)
        .body(ErrorResponse.from(HttpStatus.FORBIDDEN, ex));
}
```

## 정리
- 서비스 단위 테스트는 비즈니스 로직 검증에 적합하다.
- `@PreAuthorize`는 Spring AOP 프록시를 타야 하므로 단위 테스트로 검증되지 않는다.
- 메서드 시큐리티 검증은 Spring 컨텍스트 테스트가 필요하다.
- 컨트롤러 slice 테스트에서도 실제 service bean을 import하면 메서드 시큐리티 프록시를 검증할 수 있다.
- 보안 테스트는 성공/권한 실패/CSRF 실패를 함께 묶어야 한다.
- 프로젝트 커스텀 principal을 SpEL에서 참조한다면 테스트에서도 같은 principal 타입을 넣어야 한다.

## 관련
- [[02-knowledge/auth/인증인가와 JWT 정리]]
- [[02-knowledge/test/spring-security/Spring Security 테스트 slice와 WebSocket]]
