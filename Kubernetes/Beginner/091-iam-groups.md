# IAM 그룹을 사용하여 KUBERNETES 클러스터 액세스 관리

이 장에서는 IAM 그룹에 따라 kubernetes 클러스터의 다른 부분에 대한 액세스를 단순화하는 방법을 배웁니다.

## 쿠버네티스 인증
에서 RBAC 소개 모듈에서 개별 사용자에게 Kubernetes에 대한 액세스 권한을 부여하는 방법을 살펴보았습니다.

다른 종류의 클러스터 액세스가 필요한 다른 팀이 있는 경우 액세스 권한을 부여하거나 제거할 각 EKS 클러스터에 대한 액세스를 수동으로 추가하거나 제거하기 어려울 것입니다.

AWS에서 활용할 수 있습니다. IAM 그룹 사용자를 쉽게 추가 또는 제거하고 전체 클러스터에 대한 권한을 부여하거나 그들이 속한 그룹에 따라 일부만 권한을 부여합니다.

이 레슨에서는 3개의 IAM 그룹에 매핑할 3개의 IAM 역할을 생성합니다.

## IAM 역할 생성
우리는 3개의 역할을 생성할 것입니다:

- k8sAdmin의 것 역할 관리자 우리 EKS 클러스터의 권한을

- EKS 클러스터 의 개발자 네임스페이스에 대한 액세스 권한을 부여 하는 k8sDev 역할

- EKS 클러스터 의 통합 네임스페이스에 대한 액세스 권한을 부여 하는 k8sInteg 역할

역할 생성:

```
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

echo ACCOUNT_ID=$ACCOUNT_ID
echo POLICY=$POLICY

aws iam create-role \
  --role-name k8sAdmin \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'

aws iam create-role \
  --role-name k8sDev \
  --description "Kubernetes developer role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
  
aws iam create-role \
  --role-name k8sInteg \
  --description "Kubernetes role for integration namespace in quick cluster." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
```

> 이 예에서 assign-role-policy는 루트 계정이 역할을 맡도록 허용합니다. 특정 그룹도 이러한 역할을 맡을 수 있도록 허용할 것입니다. 자세한 내용을 체크 해봐[공식 문서](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html).

위의 역할은 EKS 클러스터 내에서 인증하는 데만 사용되기 때문에 AWS 권한이 필요하지 않습니다. EKS 클러스터에 액세스하기 위해 일부 IAM 그룹이 이 역할을 맡도록 허용하는 데만 사용할 것입니다.

## IAM 그룹 생성
kubernetes 클러스터에서 다른 권한을 갖기 위해 특정 IAM 그룹에 추가될 다른 IAM 사용자를 원합니다.

우리는 3개의 그룹을 정의할 것입니다:

- k8sAdmin - 이 그룹의 사용자는 kubernetes 클러스터에 대한 관리자 권한을 갖습니다.

- k8sDev - 이 그룹의 사용자는 클러스터의 개발 네임스페이스에서만 전체 액세스 권한을 갖습니다.

- k8sInteg - 이 그룹의 사용자는 통합 네임스페이스에 액세스할 수 있습니다.

> 실제로 k8sDev 및 k8sInteg 그룹의 사용자는 연결된 kubernetes 역할에 대해 kubernetes RBAC 액세스를 정의할 네임스페이스에만 액세스할 수 있습니다. 우리는 이것을 볼 것이지만 먼저 그룹을 생성합시다.

### k8sAdmin IAM 그룹 생성
k8sAdmin 그룹은 가정 허용됩니다 k8sAdmin IAM 역할을.

```
aws iam create-group --group-name k8sAdmin
```

이 그룹의 사용자가 k8sAdmin 역할을 맡도록 허용하는 정책을 그룹에 추가해 보겠습니다.

```
ADMIN_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sAdmin"
    }
  ]
}')

echo ADMIN_GROUP_POLICY=$ADMIN_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sAdmin \
--policy-name k8sAdmin-policy \
--policy-document "$ADMIN_GROUP_POLICY"
```

### k8sDev IAM 그룹 생성
k8sDev 그룹은 가정 허용됩니다 k8sDev IAM 역할을.

```
aws iam create-group --group-name k8sDev
```

