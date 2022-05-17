# 포드용 보안 그룹

## 소개

컨테이너화된 애플리케이션은 클러스터 내에서 실행되는 다른 서비스와 다음과 같은 외부 AWS 서비스에 대한 액세스가 필요한 경우가 많습니다. [Amazon 관계형 데이터베이스 서비스](https://www.google.com/url?sa=t\&rct=j\&q=\&esrc=s\&source=web\&cd=\&cad=rja\&uact=8\&ved=2ahUKEwiYkYfF9bHtAhWEwFkKHT6nD7kQFjAAegQIARAD\&url=https%3A%2F%2Faws.amazon.com%2Frds%2F\&usg=AOvVaw1EJQFNeMAoVICsb0iec7IR) (아마존 RDS).

AWS에서 서비스 간의 네트워크 수준 액세스 제어는 종종 다음을 통해 수행됩니다. [보안 그룹](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html).

이 새로운 기능이 출시되기 전에는 노드 수준에서만 보안 그룹을 할당할 수 있었습니다. 그리고 노드 그룹 내의 모든 노드가 보안 그룹을 공유하기 때문에 노드 그룹 보안 그룹이 RDS 인스턴스에 액세스할 수 있도록 함으로써 녹색 포드만 액세스해야 하는 경우에도 이 노드에서 실행되는 모든 포드가 데이터베이스에 액세스할 수 있습니다.

![](../Beginner/images/sg-per-pod\_1.png)

[포드용 보안 그룹](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html) Amazon EC2 보안 그룹을 Kubernetes 포드와 통합합니다. Amazon EC2 보안 그룹을 사용하여 여러 Amazon EC2 인스턴스 유형에서 실행되는 노드에 배포하는 포드에서 들어오고 나가는 인바운드 및 아웃바운드 네트워크 트래픽을 허용하는 규칙을 정의할 수 있습니다. 이 기능에 대한 자세한 설명은 다음을 참조하십시오. [포드용 보안 그룹 소개 블로그 게시물](https://aws.amazon.com/blogs/containers/introducing-security-groups-for-pods/) 그리고 [공식 문서](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html).

## 목표

워크숍의 이 섹션에서:

* RDS\_SG라는 보안 그룹으로 보호되는 Amazon RDS 데이터베이스를 생성합니다.
* RDS 인스턴스에 연결할 수 있는 POD\_SG라는 보안 그룹을 생성합니다.
* 그런 다음 SecurityGroupPolicyPOD\_SG 보안 그룹을 올바른 메타데이터가 있는 포드에 자동으로 연결하는 배포를 배포합니다 .
* 마지막으로 동일한 이미지를 사용하여 두 개의 포드(녹색 및 빨간색)를 배포하고 그 중 하나만(녹색) Amazon RDS 데이터베이스에 연결할 수 있는지 확인합니다.

![](../Beginner/images/sg-per-pod\_3.png)

## 전제 조건

포드의 보안 그룹은 포함한 대부분의 니트로 기반 아마존 EC2 인스턴스 가족, 지원하는 m5, c5, r5, p3, m6g, c6g, 및 r6g인스턴스 가족. 인스턴스 가족은 지원되지 않습니다 그래서 우리는 하나 개를 사용하여 두 번째 노드 그룹이 생성됩니다 인스턴스를.t3m5.large

```
mkdir ${HOME}/environment/sg-per-pod

cat << EoF > ${HOME}/environment/sg-per-pod/nodegroup-sec-group.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}

managedNodeGroups:
- name: nodegroup-sec-group
  desiredCapacity: 1
  instanceType: m5.large
EoF

eksctl create nodegroup -f ${HOME}/environment/sg-per-pod/nodegroup-sec-group.yaml

 kubectl get nodes \
  --selector beta.kubernetes.io/instance-type=m5.large
```

```
NAME                                           STATUS   ROLES    AGE     VERSION
ip-192-168-34-45.us-east-2.compute.internal    Ready    <none>   4m57s   v1.17.12-eks-7684af
```

## 보안 그룹 생성

### 보안 그룹 생성 및 구성

먼저 RDS 보안 그룹(RDS\_SG)을 생성해 보겠습니다. Amazon RDS 인스턴스에서 네트워크 액세스를 제어하는 ​​데 사용됩니다.

```
export VPC_ID=$(aws eks describe-cluster \
    --name eksworkshop-eksctl \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)

# create RDS security group
aws ec2 create-security-group \
    --description 'RDS SG' \
    --group-name 'RDS_SG' \
    --vpc-id ${VPC_ID}

# save the security group ID for future use
export RDS_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=RDS_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)

echo "RDS security group ID: ${RDS_SG}"
```

이제 Pod 보안 그룹(POD\_SG)을 생성해 보겠습니다.

```
# create the POD security group
aws ec2 create-security-group \
    --description 'POD SG' \
    --group-name 'POD_SG' \
    --vpc-id ${VPC_ID}

# save the security group ID for future use
export POD_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=POD_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)

echo "POD security group ID: ${POD_SG}"
```

포드는 DNS 확인을 위해 노드와 통신해야 하므로 그에 따라 노드 그룹 보안 그룹을 업데이트합니다.

```
export NODE_GROUP_SG=$(aws ec2 describe-security-groups \
    --filters Name=tag:Name,Values=eks-cluster-sg-eksworkshop-eksctl-* Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" \
    --output text)
echo "Node Group security group ID: ${NODE_GROUP_SG}"

# allow POD_SG to connect to NODE_GROUP_SG using TCP 53
aws ec2 authorize-security-group-ingress \
    --group-id ${NODE_GROUP_SG} \
    --protocol tcp \
    --port 53 \
    --source-group ${POD_SG}

# allow POD_SG to connect to NODE_GROUP_SG using UDP 53
aws ec2 authorize-security-group-ingress \
    --group-id ${NODE_GROUP_SG} \
    --protocol udp \
    --port 53 \
    --source-group ${POD_SG}
```

마지막으로 두 개의 인바운드 트래픽(인그레스)을 추가합니다. 규칙 RDS\_SG 보안 그룹:

* 하나는 Cloud9용(데이터베이스 채우기용)입니다.
* 그리고 두 번째는 POD\_SG 보안 그룹이 데이터베이스에 연결할 수 있도록 허용합니다.

```
# Cloud9 IP
export C9_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)

# allow Cloud9 to connect to RDS
aws ec2 authorize-security-group-ingress \
    --group-id ${RDS_SG} \
    --protocol tcp \
    --port 5432 \
    --cidr ${C9_IP}/32

# Allow POD_SG to connect to the RDS
aws ec2 authorize-security-group-ingress \
    --group-id ${RDS_SG} \
    --protocol tcp \
    --port 5432 \
    --source-group ${POD_SG}
```

## RDS 생성

이제 보안 그룹이 준비되었으므로 PostgreSQL용 Amazon RDS 데이터베이스를 생성해 보겠습니다.

먼저 생성해야 합니다. DB 서브넷 그룹. EKS 클러스터와 동일한 서브넷을 사용합니다.

```
export PUBLIC_SUBNETS_ID=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=eksctl-eksworkshop-eksctl-cluster/SubnetPublic*" \
    --query 'Subnets[*].SubnetId' \
    --output json | jq -c .)

# create a db subnet group
aws rds create-db-subnet-group \
    --db-subnet-group-name rds-eksworkshop \
    --db-subnet-group-description rds-eksworkshop \
    --subnet-ids ${PUBLIC_SUBNETS_ID}
```

이제 데이터베이스를 생성할 수 있습니다.

```
# get RDS SG ID
export RDS_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=RDS_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)

# generate a password for RDS
export RDS_PASSWORD="$(date | md5sum  |cut -f1 -d' ')"
echo ${RDS_PASSWORD}  > ~/environment/sg-per-pod/rds_password


# create RDS Postgresql instance
aws rds create-db-instance \
    --db-instance-identifier rds-eksworkshop \
    --db-name eksworkshop \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --db-subnet-group-name rds-eksworkshop \
    --vpc-security-group-ids $RDS_SG \
    --master-username eksworkshop \
    --publicly-accessible \
    --master-user-password ${RDS_PASSWORD} \
    --backup-retention-period 0 \
    --allocated-storage 20
```

데이터베이스를 생성하는 데 최대 4분이 소요됩니다.

이 명령을 사용하여 사용 가능한지 확인할 수 있습니다.

```
aws rds describe-db-instances \
    --db-instance-identifier rds-eksworkshop \
    --query "DBInstances[].DBInstanceStatus" \
    --output text
```

예상 출력

```
available
```

이제 데이터베이스를 사용할 수 있으므로 데이터베이스 엔드포인트를 가져오겠습니다.

```
# get RDS endpoint
export RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier rds-eksworkshop \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

echo "RDS endpoint: ${RDS_ENDPOINT}"
```

마지막 단계는 데이터베이스에 일부 콘텐츠를 만드는 것입니다.

```
sudo yum install -y postgresql

cd sg-per-pod

cat << EoF > ~/environment/sg-per-pod/pgsql.sql
CREATE TABLE welcome (column1 TEXT);
insert into welcome values ('--------------------------');
insert into welcome values ('Welcome to the eksworkshop');
insert into welcome values ('--------------------------');
EoF

export RDS_PASSWORD=$(cat ~/environment/sg-per-pod/rds_password)

psql postgresql://eksworkshop:${RDS_PASSWORD}@${RDS_ENDPOINT}:5432/eksworkshop \
    -f ~/environment/sg-per-pod/pgsql.sql
```

## CNI 구성

이 새로운 기능을 활성화하기 위해 Amazon EKS 클러스터에는 Kubernetes 제어 플레인에서 실행되는 두 가지 새로운 구성 요소가 있습니다.

* 보안 그룹이 필요한 포드에 제한 및 요청을 추가하는 작업을 담당 하는 변형 웹훅 입니다.
* 관리를 담당 하는 리소스 컨트롤러네트워크 인터페이스 해당 포드와 연결됩니다.

이 기능을 용이하게 하기 위해 각 작업자 노드는 단일 트렁크 네트워크 인터페이스 및 여러 분기 네트워크 인터페이스와 연결됩니다. 트렁크 인터페이스는 인스턴스에 연결된 표준 네트워크 인터페이스 역할을 합니다. 그런 다음 VPC 리소스 컨트롤러는 분기 인터페이스를 트렁크 인터페이스에 연결합니다. 이렇게 하면 인스턴스당 연결할 수 있는 네트워크 인터페이스 수가 늘어납니다. 보안 그룹은 네트워크 인터페이스로 지정되므로 이제 작업자 노드에 할당된 이러한 추가 네트워크 인터페이스에 특정 보안 그룹이 필요한 팟(Pod)을 예약할 수 있습니다.

먼저 EC2 인스턴스가 네트워크 인터페이스, 프라이빗 IP 주소, 인스턴스에 대한 연결 및 분리를 관리할 수 있도록 새 IAM 정책을 노드 그룹 역할에 연결해야 합니다.

다음 명령은 정책 AmazonEKSVPCResourceController을 클러스터 역할에 추가합니다 .

```
aws iam attach-role-policy \
    --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController \
    --role-name ${ROLE_NAME}
```

다음으로 ENABLE\_POD\_ENIaws-node 에서 변수를 true 로 설정하여 CNI 플러그인이 포드의 네트워크 인터페이스를 관리할 수 있도록 합니다 DaemonSet.

```
kubectl -n kube-system set env daemonset aws-node ENABLE_POD_ENI=true

# let's way for the rolling update of the daemonset
kubectl -n kube-system rollout status ds aws-node
```

이 설정이 true로 설정되면 플러그인은 클러스터의 각 노드에 대해 값 vpc.amazonaws.com/has-trunk-attached=true이 포함 된 레이블을 호환 가능한 인스턴스에 추가합니다. VPC 리소스 컨트롤러는 설명이 aws-k8s-trunk-eni인 트렁크 네트워크 인터페이스라는 하나의 특수 네트워크 인터페이스를 생성하고 연결합니다.

```
 kubectl get nodes \
  --selector alpha.eksctl.io/nodegroup-name=nodegroup-sec-group \
  --show-labels
```

![](../Beginner/images/sg-per-pod\_4.png)

## 보안 그룹 정책

### 보안 그룹 정책

새 사용자 지정 리소스 정의(CRD)도 클러스터 생성 시 자동으로 추가되었습니다. 클러스터 관리자는 SecurityGroupPolicyCRD를 통해 포드에 할당할 보안 그룹을 지정할 수 있습니다. 네임스페이스 내에서 포드 레이블을 기반으로 하거나 포드와 연결된 서비스 계정의 레이블을 기반으로 포드를 선택할 수 있습니다. 일치하는 모든 포드에 대해 적용할 보안 그룹 ID도 정의합니다.

이 명령으로 CRD가 있는지 확인할 수 있습니다.

```
kubectl get crd securitygrouppolicies.vpcresources.k8s.aws
```

산출

```
securitygrouppolicies.vpcresources.k8s.aws 2020-11-04T17:01:27Z
```

웹훅 SecurityGroupPolicy은 변경 사항에 대해 사용자 지정 리소스를 감시 하고 사용 가능한 분기 네트워크 인터페이스 용량이 있는 노드에 포드를 예약하는 데 필요한 확장 리소스 요청과 함께 일치하는 포드를 자동으로 주입합니다. 포드가 예약되면 리소스 컨트롤러가 분기 인터페이스를 생성하고 트렁크 인터페이스에 연결합니다. 연결에 성공하면 컨트롤러는 분기 인터페이스 세부 정보와 함께 포드 개체에 주석을 추가합니다.

이제 정책을 만들어 보겠습니다.

```
cat << EoF > ~/environment/sg-per-pod/sg-policy.yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: allow-rds-access
spec:
  podSelector:
    matchLabels:
      app: green-pod
  securityGroups:
    groupIds:
      - ${POD_SG}
EoF
```

보시다시피 포드에 레이블 app: green-pod이 있는 경우 보안 그룹이 연결됩니다.

마침내 특정 namespace.

```
kubectl create namespace sg-per-pod

kubectl -n sg-per-pod apply -f ~/environment/sg-per-pod/sg-policy.yaml

kubectl -n sg-per-pod describe securitygrouppolicy
```

산출

```
Name:         allow-rds-access
Namespace:    sg-per-pod
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"vpcresources.k8s.aws/v1beta1","kind":"SecurityGroupPolicy","metadata":{"annotations":{},"name":"allow-rds-access","namespac...
API Version:  vpcresources.k8s.aws/v1beta1
Kind:         SecurityGroupPolicy
Metadata:
  Creation Timestamp:  2020-12-03T04:35:57Z
  Generation:          1
  Resource Version:    9142629
  Self Link:           /apis/vpcresources.k8s.aws/v1beta1/namespaces/sg-per-pod/securitygrouppolicies/allow-rds-access
  UID:                 bf1e329d-816e-4ab0-abe8-934cadabfdd3
Spec:
  Pod Selector:
    Match Labels:
      App:  green-pod
  Security Groups:
    Group Ids:
      sg-0ff967bc903e9639e
Events:  <none>
```

## 포드 배포

### 쿠버네티스 비밀

두 개의 포드를 배포하기 전에 RDS 엔드포인트와 암호를 제공해야 합니다. 우리는 kubernetes 비밀을 만들 것입니다.

```
export RDS_PASSWORD=$(cat ~/environment/sg-per-pod/rds_password)

export RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier rds-eksworkshop \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

kubectl create secret generic rds\
    --namespace=sg-per-pod \
    --from-literal="password=${RDS_PASSWORD}" \
    --from-literal="host=${RDS_ENDPOINT}"

kubectl -n sg-per-pod describe  secret rds
```

산출

```
Name:         rds
Namespace:    sg-per-pod
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
host:      56 bytes
password:  32 bytes
```

### 배포

두 포드 배포 파일을 모두 다운로드합시다.

```
cd ~/environment/sg-per-pod

curl -s -O https://www.eksworkshop.com/beginner/115_sg-per-pod/deployments.files/green-pod.yaml
curl -s -O https://www.eksworkshop.com/beginner/115_sg-per-pod/deployments.files/red-pod.yaml
```

시간을 내어 두 YAML 파일을 모두 살펴보고 두 파일의 차이점을 확인하세요.

![](../Beginner/images/sg-per-pod\_5.png)

### 그린포드

이제 녹색 포드를 배포해 보겠습니다.

```
kubectl -n sg-per-pod apply -f ~/environment/sg-per-pod/green-pod.yaml

kubectl -n sg-per-pod rollout status deployment green-pod
```

컨테이너는 다음을 시도합니다.

* 데이터베이스에 연결하고 테이블의 내용을 STDOUT 으로 출력 합니다.
* 데이터베이스 연결이 실패하면 오류 메시지도 STDOUT 으로 출력됩니다 .

로그를 확인해보자.

```
export GREEN_POD_NAME=$(kubectl -n sg-per-pod get pods -l app=green-pod -o jsonpath='{.items[].metadata.name}')

kubectl -n sg-per-pod  logs -f ${GREEN_POD_NAME}
```

산출

```
CTRL+C 를 사용 하여 로그 종료
```

보시다시피, 우리의 시도는 성공적이었습니다!

이제 다음을 확인하겠습니다.

* ENI가 포드에 연결됩니다.
* 그리고 ENI에는 보안 그룹 POD\_SG가 연결되어 있습니다. Annotations이 명령을 사용하여 pod 섹션 에서 ENI ID를 찾을 수 있습니다 .

```
kubectl -n sg-per-pod  describe pod $GREEN_POD_NAME | head -11
```

산출

```
Name:         green-pod-5c786d8dff-4kmvc
Namespace:    sg-per-pod
Priority:     0
Node:         ip-192-168-33-222.us-east-2.compute.internal/192.168.33.222
Start Time:   Thu, 03 Dec 2020 05:25:54 +0000
Labels:       app=green-pod
              pod-template-hash=5c786d8dff
Annotations:  kubernetes.io/psp: eks.privileged
              vpc.amazonaws.com/pod-eni:
                [{"eniId":"eni-0d8a3a3a7f2eb57ab","ifAddress":"06:20:0d:3c:5f:bc","privateIp":"192.168.47.64","vlanId":1,"subnetCidr":"192.168.32.0/19"}]
Status:       Running
```

이것을 열어서 보안 그룹 POD\_SG가 eni위에 표시된 것에 연결되어 있는지 확인할 수 있습니다.링크.

![](../Beginner/images/sg-per-pod\_6.png)

레드포드 빨간색 포드를 배포하고 데이터베이스에 연결할 수 없는지 확인합니다.

녹색 포드와 마찬가지로 컨테이너는 다음을 시도합니다.

* 데이터베이스에 연결 하고 테이블의 내용 을 STDOUT으로 출력 합니다.
* 데이터베이스 연결이 실패하면 오류 메시지도 STDOUT 으로 출력됩니다 .

```
kubectl -n sg-per-pod apply -f ~/environment/sg-per-pod/red-pod.yaml

kubectl -n sg-per-pod rollout status deployment red-pod
```

로그를 확인하겠습니다( CTRL+C 를 사용 하여 로그 종료).

```
export RED_POD_NAME=$(kubectl -n sg-per-pod get pods -l app=red-pod -o jsonpath='{.items[].metadata.name}')

kubectl -n sg-per-pod  logs -f ${RED_POD_NAME}
```

산출

```
Database connection failed due to timeout expired
```

마지막으로 포드에 enitId 가 없는지 확인 하겠습니다annotation .

```
kubectl -n sg-per-pod  describe pod ${RED_POD_NAME} | head -11
```

산출

```
Name:         red-pod-7f68d78475-vlm77
Namespace:    sg-per-pod
Priority:     0
Node:         ip-192-168-6-158.us-east-2.compute.internal/192.168.6.158
Start Time:   Thu, 03 Dec 2020 07:08:28 +0000
Labels:       app=red-pod
              pod-template-hash=7f68d78475
Annotations:  kubernetes.io/psp: eks.privileged
Status:       Running
IP:           192.168.0.188
IPs:
```

### 결론

이 모듈에서는 포드당 보안 그룹 기능을 활성화하도록 EKS 클러스터를 구성했습니다.

SecurityGroup정책을 만들고 보안 그룹으로 보호되는 2개의 포드(동일한 도커 이미지 사용)와 RDS 데이터베이스를 배포했습니다.

이 정책에 따라 두 포드 중 하나만 데이터베이스에 연결할 수 있었습니다.

마지막으로 CLI와 AWS 콘솔을 사용하여 포드의 ENI를 찾고 보안 그룹이 연결되었는지 확인할 수 있었습니다.

## CLEAN UP

```
export VPC_ID=$(aws eks describe-cluster \
    --name eksworkshop-eksctl \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
export RDS_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=RDS_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)
export POD_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=POD_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)
export C9_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
export NODE_GROUP_SG=$(aws ec2 describe-security-groups \
    --filters Name=tag:Name,Values=eks-cluster-sg-eksworkshop-eksctl-* Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" \
    --output text)

# uninstall the RPM package
sudo yum erase -y postgresql

# delete database
aws rds delete-db-instance \
    --db-instance-identifier rds-eksworkshop \
    --delete-automated-backups \
    --skip-final-snapshot

# delete kubernetes element
kubectl -n sg-per-pod delete -f ~/environment/sg-per-pod/green-pod.yaml
kubectl -n sg-per-pod delete -f ~/environment/sg-per-pod/red-pod.yaml
kubectl -n sg-per-pod delete -f ~/environment/sg-per-pod/sg-policy.yaml
kubectl -n sg-per-pod delete secret rds

# delete the namespace
kubectl delete ns sg-per-pod

# disable ENI trunking
kubectl -n kube-system set env daemonset aws-node ENABLE_POD_ENI=false
kubectl -n kube-system rollout status ds aws-node

# detach the IAM policy
aws iam detach-role-policy \
    --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController \
    --role-name ${ROLE_NAME}

# remove the security groups rules
aws ec2 revoke-security-group-ingress \
    --group-id ${RDS_SG} \
    --protocol tcp \
    --port 5432 \
    --source-group ${POD_SG}

aws ec2 revoke-security-group-ingress \
    --group-id ${RDS_SG} \
    --protocol tcp \
    --port 5432 \
    --cidr ${C9_IP}/32

aws ec2 revoke-security-group-ingress \
    --group-id ${NODE_GROUP_SG} \
    --protocol tcp \
    --port 53 \
    --source-group ${POD_SG}

aws ec2 revoke-security-group-ingress \
    --group-id ${NODE_GROUP_SG} \
    --protocol udp \
    --port 53 \
    --source-group ${POD_SG}

# delete POD security group
aws ec2 delete-security-group \
    --group-id ${POD_SG}
```

RDS 인스턴스가 삭제되었는지 확인합니다.

```
aws rds describe-db-instances \
    --db-instance-identifier rds-eksworkshop \
    --query "DBInstances[].DBInstanceStatus" \
    --output text
```

예상 출력

```
An error occurred (DBInstanceNotFound) when calling the DescribeDBInstances operation: DBInstance rds-eksworkshop not found.
```

이제 DB 보안 그룹과 DB 서브넷 그룹을 안전하게 삭제할 수 있습니다.

```
# delete RDS SG
aws ec2 delete-security-group \
    --group-id ${RDS_SG}

# delete DB subnet group
aws rds delete-db-subnet-group \
    --db-subnet-group-name rds-eksworkshop
```

마지막으로 EKS 노드 그룹을 삭제합니다.

```
# delete the nodegroup
eksctl delete nodegroup -f ${HOME}/environment/sg-per-pod/nodegroup-sec-group.yaml --approve

# remove the trunk label
kubectl label node  --all 'vpc.amazonaws.com/has-trunk-attached'-

cd ~/environment
rm -rf sg-per-pod
```
