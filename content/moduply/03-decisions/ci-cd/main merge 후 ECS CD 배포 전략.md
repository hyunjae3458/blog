---
tags: [ci-cd, github-actions, ecs, ecr, decision]
date: 2026-07-08
status: 채택됨
---

# main merge 후 ECS CD 배포 전략

## 결정
`main` 브랜치에 merge된 코드는 GitHub Actions CD를 통해 자동 배포한다.

배포 전 테스트는 CD에서 다시 수행하지 않고, PR 단계의 CI와 GitHub branch protection required check로 보장한다.

CD는 다음 역할만 담당한다.

- Docker image build
- ECR push
- ECS task definition 새 revision 등록
- ECS service update
- 배포 실패 시 이전 task definition rollback

## 배경 / 왜 이 결정이 필요했는가
Dockerfile에서는 배포 이미지 생성 속도를 위해 `bootJar -x test`를 사용한다. 테스트는 이미지 생성 중에 수행하는 것이 아니라, `main`에 merge되기 전 CI에서 끝나야 한다.

따라서 배포 기준은 다음과 같이 나누었다.

- PR: 테스트와 커버리지 검증
- main branch protection: CI 통과 전 merge 금지, 직접 push 금지
- CD: 검증된 main 코드를 이미지화하고 ECS에 반영

## 고려한 대안

| 대안 | 장점 | 단점 |
|---|---|---|
| CD에서도 테스트/커버리지 재실행 | main 직접 push에도 안전장치가 있음 | CI와 중복되어 배포 시간이 길어짐 |
| PR CI + branch protection으로 검증 보장, CD는 배포만 담당 (채택) | 역할이 명확하고 배포가 빠름 | GitHub branch protection 설정이 누락되면 위험 |
| Dockerfile에서 `build`로 테스트까지 실행 | 이미지 생성 자체가 검증을 포함 | 이미 검증된 코드를 배포할 때도 매번 느려짐 |

## 구현 방식

CD 트리거:

```yaml
on:
  push:
    branches: [ main ]
```

PR이 `main`에 merge되면 `main`에 새 commit이 생기므로 push trigger가 실행된다.

문서성 변경은 배포하지 않는다.

```yaml
paths-ignore:
  - '**/*.md'
  - '**/*.mdx'
  - 'docs/**'
  - 'notes/**'
  - '.gitignore'
```

배포 image는 `latest`와 commit SHA 태그를 함께 push한다. ECS에는 SHA 태그를 적용해 배포된 코드 추적성을 확보한다.

## 트레이드오프 / 감수한 것
- CD에서 테스트를 재실행하지 않으므로 branch protection 설정이 필수다.
- 단일 EC2 비용 절감 구조라 무중단 배포는 포기했다.
- 기존 task를 내린 뒤 새 task를 올리므로 배포 중 짧은 중단 시간이 발생한다.
- rollback은 이전 task definition 복구까지 담당하지만, 데이터베이스 migration 실패 같은 상태 변경까지 되돌리지는 못한다.

## GitHub에서 반드시 설정할 것
- `main` 직접 push 금지
- PR 없이는 merge 금지
- CI status check 통과 전 merge 금지
- 가능하면 branch 최신화 후 merge 요구

## 재검토 시점
- 도메인/HTTPS 적용을 위해 ALB를 도입할 때
- `t3.small` 리소스 부족으로 배포 실패가 반복될 때
- 무중단 배포가 필요해질 때
- GitHub OIDC + IAM Role 방식으로 AWS 인증을 전환할 때

## 관련 노트
- [[02-knowledge/ci-cd/GitHub Actions ECS CD 구현]]
- [[03-decisions/infra/ECS EC2 단일 인스턴스 최소 비용 배포 결정]]
