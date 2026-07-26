# 회원가입, 로그인, JWT 요청 흐름

## 문제
사용자 인증 흐름은 회원가입, 로그인, JWT 발급, 인증된 요청 처리로 이어진다.

각 단계에서 어떤 객체와 컴포넌트가 관여하는지 헷갈리기 쉽기 때문에 전체 흐름과 JWT 구성요소를 정리한다.

## 전체 흐름
큰 흐름은 다음과 같다.

```text
회원가입
→ 로그인
→ JWT 발급
→ 클라이언트가 JWT 저장
→ 인증이 필요한 API 요청 시 Authorization 헤더에 JWT 포함
→ 서버가 JWT 검증
→ SecurityContext에 인증 정보 저장
→ 컨트롤러/서비스에서 인증 사용자 정보 사용
```

## 1. 회원가입
회원가입은 인증 전 단계다.

클라이언트는 `POST /api/users`로 회원가입 요청을 보낸다.

```json
{
  "username": "test@example.com",
  "password": "password1!"
}
```

서버는 다음을 수행한다.

- 요청 DTO 검증
- 이메일 중복 확인
- 비밀번호 인코딩
- `User` 엔티티 생성
- DB 저장
- `UserDto` 응답 반환

중요한 점은 비밀번호를 절대 평문으로 저장하지 않는 것이다.

```text
password
→ PasswordEncoder.encode(password)
→ encodedPassword
→ User.encodedPassword 저장
```

회원가입이 끝났다고 바로 인증된 상태가 되는 것은 아니다. 일반적으로 회원가입 이후 별도 로그인을 통해 JWT를 발급받는다.

## 2. 로그인
로그인은 사용자가 입력한 이메일/비밀번호가 실제 저장된 사용자 정보와 일치하는지 확인하는 과정이다.

흐름은 다음과 같다.

```text
POST /api/auth/sign-in
→ Spring Security 인증 필터
→ AuthenticationManager
→ UserDetailsService.loadUserByUsername(email)
→ DB에서 사용자 조회
→ PasswordEncoder.matches(rawPassword, encodedPassword)
→ 인증 성공
→ LoginSuccessHandler
→ JwtTokenProvider로 JWT 발급
```

`UserDetailsService`는 DB에서 사용자를 조회하고, Spring Security가 이해할 수 있는 `UserDetails`를 반환한다.

현재 프로젝트에서는 다음 구조를 사용한다.

```text
ModuPlyUserDetailsService
→ UserRepository.findByEmail(email)
→ UserMapper.toDto(user)
→ new ModuPlyUserDetails(userDto, encodedPassword)
```

`ModuPlyUserDetails`는 다음 정보를 제공한다.

- username: email
- password: encodedPassword
- authorities: `ROLE_USER`, `ROLE_ADMIN`
- account locked 여부

## 3. JWT 발급
로그인 성공 후 서버는 JWT를 발급한다.

JWT는 서버가 “이 사용자는 인증된 사용자다”라는 사실을 서명해서 클라이언트에게 전달하는 토큰이다.

예시 claim:

```text
sub: 사용자 식별자
userId: 사용자 ID
role: ROLE_USER
name: 사용자 이름
iat: 발급 시간
exp: 만료 시간
```

서버는 `JwtTokenProvider`에서 다음 과정을 수행한다.

```text
Authentication에서 principal 추출
→ UserDetails에서 사용자 정보 추출
→ JWTClaimsSet 생성
→ JWSHeader 생성
→ SignedJWT 생성
→ secret key로 서명
→ token 문자열 반환
```

Nimbus JOSE + JWT 기준으로는 대략 다음 객체들이 관여한다.

- `JWTClaimsSet`: JWT payload의 claim 구성
- `JWSHeader`: 서명 알고리즘 등 header 구성
- `SignedJWT`: header + claims를 가진 서명 대상 JWT
- `MACSigner`: HS256 같은 대칭키 서명 수행

## 4. 클라이언트의 인증 요청
클라이언트는 발급받은 access token을 이후 요청에 포함한다.

