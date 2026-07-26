---
tags: [ci-cd, github-actions, aws, ecs, ecr, docker]
date: 2026-07-08
status: 구현됨
---

# GitHub Actions ECS CD 구현

## 한 줄 요약
> `main`에 merge된 검증 완료 코드를 Docker image로 만들고 ECR에 push한 뒤, ECS EC2 서비스의 task definition을 새 image로 교체해 배포하도록 CD를 구성했다.

## 배경 / 어떤 상황이었는가
수동으로 Docker image를 buildx로 빌드하고 ECR에 push한 뒤 ECS service를 업데이트하던 흐름을 GitHub Actions로 자동화해야 했다.

현재 운영 구조는 다음과 같다.

- ECR repository: `moduply-repo`
- ECS cluster: `moduply-prod-cluster`
- ECS service: `moduply-service`
- ECS task family: `moduply-task`
- container name: `moduply-app`
- launch type: EC2
- 운영 인스턴스: `t3.small` 단일 EC2
- 앱 컨테이너와 Redis 컨테이너를 같은 EC2에서 운영

## 문제 / 고려한 제약
- Dockerfile에서는 이미지 생성 속도를 위해 테스트를 제외한다.
- 검증은 PR 단계의 CI와 GitHub branch protection에서 보장한다.
- 단일 `t3.small`에서는 기존 앱 task와 새 앱 task를 동시에 띄울 여유가 부족할 수 있다.
- 새 task가 env, RDS, Redis, memory 문제로 기동 실패하면 서비스가 내려간 상태로 남을 수 있다.
- 문서나 노트 수정만으로 Docker build/ECS deploy가 실행되면 비용과 시간이 낭비된다.

## 실제 구현 흐름

CD 트리거는 `main` push다. PR이 `main`에 merge되면 merge commit 또는 squash commit이 `main`에 push되므로 CD가 실행된다.

```yaml
on:
  push:
    branches: [ main ]
    paths-ignore:
      - '**/*.md'
      - '**/*.mdx'
      - 'docs/**'
      - 'notes/**'
      - '.gitignore'
```

문서성 변경은 배포 대상이 아니므로 `paths-ignore`로 제외했다.

## build-and-push job

`build-and-push` job은 Docker image를 만들고 ECR에 push한다.

```yaml
docker build \
  -t "$ECR_URI:latest" \
  -t "$ECR_URI:$SHA" \
  .
docker push "$ECR_URI:latest"
docker push "$ECR_URI:$SHA"
echo "uri=$ECR_URI:$SHA" >> "$GITHUB_OUTPUT"
```

태그는 2개를 사용한다.

- `latest`: 사람이 콘솔에서 최신 이미지를 확인하기 쉬움
- `${github.sha}`: 실제 ECS 배포에는 커밋 SHA 태그를 사용해 어떤 코드가 배포됐는지 추적 가능

## deploy job

`deploy` job은 현재 task definition을 가져와 새 image URI만 교체한 뒤 새 revision을 등록한다.

```yaml
aws ecs describe-task-definition \
  --task-definition ${{ vars.ECS_TASK_DEFINITION }} \
  --query taskDefinition \
  | jq 'del(...)' > task-definition.json
```

`describe-task-definition` 결과에는 `register-task-definition`에서 거부하는 읽기 전용 필드가 포함되므로 `jq`로 제거한다.

이미지 교체는 `aws-actions/amazon-ecs-render-task-definition@v1`을 사용한다.

```yaml
with:
  task-definition: task-definition.json
  container-name: moduply-app
  image: ${{ needs.build-and-push.outputs.image-uri }}
```

## 단일 EC2 용량 확보

단일 `t3.small`에서는 기존 앱 task와 새 앱 task가 동시에 떠야 하는 rolling deployment가 CPU/memory 부족으로 실패할 수 있다.

그래서 기존 서비스를 먼저 내린다.

```yaml
aws ecs update-service \
  --cluster ${{ vars.ECS_CLUSTER }} \
  --service ${{ vars.ECS_SERVICE }} \
  --desired-count 0
aws ecs wait services-stable ...
```

그 다음 새 task definition으로 다시 올린다.

```yaml
aws ecs update-service \
  --cluster ${{ vars.ECS_CLUSTER }} \
  --service ${{ vars.ECS_SERVICE }} \
  --task-definition ${{ steps.register.outputs.arn }} \
  --desired-count 1
aws ecs wait services-stable ...
```

이 방식은 무중단 배포가 아니다. 비용과 단일 인스턴스 제약을 우선한 배포 방식이다.

## 실패 시 롤백

기존 task를 내린 뒤 새 task가 실패하면 서비스가 내려간 상태로 남을 수 있다. 그래서 배포 전에 현재 service가 사용 중인 task definition ARN을 저장한다.

```yaml
CURRENT_TD_ARN=$(aws ecs describe-services \
  --cluster ${{ vars.ECS_CLUSTER }} \
  --services ${{ vars.ECS_SERVICE }} \
  --query 'services[0].taskDefinition' \
  --output text)
```

배포 실패 시 `if: failure()` 조건으로 이전 task definition을 다시 적용한다.

```yaml
aws ecs update-service \
  --cluster ${{ vars.ECS_CLUSTER }} \
  --service ${{ vars.ECS_SERVICE }} \
  --task-definition "$PREVIOUS_TASK_DEFINITION" \
  --desired-count 1
```

## GitHub Variables / Secrets

CD에 필요한 GitHub 설정값은 다음과 같다.

Secrets:

- `AWS_ACCESS_KEY`
- `AWS_SECRET_KEY`

Variables:

- `AWS_REGION`
- `ECR_REPOSITORY_URI`
- `ECS_TASK_DEFINITION`
- `ECS_CLUSTER`
- `ECS_SERVICE`

현재는 Access Key 방식으로 구현했다. 추후 보안 수준을 올릴 때 GitHub OIDC와 IAM Role assume 방식으로 전환을 검토한다.

## 결과 / 검증 방법
- GitHub Actions에서 CD workflow가 성공해야 한다.
- ECR에 `latest`, `${github.sha}` 태그가 push되어야 한다.
- ECS task definition에 새 revision이 생성되어야 한다.
- ECS service가 새 task definition으로 steady state에 도달해야 한다.
- 실패 시 rollback step이 이전 task definition으로 service를 복구해야 한다.

## 배운 점 / 다음에 또 마주치면
- ECS EC2 단일 인스턴스 배포는 rolling deployment보다 리소스 확보 전략이 먼저 필요하다.
- Docker image는 테스트 없이 빠르게 만들 수 있지만, 그 전 단계에서 PR CI와 branch protection이 반드시 배포 품질을 보장해야 한다.
- ECS 서비스 이벤트에는 과거 이벤트가 함께 보일 수 있으므로, 장애 판단은 이벤트 시간과 최신 deployment 상태를 같이 봐야 한다.

## 관련 노트
- [[03-decisions/ci-cd/main merge 후 ECS CD 배포 전략]]
- [[03-decisions/infra/ECS EC2 단일 인스턴스 최소 비용 배포 결정]]
- [[02-knowledge/infra/단일 EC2 ECS task placement와 리소스 부족]]
