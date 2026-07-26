# JWT 액세스/리프레시 토큰 타입 검증

## 문제
Access Token과 Refresh Token은 둘 다 JWT이고 같은 secret key로 서명된다.

따라서 검증 로직에서 서명과 만료 시간만 확인하면 Refresh Token이 Access Token처럼 통과할 수 있다.

예를 들어 Refresh Token이 다음 조건을 만족한다고 가정한다.

```text
서명 정상
만료 안 됨
JWT 형식 정상
```

이 상태에서 `validateAccessToken()`이 서명과 만료만 확인하면, 클라이언트가 실수로 또는 악의적으로 다음 요청을 보내도 통과할 수 있다.

```http
Authorization: Bearer {refreshToken}
```

이렇게 되면 Refresh Token이 API 요청 인증용 토큰처럼 사용된다.

## 원인
Access Token과 Refresh Token은 목적이 다르다.

```text
Access Token  = API 요청 인증용
Refresh Token = Access Token 재발급용
```

하지만 기술적으로는 둘 다 JWT라서 같은 방식으로 파싱하고 서명 검증할 수 있다.

그래서 “우리 서버가 발급했고 아직 만료되지 않았다”만으로는 부족하다.

토큰의 용도까지 확인해야 한다.

```text
서명 검증 = 우리 서버가 발급한 토큰인가?
만료 검증 = 아직 유효 기간 안인가?
type 검증 = 이 용도에 맞는 토큰인가?
```

## 해결
토큰 생성 시 `type` claim을 넣는다.

Access Token:

```text
type: access
```

Refresh Token:

```text
type: refresh
```

검증 시에는 공통 검증 메서드에 기대하는 token type을 넘긴다.

```java
public boolean validateAccessToken(String token) {
  return validateToken(token, ACCESS_TOKEN_TYPE);
}

public boolean validateRefreshToken(String token) {
  return validateToken(token, REFRESH_TOKEN_TYPE);
}
```

공통 메서드에서 parse, alg, signature, exp, type을 한 번에 검증한다.

```java
private boolean validateToken(String token, String expectedTokenType) {
  try {
    SignedJWT signedJWT = SignedJWT.parse(token);

    boolean isExpectedAlgorithm =
        JWSAlgorithm.HS256.equals(signedJWT.getHeader().getAlgorithm());

    JWSVerifier verifier = new MACVerifier(signingKey);
    boolean isValidSignature = signedJWT.verify(verifier);

    JWTClaimsSet claims = signedJWT.getJWTClaimsSet();

    Date expirationTime = claims.getExpirationTime();
    boolean isExpired = expirationTime == null || expirationTime.before(new Date());

    String tokenType = claims.getStringClaim("type");
    boolean isExpectedTokenType = expectedTokenType.equals(tokenType);

    return isExpectedAlgorithm
        && isValidSignature
        && !isExpired
        && isExpectedTokenType;
  } catch (ParseException | JOSEException e) {
    log.warn("유효하지 않은 JWT 토큰입니다. expectedType={}", expectedTokenType, e);
    return false;
  }
}
```

## 왜 공통 메서드 안에서 type을 검증하는가
`type` claim을 확인하려면 어차피 JWT를 파싱해야 한다.

각 메서드에서 먼저 type을 확인하고 공통 메서드를 호출하면 다음 문제가 생긴다.

```text
validateAccessToken
→ parse
→ type 확인
→ validateToken
→ 다시 parse
```

중복 파싱이 생기거나, `SignedJWT`를 넘겨받는 구조로 복잡해진다.

따라서 다음 구조가 더 단순하다.

```text
validateAccessToken(token)
→ validateToken(token, "access")

validateRefreshToken(token)
→ validateToken(token, "refresh")
```

검증 규칙은 공통 메서드에 모이고, public 메서드는 의미만 드러낸다.

## 정리
- Refresh Token은 Access Token처럼 API 요청 인증에 쓰이면 안 된다.
- 서명과 만료만 검증하면 Refresh Token도 Access Token 검증을 통과할 수 있다.
- `type` claim으로 토큰 용도를 분리한다.
- `validateToken(token, expectedType)` 구조로 공통 검증한다.
- Refresh Token 검증은 나중에 Redis의 `jti` 존재 여부까지 확인해야 한다.

## 관련
[[02-knowledge/auth/JWT 생성 공통화와 Builder 패턴]]
