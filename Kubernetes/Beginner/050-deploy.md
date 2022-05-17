## 예제 마이크로서비스 배포
### 샘플 애플리케이션 배포

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecsdemo-nodejs
  labels:
    app: ecsdemo-nodejs
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ecsdemo-nodejs
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ecsdemo-nodejs
    spec:
      containers:
      - image: brentley/ecsdemo-nodejs:latest
        imagePullPolicy: Always
        name: ecsdemo-nodejs
        ports:
        - containerPort: 3000
          protocol: TCP
```

위의 샘플 파일에서 서비스와 배포 방법에 대해 설명합니다 . 이 설명을 kubectl을 사용하여 kubernetes API에 작성하고 kubernetes는 애플리케이션이 배포될 때 기본 설정이 충족되는지 확인합니다.

컨테이너는 포트 3000에서 수신 대기하고 기본 서비스 검색을 사용하여 실행 중인 컨테이너를 찾고 통신합니다.

### NODEJS 백엔드 API 배포
NodeJS 백엔드 API를 불러오자!

다음 명령을 복사하여 Cloud9 작업 공간에 붙여넣습니다.

```
cd ~/environment/ecsdemo-nodejs
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

배포 상태를 보면 진행 상황을 볼 수 있습니다.

```
kubectl get deployment ecsdemo-nodejs
```

### CRYSTAL 백엔드 API 배포
Crystal Backend API를 불러오겠습니다!

다음 명령을 복사하여 Cloud9 작업 공간에 붙여넣습니다.

```
cd ~/environment/ecsdemo-crystal
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

배포 상태를 보면 진행 상황을 볼 수 있습니다.

```
kubectl get deployment ecsdemo-crystal
```

### 서비스 유형을 확인하자
프론트엔드 서비스를 시작하기 전에 사용 중인 서비스 유형을 살펴보겠습니다. 이것은 kubernetes/service.yaml프론트엔드 서비스를 위한 것입니다.

```
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
spec:
  selector:
    app: ecsdemo-frontend
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```

알림 type: LoadBalancer: 이 서비스로 들어오는 트래픽을 처리하도록 ELB를 구성합니다.

이를 kubernetes/service.yaml백엔드 서비스 중 하나와 비교하십시오 .

```
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-nodejs
spec:
  selector:
    app: ecsdemo-nodejs
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```

설명된 특정 서비스 유형이 없습니다. 우리가 확인할 때쿠버네티스 문서 기본 유형은 ClusterIP입니다. 클러스터 내부 IP에서 서비스를 노출합니다. 이 값을 선택하면 클러스터 내에서만 서비스에 연결할 수 있습니다.

### ELB 서비스 역할이 존재하는지 확인
이전에 로드 밸런서를 생성한 적이 없는 AWS 계정에서는 ELB에 대한 서비스 역할이 아직 존재하지 않을 수 있습니다.

역할을 확인하고 누락된 경우 생성할 수 있습니다.

다음 명령을 복사하여 Cloud9 작업 공간에 붙여넣습니다.

```
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" || aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```

### 프런트엔드 서비스 배포
도전:
Ruby Frontend를 불러오자!

솔루션을 보려면 여기를 확장하십시오.

다음 명령을 복사하여 Cloud9 작업 공간에 붙여넣습니다.

```
cd ~/environment/ecsdemo-frontend
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

배포 상태를 보면 진행 상황을 볼 수 있습니다.

```
kubectl get deployment ecsdemo-frontend
```

### 서비스 주소 찾기
이제 실행 중인 서비스가 type: LoadBalancer있으므로 ELB의 주소를 찾아야 합니다. get serviceskubectl 작업을 사용하여 이 작업을 수행할 수 있습니다 .

```
kubectl get service ecsdemo-frontend
```

필드가 ELB의 FQDN을 표시할 만큼 충분히 넓지 않다는 점에 유의하십시오. 다음 명령으로 출력 형식을 조정할 수 있습니다.

```
kubectl get service ecsdemo-frontend -o wide
```

데이터를 프로그래밍 방식으로 사용하려면 json을 통해 출력할 수도 있습니다. 다음은 json 출력을 사용할 수 있는 방법의 예입니다.

```
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')

curl -m3 -v $ELB
```

ELB가 정상 상태가 되고 트래픽을 프런트엔드 포드로 전달하기 시작하는 데 몇 분이 걸립니다.

또한 loadBalancer 호스트 이름을 브라우저에 복사/붙여넣기하고 실행 중인 애플리케이션을 볼 수 있어야 합니다. 다음 페이지에서 서비스를 확장하는 동안 이 탭을 열어 두십시오.

### 백엔드 서비스 확장
서비스를 출시할 때 각각 하나의 컨테이너만 출시했습니다. 실행 중인 포드를 보면 이를 확인할 수 있습니다.

```
kubectl get deployments
```

이제 백엔드 서비스를 확장해 보겠습니다.

```
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
```

배포를 다시 확인하여 확인합니다.

```
kubectl get deployments
```

또한 실행 중인 애플리케이션을 볼 수 있는 브라우저 탭을 확인하십시오. 이제 여러 백엔드 서비스로 흐르는 트래픽이 표시되어야 합니다.

### 프런트엔드 확장
도전:
프론트엔드 서비스도 확장해 봅시다!

 솔루션을 보려면 여기를 확장하십시오.

```
kubectl get deployments
kubectl scale deployment ecsdemo-frontend --replicas=3
kubectl get deployments
```

실행 중인 애플리케이션을 볼 수 있는 브라우저 탭을 확인하십시오. 이제 여러 프런트엔드 서비스로 트래픽이 흐르는 것을 볼 수 있습니다.

### 애플리케이션 정리
애플리케이션에서 생성한 리소스를 삭제하려면 애플리케이션 배포를 삭제해야 합니다.

애플리케이션 배포 취소:

```
cd ~/environment/ecsdemo-frontend
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```

```
cd ~/environment/ecsdemo-crystal
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```

```
cd ~/environment/ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```

