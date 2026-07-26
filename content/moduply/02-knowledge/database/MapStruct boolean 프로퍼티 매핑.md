# MapStruct boolean 프로퍼티 매핑

## 문제
`User` 엔티티에는 필드가 `isBlocked`로 선언되어 있는데, MapStruct에서 `UserDto.locked`로 매핑할 때 `source = "isBlocked"`를 쓰면 컴파일 오류가 발생했다.

```java
@Mapping(target = "locked", source = "isBlocked")
UserDto toDto(User user);
```

오류 메시지는 `isBlocked`라는 프로퍼티를 찾을 수 없다는 내용이었다.

## 원인
MapStruct는 필드명을 직접 기준으로 삼기보다 Java Bean 프로퍼티 규칙을 따른다.

boolean 필드가 다음처럼 선언되어 있고 Lombok `@Getter`가 붙어 있으면:

```java
private boolean isBlocked;
```

Lombok은 보통 다음 getter를 만든다.

```java
public boolean isBlocked()
```

Java Bean 규칙에서 `isBlocked()` 메서드의 프로퍼티 이름은 `blocked`로 해석된다. 즉 필드명은 `isBlocked`지만, 외부에서 보이는 bean property 이름은 `blocked`다.

그래서 MapStruct source에는 필드명인 `isBlocked`가 아니라 프로퍼티명인 `blocked`를 써야 한다.

```java
@Mapping(target = "locked", source = "blocked")
UserDto toDto(User user);
```

## 해결
`UserMapper`에서는 다음처럼 매핑한다.

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

  @Mapping(target = "locked", source = "blocked")
  @Mapping(target = "profileImageUrl", ignore = true)
  UserDto toDto(User user);
}
```

`profileImageUrl`은 현재 `User`와 `BinaryContent` 테이블 구조가 확정되지 않았기 때문에 의도적으로 ignore 처리한다. 이렇게 해두면 MapStruct 경고 없이 아직 매핑하지 않는 필드라는 의도가 코드에 남는다.

## 정리
- MapStruct는 Java Bean 프로퍼티 이름을 기준으로 매핑한다.
- boolean getter가 `isBlocked()`라면 프로퍼티 이름은 `blocked`다.
- `private boolean isBlocked` 필드라도 MapStruct source는 `blocked`를 사용한다.
- 필드명이 헷갈리면 생성되는 getter 이름과 bean property 이름을 기준으로 판단한다.

## 관련
