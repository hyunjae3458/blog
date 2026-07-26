# JaCoCo 커버리지 기준과 finalizedBy

## 문제
CI에서 dev 브랜치 대상 커버리지 기준을 `0.10`으로 올리고, `build.gradle`의 기본값은 `0.40`으로 설정했다.

의도는 CI에서 `-PcoverageMinimum=0.10`을 넘기면 CI는 10% 기준으로 검증하고, property가 없는 로컬 실행에서는 기본값을 쓰는 것이었다.

하지만 CI가 `0.10`이 아니라 `build.gradle` 기본값인 `0.40` 기준으로 먼저 실패했다.

## 원인
`build.gradle`의 기본값 처리는 정상이다.

```groovy
def coverageMinimum = project.hasProperty('coverageMinimum')
        ? (project.property('coverageMinimum') as String).toBigDecimal()
        : 0.40
```

문제는 CI 실행 순서와 Gradle task 연결이었다.

CI는 다음처럼 실행하고 있었다.

```yaml
- name: Run tests
  run: ./gradlew test

- name: Verify coverage threshold
  run: ./gradlew jacocoTestCoverageVerification -PcoverageMinimum=0.10
```

그런데 `build.gradle`에는 다음 연결이 있었다.

```groovy
tasks.named('test') {
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    finalizedBy 'jacocoTestCoverageVerification'
}
```

따라서 CI의 첫 번째 단계인 `./gradlew test`만 실행해도 실제 실행 흐름은 다음처럼 이어진다.

```text
test
→ jacocoTestReport
→ jacocoTestCoverageVerification
```

이 첫 번째 `./gradlew test` 명령에는 `-PcoverageMinimum=0.10`이 없다. 그래서 `coverageMinimum` property가 존재하지 않고, `build.gradle`의 기본값 `0.40`이 적용된다.

결과적으로 CI의 명시적인 `Verify coverage threshold` 단계가 실행되기 전에 이미 기본값 40% 기준으로 검증이 실패한다.

## 해결
`jacocoTestReport`에서 `jacocoTestCoverageVerification`을 자동 실행하는 연결을 제거한다.

```groovy
jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true
        html.required = true
    }

    classDirectories.setFrom(
            files(classDirectories.files.collect {
                fileTree(dir: it, exclude: excludes)
            })
    )

    // finalizedBy 'jacocoTestCoverageVerification' 제거
}
```

`test.finalizedBy jacocoTestReport`는 유지해도 된다. 테스트 실행 후 Codecov 업로드용 XML 리포트를 생성하는 목적이므로 자연스럽다.

이렇게 바꾸면 CI 흐름이 명확해진다.

```text
./gradlew test
→ 테스트 실행
→ jacocoTestReport 생성

./gradlew jacocoTestCoverageVerification -PcoverageMinimum=0.10
→ CI에서 넘긴 기준으로 커버리지 검증
```

## 유지할 설정
`build`에 커버리지 검증을 연결하는 설정은 유지한다.

```groovy
build {
    dependsOn 'jacocoTestCoverageVerification'
}
```

이 설정 덕분에 로컬이나 CI에서 `./gradlew build`를 실행하면 커버리지 검증이 여전히 강제된다.

## 정리
- `coverageMinimum` 기본값은 property가 없을 때만 사용된다.
- CI의 `./gradlew test` 단계에는 `-PcoverageMinimum`이 없었다.
- `test.finalizedBy(jacocoTestReport)`와 `jacocoTestReport.finalizedBy(jacocoTestCoverageVerification)` 때문에 `test` 단계에서 검증까지 실행됐다.
- 그래서 CI의 별도 검증 단계 전에 기본값 기준으로 실패했다.
- 해결은 `jacocoTestReport.finalizedBy 'jacocoTestCoverageVerification'`를 제거하고, 검증은 명시적으로 실행하는 것이다.

## 관련