이 그룹의 사용자가 k8sDev 역할을 맡도록 허용하는 정책을 그룹에 추가해 보겠습니다.

```
DEV_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sDev"
    }
  ]
}')

echo DEV_GROUP_POLICY=$DEV_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sDev \
--policy-name k8sDev-policy \
--policy-document "$DEV_GROUP_POLICY"
```

### k8sInteg IAM 그룹 생성

```
aws iam create-group --group-name k8sInteg
```

이 그룹의 사용자가 k8sInteg 역할을 맡도록 허용하는 정책을 그룹에 추가해 보겠습니다.

```
INTEG_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sInteg"
    }
  ]
}')

echo INTEG_GROUP_POLICY=$INTEG_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sInteg \
--policy-name k8sInteg-policy \
--policy-document "$INTEG_GROUP_POLICY"
```

이제 3개의 그룹이 있어야 합니다.

```
aws iam list-groups
```

```
{
    "Groups": [
        {
            "Path": "/",
            "GroupName": "k8sAdmin",
            "GroupId": "AGPAZRV3OHPJZGT2JKVDV",
            "Arn": "arn:aws:iam::xxxxxxxxxx:group/k8sAdmin",
            "CreateDate": "2020-04-07T13:32:52Z"
        },
        {
            "Path": "/",
            "GroupName": "k8sDev",
            "GroupId": "AGPAZRV3OHPJUOBR375KI",
            "Arn": "arn:aws:iam::xxxxxxxxxx:group/k8sDev",
            "CreateDate": "2020-04-07T13:33:15Z"
        },
        {
            "Path": "/",
            "GroupName": "k8sInteg",
            "GroupId": "AGPAZRV3OHPJR6GM6PFDG",
            "Arn": "arn:aws:iam::xxxxxxxxxx:group/k8sInteg",
            "CreateDate": "2020-04-07T13:33:25Z"
        }
    ]
}
```

## IAM 사용자 생성
시나리오를 테스트하기 위해 생성한 각 그룹에 대해 하나씩 3명의 사용자를 생성합니다.

```
aws iam create-user --user-name PaulAdmin
aws iam create-user --user-name JeanDev
aws iam create-user --user-name PierreInteg
```

연결된 그룹에 사용자 추가:

```
aws iam add-user-to-group --group-name k8sAdmin --user-name PaulAdmin
aws iam add-user-to-group --group-name k8sDev --user-name JeanDev
aws iam add-user-to-group --group-name k8sInteg --user-name PierreInteg
```

사용자가 그룹에 올바르게 추가되었는지 확인하십시오.

```
aws iam get-group --group-name k8sAdmin
aws iam get-group --group-name k8sDev
aws iam get-group --group-name k8sInteg
```

```
간단하게 하기 위해 이 장에서는 사용자 간에 쉽게 전환할 수 있도록 자격 증명을 파일에 저장합니다. 프로덕션 환경에서 또는 권한 있는 액세스 권한이 있는 자격 증명을 사용하여 이 작업을 수행하지 마십시오. 파일 시스템에 자격 증명을 저장하는 것은 보안 모범 사례가 아닙니다.
```

가짜 사용자의 액세스 키 검색:

```
aws iam create-access-key --user-name PaulAdmin | tee /tmp/PaulAdmin.json
aws iam create-access-key --user-name JeanDev | tee /tmp/JeanDev.json
aws iam create-access-key --user-name PierreInteg | tee /tmp/PierreInteg.json
```

요약:

- PaulAdmin 은 k8sAdmin 그룹에 있으며 k8sAdmin 역할 을 맡을 수 있습니다 .

- JeanDev 는 k8sDev Group에 있으며 IAM 역할 k8sDev 를 맡을 수 있습니다.

- PierreInteg 는 k8sInteg 그룹에 있으며 IAM 역할 k8sInteg 를 맡을 수 있습니다.

## KUBERNETES RBAC 구성
당신은 참조 할 수 있습니다 RBAC 소개 Kubernetes RBAC의 기본을 이해하는 모듈입니다.

### Kubernetes 네임스페이스 생성

- k8sDev 그룹의 IAM 사용자가 개발 네임스페이스에 액세스할 수 있습니다.

- k8sInteg 그룹의 IAM 사용자는 통합 네임스페이스에 액세스할 수 있습니다.

