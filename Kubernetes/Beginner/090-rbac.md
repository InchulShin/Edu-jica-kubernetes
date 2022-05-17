# RBAC 소개

이 장에서는 Kubernetes에서 RBAC(역할 기반 액세스 제어)가 작동하는 방식에 대해 알아봅니다.

## RBAC 란 무엇입니까?
[공식 쿠버네티스 문서}(https://kubernetes.io/docs/reference/access-authn-authz/rbac/)에 따르면 :

> RBAC(역할 기반 액세스 제어)는 기업 내 개별 사용자의 역할에 따라 컴퓨터 또는 네트워크 리소스에 대한 액세스를 규제하는 방법입니다.

RBAC의 핵심 논리적 구성 요소는 다음과 같습니다.

**엔터티**
그룹, 사용자 또는 서비스 계정(특정 작업(작업)을 실행하기 위해 권한이 필요한 응용 프로그램을 나타내는 ID).

**리소스**
엔터티가 특정 작업을 사용하여 액세스하려는 포드, 서비스 또는 비밀.

**역할**
엔터티가 다양한 리소스에 대해 수행할 수 있는 작업에 대한 규칙을 정의하는 데 사용됩니다.

**역할 바인딩**
지정된 리소스에 대해 연결된 엔터티가 허용하는 작업을 정의하는 규칙 집합을 나타내는 엔터티에 역할을 연결(바인딩)합니다.

역할에는 두 가지 유형(Role, ClusterRole)과 각각의 바인딩(RoleBinding, ClusterRoleBinding)이 있습니다. 이것들은 네임스페이스의 권한 부여와 클러스터 전체의 권한 부여를 구별합니다.

**네임스페이스**
네임스페이스는 보안 경계를 생성하는 훌륭한 방법이며 '네임스페이스' 이름이 암시하는 것처럼 개체 이름에 대한 고유한 범위도 제공합니다. 동일한 물리적 클러스터에 가상 kubernetes 클러스터를 생성하기 위해 다중 테넌트 환경에서 사용하기 위한 것입니다.

### 이 모듈의 목표
이 모듈에서는 rbac-user라는 IAM 사용자를 생성하여 EKS 클러스터에 액세스할 수 있는 인증을 받았지만 (RBAC를 통해) 'rbac-test' 네임스페이스.

이를 위해 IAM 사용자를 만들고 해당 사용자를 kubernetes 역할에 매핑한 다음 해당 사용자의 컨텍스트에서 kubernetes 작업을 수행합니다.

## 테스트 포드 설치
이 자습서에서는 rbac-user라는 사용자에 대해 rbac-test 네임스페이스에서 실행되는 포드에 대한 제한된 액세스를 제공하는 방법을 보여줍니다.

그렇게 하려면 먼저 rbac-test 네임스페이스를 만든 다음 nginx를 여기에 설치합니다.

```
kubectl create namespace rbac-test
kubectl create deploy nginx --image=nginx -n rbac-test
```

테스트 포드가 제대로 설치되었는지 확인하려면 다음을 실행합니다.

```
kubectl get all -n rbac-test
```

출력은 다음과 유사해야 합니다.

```
NAME                       READY   STATUS    RESTARTS   AGE
pod/nginx-5c7588df-8mvxx   1/1     Running   0          48s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           48s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-5c7588df   1         1         1       48s
```

## 사용자 만들기
간단하게 하기 위해 이 장에서는 사용자 간에 쉽게 전환할 수 있도록 자격 증명을 파일에 저장합니다. 프로덕션 환경에서 또는 권한 있는 액세스 권한이 있는 자격 증명을 사용하여 이 작업을 수행하지 마십시오. 파일 시스템에 자격 증명을 저장하는 것은 보안 모범 사례가 아닙니다.

Cloud9 터미널 내에서 rbac-user라는 새 사용자를 만들고 이에 대한 자격 증명을 생성/저장합니다.

```
aws iam create-user --user-name rbac-user
aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json
```

이전 단계를 실행하면 다음과 유사한 응답을 받아야 합니다.

```
{
    "AccessKey": {
        "UserName": "rbac-user",
        "Status": "Active",
        "CreateDate": "2019-07-17T15:37:27Z",
        "SecretAccessKey": < AWS Secret Access Key > ,
        "AccessKeyId": < AWS Access Key >
    }
}
```

클러스터를 생성한 admin 사용자와 이 새로운 rbac-user 간에 쉽게 전환할 수 있도록 하려면 다음 명령을 실행하여 소스가 생성될 때 활성 사용자를 rbac-user로 설정하는 스크립트를 생성합니다.

```
cat << EoF > rbacuser_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
EoF
```

## IAM 사용자를 K8에 매핑
다음으로 rbac-user라는 k8s 사용자를 정의하고 해당 IAM 사용자에 매핑합니다. 다음을 실행하여 기존 ConfigMap을 가져오고 aws-auth.yaml이라는 파일에 저장합니다.

```
kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml
```

다음으로 기존 configMap에 rbac-user 매핑을 추가합니다.

```
cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/rbac-user
      username: rbac-user
EoF
```

일부 값은 파일이 생성될 때 동적으로 채워질 수 있습니다. 모든 것이 채워지고 올바르게 생성되었는지 확인하려면 다음을 실행하십시오.

```
cat aws-auth.yaml
```

그리고 출력은 다음과 유사하게 채워진 rolearn 및 userarn을 반영해야 합니다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/rbac-user
      username: rbac-user
```

다음으로 ConfigMap을 적용하여 이 매핑을 시스템에 적용합니다.

```
kubectl apply -f aws-auth.yaml
```

## 새 사용자 테스트
지금까지 클러스터 운영자로서 관리자로 클러스터에 액세스했습니다. 이제 새로 생성된 rbac-user로 클러스터에 액세스할 때 어떤 일이 발생하는지 살펴보겠습니다.

다음 명령을 실행하여 rbac-user의 AWS IAM 사용자 환경 변수를 소싱합니다.

```
. rbacuser_creds.sh
```

위의 명령을 실행하여 이제 기본 관리자 사용자 또는 역할을 재정의해야 하는 AWS 환경 변수를 설정했습니다. 기본 사용자 설정을 재정의했는지 확인하려면 다음 명령을 실행합니다.

```
aws sts get-caller-identity
```

이제 rbac-user로 API를 호출하는 아래와 유사한 내용이 표시되어야 합니다.

```
{
    "계정": <AWS 계정 ID>,
    "UserId": <AWS 사용자 ID>,
    "Arn": "arn:aws:iam::<AWS 계정 ID>:user/rbac-user"
}
```

이제 rbac-user 컨텍스트에서 호출을 하고 있으므로 모든 포드를 가져오도록 빠르게 요청할 수 있습니다.

```
kubectl get pods -n rbac-test
```

다음과 유사한 응답을 받아야 합니다.

```
No resources found.  Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "rbac-test"
```

우리는 이미 rbac-user를 생성했는데 왜 그 오류가 발생했습니까?

사용자를 생성한다고 해서 해당 사용자에게 클러스터의 리소스에 대한 액세스 권한이 부여되는 것은 아닙니다. 이를 달성하려면 역할을 정의한 다음 사용자를 해당 역할에 바인딩해야 합니다. 우리는 다음에 그것을 할 것입니다.

## 역할 및 바인딩 만들기
앞서 언급했듯이 새로운 사용자 rbac-user가 있지만 아직 어떤 역할에도 바인딩되지 않았습니다. 그렇게 하려면 기본 관리자 사용자로 다시 전환해야 합니다.

다음을 실행하여 우리를 rbac-user로 정의하는 환경 변수를 설정 해제합니다.

```
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
```

우리가 admin 사용자이고 더 이상 rbac-user가 아님을 확인하려면 다음 명령을 실행하십시오.

```
aws sts get-caller-identity
```

출력은 사용자가 더 이상 rbac-user가 아님을 보여야 합니다.

```
{
    "Account": <AWS Account ID>,
    "UserId": <AWS User ID>,
    "Arn": "arn:aws:iam::<your AWS account ID>:assumed-role/eksworkshop-admin/i-123456789"
}
```

이제 다시 관리자 사용자가 되었으므로 pod 및 배포에 대한 목록, 가져오기 및 감시 액세스 권한을 제공하지만 rbac-test 네임스페이스에만 해당하는 pod-reader 역할을 생성합니다. 다음을 실행하여 이 역할을 만듭니다.

```
cat << EoF > rbacuser-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["list","get","watch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EoF
```

우리에게는 사용자가 있고 역할이 있으며 이제 이들을 RoleBinding 리소스와 함께 바인딩합니다. 다음을 실행하여 이 RoleBinding을 만듭니다.

```
cat << EoF > rbacuser-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: User
  name: rbac-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EoF
```

다음으로 생성한 Role 및 RoleBindings를 적용합니다.

```
kubectl apply -f rbacuser-role.yaml
kubectl apply -f rbacuser-role-binding.yaml
```

## 역할 및 바인딩 확인
이제 사용자, Role 및 RoleBinding이 정의되었으므로 rbac-user로 다시 전환하여 테스트해 보겠습니다.

rbac-user로 다시 전환하려면 rbac-user 환경 변수를 소싱하고 사용했는지 확인하는 다음 명령을 실행합니다.

```
. rbacuser_creds.sh; aws sts get-caller-identity
```

rbac-user로 로그인했음을 반영하는 출력이 표시되어야 합니다.

rbac-user로 다음을 실행하여 rbac 네임스페이스에서 팟(Pod)을 가져오십시오.

```
kubectl get pods -n rbac-test
```

출력은 다음과 유사해야 합니다.

```
NAME                    READY     STATUS    RESTARTS   AGE
nginx-55bd7c9fd-kmbkf   1/1       Running   0          23h
```

rbac-test 네임스페이스 외부에서 동일한 명령을 다시 실행해 보십시오.

```
kubectl get pods -n kube-system
```

다음과 유사한 오류가 표시되어야 합니다.

```
No resources found.
Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

바인딩된 역할은 rbac-test 이외의 네임스페이스에 대한 액세스를 제공하지 않기 때문입니다.

## CLEAN UP
이 장을 완료하면 다음 명령을 실행하여 생성한 파일과 리소스를 정리할 수 있습니다.

```
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
kubectl delete namespace rbac-test
rm rbacuser_creds.sh
rm rbacuser-role.yaml
rm rbacuser-role-binding.yaml
aws iam delete-access-key --user-name=rbac-user --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
aws iam delete-user --user-name rbac-user
rm /tmp/create_output.json
```

다음으로 기존 aws-auth.yaml 파일을 편집하여 기존 configMap에서 rbac-user 매핑을 제거합니다.

```
data:
  mapUsers: |
    []
```

그리고 ConfigMap을 적용하고 aws-auth.yaml 파일을 삭제합니다.

```
kubectl apply -f aws-auth.yaml
rm aws-auth.yaml
```
