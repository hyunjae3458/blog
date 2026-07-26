---
tags: [infra, aws, iam, security, decision]
date: 2026-07-07
status: 채택됨
---

# AWS IAM 계정 및 권한 세팅

## 결정
팀원들의 AWS 접근 권한은 Root 계정이 아니라 IAM User와 IAM Group으로 분리해 운영한다.

- 관리자 계정은 인프라 뼈대 생성과 전역 설정 변경에만 사용한다.
- 실무자는 ECR, S3 등 배포와 개발에 필요한 권한만 가진 그룹에 넣는다.
- 팀원이 자신의 Access Key와 MFA만 관리할 수 있도록 커스텀 정책을 부여한다.

## 배경 / 왜 이 결정이 필요했는가
ECS, ECR, S3, RDS 배포 작업을 팀원이 함께 수행해야 했다. Root 계정이나 관리자 권한을 모두에게 공유하면 실수로 인프라를 삭제하거나 민감 자격 증명을 노출할 위험이 커진다.

그래서 최소 권한 원칙을 기준으로 관리자와 실무자를 분리하고, 개인별 보안 자격 증명은 본인만 관리할 수 있게 만들었다.

## 고려한 대안

| 대안 | 장점 | 단점 |
|---|---|---|
| Root 계정 공유 | 설정이 가장 빠름 | 보안 사고 시 추적과 통제가 거의 불가능 |
| 모든 팀원에게 AdministratorAccess 부여 | 개발 중 막히는 권한 문제가 적음 | 실수 한 번으로 전체 인프라 영향 가능 |
| 관리자/실무자 그룹 분리 + 본인 자격 증명만 관리 (채택) | 권한 범위가 명확하고 사고 범위를 줄일 수 있음 | 필요한 권한이 생길 때 정책을 계속 조정해야 함 |

## 계정 및 그룹 분리 전략
최소 권한의 원칙에 따라 관리자 계정, 실무자 그룹, 조회 중심 모니터링 그룹을 분리한다.

### 관리자 계정
- 계정명: `moduply-admin`
- 권한: `AdministratorAccess`
- 역할: VPC, EC2, RDS 등 인프라 뼈대 생성과 전역 설정 변경 시에만 사용한다.
- 운영 원칙: 실무 그룹에는 포함하지 않는다.

### 실무자 그룹
- 그룹명: `Backend-Developers`
- 권한: `AmazonEC2ContainerRegistryPowerUser`, `AmazonS3FullAccess` 등 개발과 배포에 필요한 권한만 부여한다.
- 역할: 코드 배포, Docker 이미지 push, 로그 확인 등 일상적인 실무 작업을 수행한다.

### 리드/모니터링 그룹
- 필요 시 운영한다.
- 권한: `AmazonRDSReadOnlyAccess`, `CloudWatchFullAccess`, ECS Read 정책 등 조회 중심 권한을 부여한다.
- 역할: 운영 상태 확인, 장애 모니터링, 로그 및 메트릭 조회를 담당한다.

## 커스텀 보안 정책
팀원이 자신의 보안 자격 증명만 직접 관리할 수 있도록 커스텀 정책을 생성했다. 타인의 Access Key 또는 MFA 설정은 변경할 수 없게 제한한다.

- 정책명: `moduply-manage-own-accesskey-policy`
- 목적: IAM User가 자신의 Access Key와 MFA 기기만 생성, 삭제, 조회, 갱신할 수 있도록 제한한다.
- 추가 정책: `IAMUserChangePassword` AWS 관리형 정책을 함께 부여해 사용자가 자신의 로그인 비밀번호를 변경할 수 있도록 했다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListVirtualMFADevices",
      "Effect": "Allow",
      "Action": [
        "iam:ListVirtualMFADevices"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowManageOwnCredentials",
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:DeleteAccessKey",
        "iam:ListAccessKeys",
        "iam:UpdateAccessKey",
        "iam:GetUser",
        "iam:CreateVirtualMFADevice",
        "iam:DeleteVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ResyncMFADevice",
        "iam:ListMFADevices",
        "iam:DeactivateMFADevice"
      ],
      "Resource": [
        "arn:aws:iam::<AWS_ACCOUNT_ID>:user/${aws:username}",
        "arn:aws:iam::<AWS_ACCOUNT_ID>:mfa/${aws:username}"
      ]
    }
  ]
}
```

## 팀원 계정 생성 및 온보딩
명확한 네이밍 규칙을 사용해 계정을 생성하고 안전하게 초기 정보를 전달한다.

- 네이밍 규칙: `이름.성` 형식 사용. 예: `jiwon.kim`
- 초기 임시 비밀번호를 발급한다.
- `사용자가 다음 로그인 시 새 암호를 생성해야 함` 옵션을 활성화해 최초 로그인 시 비밀번호 변경을 강제한다.
- 팀원 전용 로그인 URL 또는 계정 별칭 URL과 초기 접속 정보를 전달한다.
- MFA 설정을 필수로 안내한다.

## 관리자 체크리스트
- [x] Admin 계정과 개발자 그룹 분리
- [x] 커스텀 자격 증명 관리 정책 생성 및 적용
- [x] 팀원 계정 생성 및 그룹 할당
- [x] 팀원별 초기 로그인 및 MFA 설정 확인

## 트레이드오프 / 감수한 것
- 실무자가 새 AWS 리소스에 접근해야 할 때 권한 추가 작업이 필요하다.
- 초기에 정책 설계 시간이 더 든다.
- 권한이 부족한 오류와 실제 설정 오류를 구분해야 하므로 온보딩 문서가 필요하다.

## 재검토 시점
- 배포 자동화가 GitHub Actions 또는 별도 CI/CD로 넘어가는 시점
- ECS, RDS, CloudWatch 운영 권한을 팀원별로 더 세분화해야 하는 시점
- Access Key 대신 IAM Role, OIDC, SSO 중심 운영으로 전환하는 시점

## 관련 노트
- [[02-knowledge/infra/ECS EC2 첫 배포 트러블슈팅]]