```
kubectl create namespace integration
kubectl create namespace development
```

### 개발 네임스페이스에 대한 액세스 구성
우리는는 Kubernetes를 생성 role및 rolebinding개발 네임 스페이스의는 Kubernetes 사용자에 대한 전체 액세스 권한 부여 DEV-사용자

```
cat << EOF | kubectl apply -f - -n development
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-role
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
    resources:
      - "configmaps"
      - "cronjobs"
      - "deployments"
      - "events"
      - "ingresses"
      - "jobs"
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "secrets"
      - "services"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dev-role-binding
subjects:
- kind: User
  name: dev-user
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

우리가 정의한 역할은 해당 네임스페이스의 모든 것에 대한 전체 액세스 권한을 부여합니다. ClusterRole이 아닌 Role이므로 개발 네임스페이스 에만 적용할 예정 입니다.

원하는 네임스페이스에 자유롭게 적용하거나 복제할 수 있습니다.

### 통합 네임스페이스에 대한 액세스 구성
kubernetes 사용자 integ-user로 전체 액세스를 위해 kubernetes role및 rolebinding통합 네임스페이스를 생성합니다.

```
cat << EOF | kubectl apply -f - -n integration
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: integ-role
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
    resources:
      - "configmaps"
      - "cronjobs"
      - "deployments"
      - "events"
      - "ingresses"
      - "jobs"
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "secrets"
      - "services"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: integ-role-binding
subjects:
- kind: User
  name: integ-user
roleRef:
  kind: Role
  name: integ-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

우리가 정의한 역할은 해당 네임스페이스의 모든 것에 대한 전체 액세스 권한을 부여합니다. 가 Role아니라 ClusterRole이므로 통합 네임스페이스 에만 적용됩니다 .

## KUBERNETES 역할 액세스 구성
### EKS 클러스터에 IAM 역할에 대한 액세스 권한 부여
이전에 정의한 IAM 역할에 대한 액세스 권한을 EKS 클러스터에 부여하려면 ConfigMap에 특정 mapRoles 를 추가해야 합니다.aws-auth

IAM 사용자를 직접 지정하는 대신 역할을 사용하여 클러스터에 액세스할 때의 이점은 관리가 더 쉽다는 것입니다. 사용자를 추가하거나 제거할 때마다 ConfigMap을 업데이트할 필요가 없습니다. IAM 그룹에서 사용자를 제거하고 IAM 그룹과 연결된 IAM 역할을 허용하도록 ConfigMap을 구성하기만 하면 됩니다.

### IAM 역할을 허용하도록 aws-auth ConfigMap 업데이트
arn 그룹을 허용하거나 삭제하려면 kube-system 네임스페이스 의 aws-auth ConfigMap을 편집해야 합니다.

이 파일은 IAM 역할과 k8S RBAC 권한을 매핑합니다. 수동으로 수정할 수 있습니다.