```http
Authorization: Bearer {accessToken}
```

예를 들어 사용자 단건 조회 요청은 다음처럼 갈 수 있다.

```http
GET /api/users/{userId}
Authorization: Bearer eyJ...
```

## 5. 서버의 JWT 검증
서버는 `JwtAuthenticationFilter`에서 요청마다 JWT를 확인한다.

흐름은 다음과 같다.

```text
요청 수신
→ Authorization 헤더 확인
→ Bearer token 추출
→ JwtTokenProvider.validateAccessToken(token)
→ 서명 검증
→ 만료 시간 검증
→ claim 추출
→ Authentication 객체 생성
→ SecurityContextHolder에 저장
→ 이후 컨트롤러로 요청 진행
```

`SecurityContextHolder`에 인증 정보가 들어가면 Spring Security는 해당 요청을 인증된 요청으로 처리한다.

컨트롤러나 서비스에서는 이후 `Authentication`, `Principal`, `@AuthenticationPrincipal` 등을 통해 인증 사용자 정보를 사용할 수 있다.

## JWT 구성요소
JWT는 세 부분으로 구성된다.

```text
header.payload.signature
```

각 부분은 Base64URL로 인코딩되어 `.`으로 연결된다.

### Header
토큰 타입과 서명 알고리즘을 담는다.

예시:

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

### Payload
사용자 정보와 토큰 메타데이터를 담는다. 여기 들어가는 값들을 claim이라고 부른다.

예시:

```json
{
  "sub": "user-id",
  "userId": "uuid",
  "role": "ROLE_USER",
  "name": "tester",
  "iat": 1719200000,
  "exp": 1719201800
}
```

Payload는 암호화가 아니라 인코딩만 되어 있다. 따라서 민감정보를 넣으면 안 된다.

넣으면 안 되는 값:

- 비밀번호
- 토큰 원문
- 주민번호 같은 개인정보
- 내부 보안 설정

### Signature
Header와 Payload가 변조되지 않았는지 검증하기 위한 서명이다.

HS256 기준으로는 서버의 secret key를 사용해 서명한다.

```text
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

클라이언트가 payload를 바꾸면 signature가 맞지 않게 되고, 서버 검증에서 실패한다.

## Access Token과 Refresh Token
보통 JWT 인증에서는 두 종류의 토큰을 나눈다.

### Access Token
API 요청 인증에 사용한다.

- 수명이 짧다.
- 요청마다 `Authorization` 헤더에 실린다.
- 탈취되면 위험하므로 만료 시간을 짧게 둔다.

### Refresh Token
Access Token 재발급에 사용한다.

- Access Token보다 수명이 길다.
- Redis 등에 저장해서 서버에서 무효화 가능하게 관리하는 경우가 많다.
- 로그아웃, 권한 변경, 계정 잠금 시 무효화 대상이 된다.

## 주의할 점
- JWT secret key는 하드코딩하지 않고 외부 설정으로 주입한다.
- HS256 secret key는 최소 32바이트 이상이어야 한다.
- JWT payload에는 민감정보를 넣지 않는다.
- 토큰 만료 시간은 반드시 검증한다.
- 권한 변경 또는 계정 잠금 시 기존 토큰 무효화 전략이 필요하다.
- WebSocket 연결에서도 JWT 검증이 필요하다.
- `SecurityContextHolder`는 요청 단위로 인증 정보를 담는 곳이지 영구 저장소가 아니다.

## 현재 프로젝트 기준 다음 단계
- `JwtTokenProvider`에서 access token 생성 완료
- refresh token 생성 구현 필요
- access/refresh token 검증 구현 필요
- `JwtAuthenticationFilter`에서 Authorization 헤더 추출 및 인증 객체 저장 필요
- 로그인 성공 핸들러에서 JWT 발급 필요
- 로그아웃 성공 핸들러에서 refresh token 무효화 필요
- 계정 잠금/권한 변경 시 토큰 무효화 정책 필요

## 관련
