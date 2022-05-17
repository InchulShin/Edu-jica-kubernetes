# 다음을 사용하여 시작 [EKSCTL](https://eksctl.io/)

[eksctl](https://eksctl.io/) AWS와 공동으로 개발한 도구입니다. 위브워크 EKS 클러스터 생성 경험의 대부분을 자동화합니다.

이 모듈에서는 eksctl을 사용하여 EKS 클러스터 및 노드를 시작하고 구성합니다.

## 전제 조건
이 모듈의 경우 다운로드해야 합니다. eksctl 바이너리:

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

eksctl 명령이 작동하는지 확인합니다.

```
eksctl version
```

eksctl bash-completion 활성화

```
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

## EKS 실행
다음이 없으면 이 단계를 진행하지 마십시오.IAM 역할 검증Cloud9 IDE에서 사용 중입니다. IAM 역할을 사용하여 EKS 클러스터를 구축하지 않으면 이후 모듈에서 필요한 kubectl 명령을 실행할 수 없습니다.

도전:
작업 공간에서 IAM 역할을 확인하려면 어떻게 해야 합니까?

 솔루션을 보려면 여기를 확장하십시오.

운영 aws sts get-caller-identity귀하의 Arn 이eksworkshop-admin및 인스턴스 ID.

```
{
    "Account": "123456789012",
    "UserId": "AROA1SAMPLEAWSIAMROLE:i-01234567890abcdef",
    "Arn": "arn:aws:sts::123456789012:assumed-role/eksworkshop-admin/i-01234567890abcdef"
}
```

올바른 역할이 표시되지 않으면 뒤로 돌아가서 IAM 역할 검증 문제 해결을 위해.

올바른 역할이 표시되면 다음 단계로 진행하여 EKS 클러스터를 생성합니다.

EKS 클러스터 생성
eksctl EKS 1.19를 배포하려면 버전이 0.38.0 이상이어야 합니다. 여기를 클릭 최신 버전을 얻으려면.

다음 구문을 사용하여 클러스터를 생성하는 데 사용할 eksctl 배포 파일(eksworkshop.yaml)을 생성합니다.

```
cat << EOF > eksworkshop.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}
  version: "1.19"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.small
  ssh:
    enableSsm: true

# To enable all of the control plane logs, uncomment below:
# cloudWatch:
#  clusterLogging:
#    enableTypes: ["*"]

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```

그런 다음 생성한 파일을 eksctl 클러스터 생성을 위한 입력으로 사용합니다.

우리는 에서 사용 가능한 최신 버전보다 최소 하나 이상의 Kubernetes 버전을 의도적으로 출시하고 있습니다. 아마존 EKS. 이를 통해 다음을 수행할 수 있습니다.클러스터 업그레이드 랩.

```
eksctl create cluster -f eksworkshop.yaml
```

EKS 및 모든 종속성을 시작하는 데 약 15분이 소요됩니다.

## 클러스터 테스트
클러스터를 테스트합니다.

노드 확인:

```
kubectl get nodes # if we see our 3 nodes, we know we have authenticated correctly
```

워크샵 전체에서 사용할 작업자 역할 이름 내보내기:

```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

축하합니다!
이제 사용할 준비가 된 완전히 작동하는 Amazon EKS 클러스터가 생겼습니다! 다른 실습으로 이동하기 전에 다음 페이지의 단계를 완료하여 EKS 콘솔 자격 증명을 업데이트해야 합니다.

## 콘솔 자격 증명
거의 모든 워크샵 콘텐츠가 CLI 기반이므로 이 단계는 선택 사항입니다. 그러나 EKS 콘솔에서 워크샵 클러스터에 대한 전체 액세스 권한을 원하는 경우 이 단계를 권장합니다.

EKS 콘솔을 사용하면 클러스터의 구성 측면뿐만 아니라 배포, 포드 및 노드와 같은 Kubernetes 클러스터 개체도 볼 수 있습니다. 이러한 유형의 액세스를 위해서는 콘솔 IAM 사용자 또는 역할에 클러스터 내에서 권한이 부여되어야 합니다.

기본적으로 클러스터를 만드는 데 사용된 자격 증명에는 이러한 권한이 자동으로 부여됩니다. 워크숍을 따라 Cloud9 내에서 임시 IAM 자격 증명을 사용하여 클러스터를 생성했습니다. 즉, 클러스터에 AWS 콘솔 자격 증명을 추가해야 합니다.

EKS 콘솔 자격 증명을 새 클러스터로 가져옵니다.
IAM 사용자 및 역할은 이라는 ConfigMap을 통해 EKS Kubernetes 클러스터에 바인딩됩니다 aws-auth. eksctl하나의 명령으로 이를 수행 할 수 있습니다 .

AWS 콘솔 액세스를 위해 추가할 올바른 자격 증명을 결정해야 합니다. 이미 알고 있는 경우 eksctl create iamidentitymapping아래 단계로 건너뛸 수 있습니다.

이 자습서의 일부로 Cloud9에서 클러스터를 구축한 경우 환경 내에서 다음을 호출하여 IAM 역할 또는 사용자 ARN을 결정합니다.

```
c9builder=$(aws cloud9 describe-environment-memberships --environment-id=$C9_PID | jq -r '.memberships[].userArn')
if echo ${c9builder} | grep -q user; then
	rolearn=${c9builder}
        echo Role ARN: ${rolearn}
elif echo ${c9builder} | grep -q assumed-role; then
        assumedrolename=$(echo ${c9builder} | awk -F/ '{print $(NF-1)}')
        rolearn=$(aws iam get-role --role-name ${assumedrolename} --query Role.Arn --output text) 
        echo Role ARN: ${rolearn}
fi
```

ARN이 있으면 명령을 실행하여 클러스터 내에서 ID 매핑을 생성할 수 있습니다.

```
eksctl create iamidentitymapping --cluster eksworkshop-eksctl --arn ${rolearn} --group system:masters --username admin
```

권한을 제한하고 세분화할 수 있지만 워크샵 클러스터이므로 콘솔 자격 증명을 관리자로 추가합니다.

이제 콘솔 내 AWS 인증 맵에서 항목을 확인할 수 있습니다.

```
kubectl describe configmap -n kube-system aws-auth
```

이제 계속 진행할 준비가 되었습니다. 자세한 내용은EKS 문서 이 주제에.


