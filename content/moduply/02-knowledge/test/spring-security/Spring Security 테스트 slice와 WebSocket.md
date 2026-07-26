# Spring Security 테스트 slice와 WebSocket

## 배경
커스텀 `SecurityConfig`와 `JwtAuthenticationFilter`를 추가한 뒤 기존 CI 테스트가 실패했다.

주요 실패 대상:
- `ContentControllerTest`
- `PlaylistControllerTest`
- `WebSocketConfigTest`

테스트 실패의 핵심은 운영 보안 설정이 테스트 컨텍스트에도 영향을 주기 시작했다는 점이다.

## 컨트롤러 slice 테스트 실패
### 증상
`@WebMvcTest` 기반 컨트롤러 테스트에서 ApplicationContext 로딩이 실패했다.

대표 원인:

```text
No qualifying bean of type 'com.team6.moduply.auth.JwtTokenProvider'
Error creating bean with name 'jwtAuthenticationFilter'
```

### 왜 발생했는가
`@WebMvcTest`는 전체 애플리케이션을 띄우지 않고 MVC 계층 중심으로 테스트한다.

하지만 `JwtAuthenticationFilter`는 `OncePerRequestFilter`이므로 웹 계층 Bean으로 잡힐 수 있다.

이 필터는 다음 의존성을 가진다.

```text
JwtAuthenticationFilter
→ JwtTokenProvider
→ JwtAuthenticationService
→ ModuPlyAuthenticationEntryPoint
```

반면 `@WebMvcTest` slice 컨텍스트에는 서비스/인증 관련 Bean이 전부 올라오지 않는다.

그래서 필터 Bean을 만들다가 의존성 누락으로 실패한다.

### addFilters = false의 한계
`@AutoConfigureMockMvc(addFilters = false)`는 MockMvc 요청에 필터를 적용하지 않는 설정이다.

하지만 필터 Bean 자체의 생성을 막지는 않는다.

따라서 컨텍스트 로딩 단계에서 필터 Bean 생성이 실패하면 `addFilters = false`만으로는 해결되지 않는다.

### 해결
컨트롤러 slice 테스트에서는 `JwtAuthenticationFilter`를 스캔 대상에서 제외한다.

```java
@WebMvcTest(
    controllers = ContentController.class,
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ASSIGNABLE_TYPE,
        classes = JwtAuthenticationFilter.class
    )
)
@AutoConfigureMockMvc(addFilters = false)
class ContentControllerTest {
}
```

이렇게 하면 컨트롤러 테스트는 요청 매핑, DTO 검증, 응답 status/body에 집중하고, JWT 필터는 별도 auth 테스트에서 검증할 수 있다.

## WebSocket 테스트 실패
### 증상
`WebSocketConfigTest`에서 연결 성공 테스트가 timeout으로 실패하고, 허용하지 않은 origin 테스트는 기대한 403이 아니라 401을 받았다.

로그:

```text
GET /ws/info
401
auth09 인증 토큰이 없습니다.
```

### 왜 /ws/info가 호출되는가
SockJS는 실제 WebSocket 연결 전에 서버 정보를 확인하기 위해 다음 요청을 먼저 보낸다.

```text
GET /ws/info
```

이 요청이 통과해야 이후 WebSocket/STOMP 연결이 진행된다.

### 왜 401이 발생했는가
기존 설정에서 `/ws/**`는 CSRF ignore에만 들어가 있었다.

```java
csrf.ignoringRequestMatchers(new AntPathRequestMatcher("/ws/**"))
```

이 설정은 CSRF 검사만 제외한다.

인가 규칙에서는 `/ws/**`가 열려 있지 않았기 때문에 다음 규칙에 걸렸다.

```java
anyRequest().authenticated()
```

그래서 `GET /ws/info`가 `AuthenticationEntryPoint`로 떨어지고 401을 받았다.

### 임시 해결
현재는 WebSocket JWT 인증 인터셉터가 아직 구현되지 않았으므로, CI 통과와 기본 WebSocket 설정 테스트 유지를 위해 `/ws/**`를 임시로 열었다.

```java
// TODO: WebSocket STOMP CONNECT JWT 검증 구현 후 인가 정책 재검토
.requestMatchers("/ws/**").permitAll()
```

## 운영 보안 관점
`/ws/** permitAll` 자체가 항상 잘못은 아니다.

실무에서는 HTTP handshake와 SockJS info 요청은 열어두고, STOMP `CONNECT` 프레임에서 JWT를 검증하는 구조를 자주 사용한다.

권장 최종 구조:

```text
HTTP /ws, /ws/info
→ permitAll

STOMP CONNECT
→ Authorization 또는 access token 검증
→ 유효하면 연결 허용
→ 없거나 잘못되면 연결 거부
```

즉 `SecurityConfig`의 `/ws/** permitAll`은 HTTP handshake 단계만 열어두는 설정으로 보고, 실제 사용자 인증은 STOMP 메시지 계층에서 처리한다.

```java
@Override
public void configureClientInboundChannel(ChannelRegistration registration) {
    registration.interceptors(jwtStompChannelInterceptor);
}
```

현재 프로젝트 요구사항에는 WebSocket 연결 시 JWT 검증 누락 금지가 있으므로, `configureClientInboundChannel`에서 STOMP `CONNECT` JWT 검증을 반드시 구현해야 한다.

STOMP 인터셉터가 구현되면 `/ws/** permitAll`은 유지할 수 있다. 이때의 의미는 "WebSocket 전체를 인증 없이 허용"이 아니라 "HTTP handshake는 허용하고, STOMP `CONNECT`에서 JWT 인증"이다.

반대로 STOMP 인터셉터 없이 `/ws/** permitAll`만 있는 상태는 인증 없는 WebSocket 연결을 허용하는 구조라 운영 보안 기준으로는 미완성이다.

## 테스트 분리 기준
### 컨트롤러 테스트
검증 대상:
- 요청 경로
- HTTP method
- DTO validation
- 응답 status
- 응답 body

보안 필터는 제외한다.

### 인증 필터 테스트
검증 대상:
- Authorization 헤더 추출
- access token 검증
- `SecurityContextHolder` 저장
- 잘못된 token 시 401 응답

### WebSocket 설정 테스트
검증 대상:
- `/ws` endpoint 등록
- 허용 origin 연결 성공
- 비허용 origin 연결 실패

### WebSocket 인증 테스트
JWT 기반 STOMP 인증 구현 후 별도로 추가한다.

검증 대상:
- STOMP CONNECT에 token이 없으면 실패
- 잘못된 token이면 실패
- 유효한 token이면 성공

## 관련
- [[02-knowledge/auth/인증인가와 JWT 정리]]
- [[02-knowledge/auth/JWT 인증인가 구현 리뷰]]
