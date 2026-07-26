# JWT 생성 공통화와 Builder 패턴

## 문제
Access Token과 Refresh Token은 생성 과정이 거의 비슷하다.

둘 다 다음 과정을 거친다.

```text
claim 구성
→ JWTClaimsSet 생성
→ JWSHeader 생성
→ SignedJWT 생성
→ secret key로 서명
→ 문자열로 직렬화
```

차이는 payload에 어떤 claim을 넣는지와 만료 시간뿐이다.

- Access Token: API 요청 인증에 필요한 사용자 정보와 권한 포함
- Refresh Token: 재발급에 필요한 최소 정보만 포함

그래서 `generateAccessToken`, `generateRefreshToken`이 각각 모든 로직을 반복하기보다, 공통 `generateToken` 메서드로 빼는 방식이 깔끔하다.

## 해결
`generateAccessToken`, `generateRefreshToken`은 token type과 만료 시간만 넘긴다.

```java
public String generateAccessToken(Authentication authentication) {
  return generateToken(authentication, ACCESS_TOKEN_TYPE, accessTokenExpirationMinutes);
}

public String generateRefreshToken(Authentication authentication) {
  return generateToken(authentication, REFRESH_TOKEN_TYPE, refreshTokenExpirationMinutes);
}
```

실제 생성 로직은 `generateToken`에 모은다.

```java
private String generateToken(
    Authentication authentication,
    String tokenType,
    int expirationMinutes
) {
  ...
}
```

## JWTClaimsSet.Builder 사용 방식
Nimbus의 `JWTClaimsSet.Builder`는 빌더 패턴을 사용한다.

처음에 공통 claim을 넣어 빌더를 만들고:

```java
JWTClaimsSet.Builder claimsBuilder = new JWTClaimsSet.Builder()
    .subject(userDto.getId().toString())
    .claim("type", tokenType)
    .issueTime(issuedAt)
    .expirationTime(expiresAt);
```

토큰 종류에 따라 필요한 claim을 추가로 붙일 수 있다.

Access Token이면 사용자 요청 인증에 필요한 정보를 추가한다.

```java
if (ACCESS_TOKEN_TYPE.equals(tokenType)) {
  claimsBuilder
      .claim("email", userDto.getEmail())
      .claim("role", role)
      .claim("name", userDto.getName());
}
```

Refresh Token이면 토큰 자체를 식별하기 위한 `jti`를 추가한다.

```java
if (REFRESH_TOKEN_TYPE.equals(tokenType)) {
  claimsBuilder.jwtID(UUID.randomUUID().toString());
}
```

마지막에 한 번만 `build()`를 호출해 최종 `JWTClaimsSet`을 만든다.

```java
SignedJWT signedJWT = new SignedJWT(
    new JWSHeader(JWSAlgorithm.HS256),
    claimsBuilder.build()
);
```

즉 빌더는 최종 객체를 바로 만드는 것이 아니라, 최종 객체를 만들기 전까지 값을 계속 누적하는 역할을 한다.

```text
Builder 생성
→ 공통 claim 추가
→ access/refresh 조건별 claim 추가
→ build()
→ JWTClaimsSet 완성
```

## 왜 이 방식이 좋은가
중복이 줄어든다.

Access Token과 Refresh Token이 둘 다 서명 방식, 발급 시간, 만료 시간 계산, 직렬화 과정을 공유한다. 이 공통 과정을 한 곳에 두면 나중에 서명 알고리즘이나 예외 처리를 바꿀 때 한 번만 수정하면 된다.

토큰별 차이가 명확해진다.

```java
if (ACCESS_TOKEN_TYPE.equals(tokenType)) {
  ...
}

if (REFRESH_TOKEN_TYPE.equals(tokenType)) {
  ...
}
```

이렇게 보면 access token에는 어떤 claim이 들어가고, refresh token에는 어떤 claim이 들어가는지 한눈에 보인다.

Refresh Token payload를 최소화하기 쉽다.

Access Token과 Refresh Token을 완전히 따로 구현하면 실수로 refresh token에도 role, email, name 같은 값을 그대로 넣기 쉽다. 공통 메서드 안에서 token type별 claim을 분리하면 refresh token에는 `sub`, `type`, `jti`, `iat`, `exp` 정도만 넣도록 관리하기 쉽다.

## 현재 구현 흐름
현재 `generateToken` 흐름은 다음과 같다.

```text
1. MACSigner 생성
2. Authentication에서 ModuPlyUserDetails 추출
3. UserDto 추출
4. issuedAt, expiresAt 계산
5. JWTClaimsSet.Builder에 공통 claim 설정
   - sub
   - type
   - iat
   - exp
6. access token이면 email, role, name 추가
7. refresh token이면 jti 추가
8. SignedJWT 생성
9. signingKey로 서명
10. serialize()로 문자열 반환
```

## Access Token claim
```text
sub: userId
type: access
email
role
name
iat
exp
```

## Refresh Token claim
```text
sub: userId
type: refresh
jti
iat
exp
```

## 주의할 점
- Builder에 값을 추가한 뒤 마지막에 `build()`를 호출해야 최종 `JWTClaimsSet`이 만들어진다.
- `JWTClaimsSet` 자체는 완성된 값이고, 이후 수정하는 대상이 아니다.
- Refresh Token에는 권한이나 이름 같은 요청 처리용 정보를 넣지 않는 편이 좋다.
- Access Token의 `sub`는 이메일보다 변경 가능성이 낮은 `userId`가 더 적합하다.
- 나중에 token 검증 시 `type` claim을 확인해서 access token과 refresh token을 혼용하지 못하게 해야 한다.

## 관련
[[02-knowledge/auth/회원가입 로그인 JWT 요청 흐름]]