우리는 그것을 사용하여 편집할 수 있습니다 [eksctl](https://github.com/weaveworks/eksctl) :

```
eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sDev \
  --username dev-user

eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sInteg \
  --username integ-user

eksctl create iamidentitymapping \
  --cluster eksworkshop-eksctl \
  --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sAdmin \
  --username admin \
  --group system:masters
```

항목을 삭제하는 데에도 사용할 수 있습니다.

> eksctl delete iamidentitymapping --cluster eksworkshop-eksctlv --arn arn:aws:iam::xxxxxxxxxx:role/k8sDev --username dev-user

다음과 같은 구성 맵이 있어야 합니다.

```
kubectl get cm -n kube-system aws-auth -o yaml
```

```
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::xxxxxxxxxx:role/eksctl-eksworkshop-eksctl-nodegro-NodeInstanceRole-14TKBWBD7KWFH
      username: system:node:{{EC2PrivateDNSName}}
    - rolearn: arn:aws:iam::xxxxxxxxxx:role/k8sDev
      username: dev-user
    - rolearn: arn:aws:iam::xxxxxxxxxx:role/k8sInteg
      username: integ-user
    - groups:
      - system:masters
      rolearn: arn:aws:iam::xxxxxxxxxx:role/k8sAdmin
      username: admin
  mapUsers: |
    []
kind: ConfigMap
```

eksctl을 활용하여 클러스터에서 관리되는 모든 ID 목록을 얻을 수 있습니다. 예:

```
eksctl get iamidentitymapping --cluster eksworkshop-eksctl
```

```
arn:aws:iam::xxxxxxxxxx:role/eksctl-quick-nodegroup-ng-fe1bbb6-NodeInstanceRole-1KRYARWGGHPTT	system:node:{{EC2PrivateDNSName}}	system:bootstrappers,system:nodes
arn:aws:iam::xxxxxxxxxx:role/k8sAdmin           admin					system:masters
arn:aws:iam::xxxxxxxxxx:role/k8sDev             dev-user
arn:aws:iam::xxxxxxxxxx:role/k8sInteg           integ-user
```

여기에 우리가 만들었습니다:

- K8sAdmin에 대한 RBAC 역할, admin 사용자에게 매핑하고 system:masters kubernetes Groups에 대한 액세스 권한을 부여합니다 (전체 관리자 권한을 갖기 위해 )

- 개발 네임스페이스의 dev-user에 매핑하는 k8sDev에 대한 RBAC 역할

- 통합 네임스페이스의 integ-user에 매핑하는 k8sInteg에 대한 RBAC 역할

다음 섹션에서 테스트하는 방법을 살펴보겠습니다.

## EKS 액세스 테스트
### aws cli로 수임 자동화
파일 ~/.aws/config및 에서 AWS CLI를 구성하여 수임된 역할에 대한 임시 자격 증명 검색을 자동화할 수 있습니다 ~/.aws/credentials. 예를 들어 세 가지 프로필을 정의합니다.

추가 ~/.aws/config:

```
mkdir -p ~/.aws

cat << EoF >> ~/.aws/config
[profile admin]
role_arn=arn:aws:iam::${ACCOUNT_ID}:role/k8sAdmin
source_profile=eksAdmin

[profile dev]
role_arn=arn:aws:iam::${ACCOUNT_ID}:role/k8sDev
source_profile=eksDev

[profile integ]
role_arn=arn:aws:iam::${ACCOUNT_ID}:role/k8sInteg
source_profile=eksInteg

EoF
```

추가 ~/.aws/credentials:

```
cat << EoF >> ~/.aws/credentials

[eksAdmin]
aws_access_key_id=$(jq -r .AccessKey.AccessKeyId /tmp/PaulAdmin.json)
aws_secret_access_key=$(jq -r .AccessKey.SecretAccessKey /tmp/PaulAdmin.json)

[eksDev]
aws_access_key_id=$(jq -r .AccessKey.AccessKeyId /tmp/JeanDev.json)
aws_secret_access_key=$(jq -r .AccessKey.SecretAccessKey /tmp/JeanDev.json)

[eksInteg]
aws_access_key_id=$(jq -r .AccessKey.AccessKeyId /tmp/PierreInteg.json)
aws_secret_access_key=$(jq -r .AccessKey.SecretAccessKey /tmp/PierreInteg.json)

EoF
```

개발자 프로필로 이것을 테스트하십시오.

```
aws sts get-caller-identity --profile dev
```

```
{
    "사용자 ID": "AROAUD5VMKW75WJEHFU4X:botocore-session-1581687024",
    "계정": "xxxxxxxxxxx",
    "Arn": "arn:aws:sts::xxxxxxxxxx:assumed-role/k8sDev/botocore-session-1581687024"
}
```

가정된 역할은 k8sDev이므로 목표를 달성했습니다.

–profile dev 매개변수를 지정할 때 k8sDev 역할에 대한 임시 자격 증명을 자동으로 요청합니다. integ 및 admin으로 도 이것을 테스트할 수 있습니다 .

```
aws sts get-caller-identity --profile admin
```

```
{
    "UserId": "AROAUD5VMKW75WJEHFU4X:botocore-session-1581687024",
    "Account": "xxxxxxxxxx",
    "Arn": "arn:aws:sts::xxxxxxxxxx:assumed-role/k8sDev/botocore-session-1581687024"
}
```

> –profile admin 매개변수를 지정할 때 k8sAdmin 역할에 대한 임시 자격 증명을 자동으로 요청합니다.

### Kubectl 구성 파일과 함께 AWS 프로필 사용
~/.kube/config적절한 프로필을 사용하도록 파일 에서 aws-iam-authenticator와 함께 사용할 AWS_PROFILE을 지정할 수도 있습니다 .

### 개발자 프로필
이를 테스트하기 위해 새 KUBECONFIG 파일을 만듭니다.

```
export KUBECONFIG=/tmp/kubeconfig-dev && eksctl utils write-kubeconfig eksworkshop-eksctl
cat $KUBECONFIG | yq e '.users.[].user.exec.args += ["--profile", "dev"]' - -- | sed 's/eksworkshop-eksctl./eksworkshop-eksctl-dev./g' | sponge $KUBECONFIG
```

> 참고: 이것은 yq >= 버전 4를 사용한다고 가정합니다. 이 페이지 이 명령을 다른 버전에 맞게 조정합니다.

--profile devkubectl 구성 파일에 매개변수를 추가하여 kubectl이 dev 프로필과 연결된 IAM 역할을 사용하도록 요청하고 접미사 -dev를 사용하여 컨텍스트 이름을 바꿉니다.

이 구성을 사용하면 RBAC 역할이 정의 되어 있으므로 개발 네임스페이스 와 상호 작용할 수 있어야 합니다 .

포드를 생성해 보겠습니다.

```
kubectl run --generator=run-pod/v1 nginx-dev --image=nginx -n development
```

포드를 나열할 수 있습니다.

```
kubectl get pods -n development
```

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-dev   1/1     Running   0          28h
```

... 하지만 다른 네임스페이스에는 없습니다.

```
kubectl get pods -n integration
```

```
Error from server (Forbidden): pods is forbidden: User "dev-user" cannot list resource "pods" in API group "" in the namespace "integration"
```

### 통합 프로필로 테스트

```
export KUBECONFIG=/tmp/kubeconfig-integ && eksctl utils write-kubeconfig eksworkshop-eksctl
cat $KUBECONFIG | yq e '.users.[].user.exec.args += ["--profile", "integ"]' - -- | sed 's/eksworkshop-eksctl./eksworkshop-eksctl-integ./g' | sponge $KUBECONFIG
```

> 참고: 이것은 yq >= 버전 4를 사용한다고 가정합니다. 이 페이지 이 명령을 다른 버전에 맞게 조정합니다.

포드를 생성해 보겠습니다.

```
kubectl run --generator=run-pod/v1 nginx-integ --image=nginx -n integration
```

포드를 나열할 수 있습니다.

```
kubectl get pods -n integration
```

```
NAME          READY   STATUS    RESTARTS   AGE
nginx-integ   1/1     Running   0          43s
```

... 하지만 다른 네임스페이스에는 없습니다.

```
kubectl get pods -n development
```

서버 오류(금지됨): 포드가 금지됨: 사용자 "integ-user"는 "development" 네임스페이스의 API 그룹 ""에 리소스 "pod"를 나열할 수 없습니다.

### 관리자 프로필로 테스트

```
export KUBECONFIG=/tmp/kubeconfig-admin && eksctl utils write-kubeconfig eksworkshop-eksctl
cat $KUBECONFIG | yq e '.users.[].user.exec.args += ["--profile", "admin"]' - -- | sed 's/eksworkshop-eksctl./eksworkshop-eksctl-admin./g' | sponge $KUBECONFIG
```

> 참고: 이것은 yq >= 버전 4를 사용한다고 가정합니다. 이 페이지 이 명령을 다른 버전에 맞게 조정합니다.

기본 네임스페이스에 포드를 생성해 보겠습니다.

```
kubectl run --generator=run-pod/v1 nginx-admin --image=nginx
```

포드를 나열할 수 있습니다.

```
kubectl get pods
```

```
NAME          READY   STATUS    RESTARTS   AGE
nginx-integ   1/1     Running   0          43s
```

모든 네임스페이스의 모든 포드를 나열할 수 있습니다.

```
kubectl get pods -A
```

```
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
default       nginx-admin                1/1     Running   0          15s
development   nginx-dev                  1/1     Running   0          11m
integration   nginx-integ                1/1     Running   0          4m29s
kube-system   aws-node-mzbh4             1/1     Running   0          100m
kube-system   aws-node-p7nj7             1/1     Running   0          100m
kube-system   aws-node-v2kg9             1/1     Running   0          100m
kube-system   coredns-85bb8bb6bc-2qbx6   1/1     Running   0          105m
kube-system   coredns-85bb8bb6bc-87ndr   1/1     Running   0          105m
kube-system   kube-proxy-4n5lc           1/1     Running   0          100m
kube-system   kube-proxy-b65xm           1/1     Running   0          100m
kube-system   kube-proxy-pr7k7           1/1     Running   0          100m
```

### 다른 컨텍스트 간 전환
동일한 KUBECONFIG 파일에 여러 Kubernetes API 액세스 키를 구성하거나 Kubectl에게 여러 파일을 조회하도록 지시할 수 있습니다.

```
export KUBECONFIG=/tmp/kubeconfig-dev:/tmp/kubeconfig-integ:/tmp/kubeconfig-admin
```

도구가 있다 kubectx / 쿠벤스 여러 컨텍스트로 KUBECONFIG 파일을 관리하는 데 도움이 됩니다.

```
curl -sSLO https://raw.githubusercontent.com/ahmetb/kubectx/master/kubectx && chmod 755 kubectx && sudo mv kubectx /usr/local/bin
```

kubectx를 사용하여 Kubernetes 컨텍스트를 빠르게 나열하거나 전환할 수 있습니다.

```
kubectx
```

```
i-0397aa1339e238a99@eksworkshop-eksctl-admin.eu-west-2.eksctl.io
i-0397aa1339e238a99@eksworkshop-eksctl-dev.eu-west-2.eksctl.io
i-0397aa1339e238a99@eksworkshop-eksctl-integ.eu-west-2.eksctl.io
```

### 결론
이 모듈에서는 IAM 그룹과 Kubernetes RBAC를 결합한 사용자에게 보다 정밀한 액세스를 제공하도록 EKS를 구성하는 방법을 살펴보았습니다. 필요에 따라 다른 그룹을 만들고 클러스터에서 연결된 RBAC 액세스를 구성하고 그룹에서 사용자를 추가하거나 제거하여 클러스터에 대한 액세스 권한을 부여하거나 취소할 수 있습니다.

사용자는 클러스터에서 연결된 권한을 자동으로 검색하기 위해 AWS CLI만 구성하면 됩니다.

# CLEAN UP
이 장을 완료하면 다음 명령을 실행하여 생성한 파일과 리소스를 정리할 수 있습니다.

```
unset KUBECONFIG

kubectl delete namespace development integration
kubectl delete pod nginx-admin

eksctl delete iamidentitymapping --cluster eksworkshop-eksctl --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sAdmin
eksctl delete iamidentitymapping --cluster eksworkshop-eksctl --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sDev
eksctl delete iamidentitymapping --cluster eksworkshop-eksctl --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sInteg

aws iam remove-user-from-group --group-name k8sAdmin --user-name PaulAdmin
aws iam remove-user-from-group --group-name k8sDev --user-name JeanDev
aws iam remove-user-from-group --group-name k8sInteg --user-name PierreInteg

aws iam delete-group-policy --group-name k8sAdmin --policy-name k8sAdmin-policy 
aws iam delete-group-policy --group-name k8sDev --policy-name k8sDev-policy 
aws iam delete-group-policy --group-name k8sInteg --policy-name k8sInteg-policy 

aws iam delete-group --group-name k8sAdmin
aws iam delete-group --group-name k8sDev
aws iam delete-group --group-name k8sInteg

aws iam delete-access-key --user-name PaulAdmin --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/PaulAdmin.json)
aws iam delete-access-key --user-name JeanDev --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/JeanDev.json)
aws iam delete-access-key --user-name PierreInteg --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/PierreInteg.json)

aws iam delete-user --user-name PaulAdmin
aws iam delete-user --user-name JeanDev
aws iam delete-user --user-name PierreInteg

aws iam delete-role --role-name k8sAdmin
aws iam delete-role --role-name k8sDev
aws iam delete-role --role-name k8sInteg

rm /tmp/*.json
rm /tmp/kubeconfig*

# reset aws credentials and config files
rm  ~/.aws/{config,credentials}
aws configure set default.region ${AWS_REGION}
```
