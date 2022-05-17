# 상태 확인

기본적으로 Kubernetes는 어떤 이유로든 충돌하는 경우 컨테이너를 다시 시작합니다. 이는 트래픽을 보낼 정상 컨테이너를 식별하고 필요할 때 컨테이너를 다시 시작하여 강력한 애플리케이션을 실행하도록 구성할 수 있는 활성 및 준비 프로브를 사용합니다.

이 섹션에서는 어떻게 [활성 및 준비 상태 프로브](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) 포드의 다른 상태에 대해 동일하게 정의되고 테스트됩니다. 다음은 이러한 프로브의 작동 방식에 대한 높은 수준의 설명입니다.

활성 상태 프로브 는 Kubernetes에서 포드가 활성 상태인지 또는 작동 중지 상태인지 확인하는 데 사용됩니다. 포드는 다양한 이유로 데드 상태가 될 수 있습니다. Kubernetes는 활성 프로브가 통과하지 못하면 포드를 종료하고 다시 생성합니다.

Kubernetes에서 준비 프로브 를 사용하여 포드가 트래픽을 제공할 준비가 된 시점을 알 수 있습니다. 준비 상태 프로브가 통과한 경우에만 포드가 서비스에서 트래픽을 수신합니다. 준비 상태 프로브가 실패하면 트래픽이 포드로 전송되지 않습니다.

활성 및 준비 상태 프로브를 구성하기 위한 다양한 옵션을 이해하기 위해 이 모듈의 몇 가지 예를 검토합니다.

## 활성 프로브 구성
### 프로브 구성
아래 명령어를 사용하여 디렉토리 생성

```
mkdir -p ~/environment/healthchecks
```

다음 코드 블록을 실행하여 매니페스트 파일 ~/environment/healthchecks/liveness-app.yaml 을 채웁니다 . 구성 파일에서 livenessProbe필드는 kubelet이 컨테이너가 건강한지 여부를 고려하기 위해 컨테이너를 검사하는 방법을 결정합니다. kubelet은 periodSeconds 필드를 사용하여 컨테이너를 자주 확인합니다. 이 경우 kubelet은 5초마다 활성 프로브를 확인합니다. initialDelaySeconds 필드는 첫 번째 프로브를 수행하기 전에 5초 동안 기다려야 함을 kubelet에 알리는 데 사용됩니다. 프로브를 수행하기 위해 kubelet은 이 포드를 호스팅하는 서버에 HTTP GET 요청을 보내고 서버 /health에 대한 핸들러가 성공 코드를 반환하면 컨테이너는 정상으로 간주됩니다. 핸들러가 실패 코드를 반환하면 kubelet은 컨테이너를 종료하고 다시 시작합니다.

```
cat <<EoF > ~/environment/healthchecks/liveness-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-app
spec:
  containers:
  - name: liveness
    image: brentley/ecsdemo-nodejs
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
EoF
```

매니페스트를 사용하여 포드를 생성해 보겠습니다.

```
kubectl apply -f ~/environment/healthchecks/liveness-app.yaml
```

위의 명령은 활성 프로브가 있는 포드를 생성합니다.

```
kubectl get pod liveness-app
```

출력은 아래와 같습니다. RESTARTS 알림

```
NAME           READY     STATUS    RESTARTS   AGE
liveness-app   1/1       Running   0          11s
```

이 kubectl describe명령은 프로브 실패 또는 다시 시작을 표시하는 이벤트 기록을 표시합니다.

```
kubectl describe pod liveness-app
```

```
Events:
  Type    Reason                 Age   From                                    Message
  ----    ------                 ----  ----                                    -------
  Normal  Scheduled              38s   default-scheduler                       Successfully assigned liveness-app to ip-192-168-18-63.ec2.internal
  Normal  SuccessfulMountVolume  38s   kubelet, ip-192-168-18-63.ec2.internal  MountVolume.SetUp succeeded for volume "default-token-8bmt2"
  Normal  Pulling                37s   kubelet, ip-192-168-18-63.ec2.internal  pulling image "brentley/ecsdemo-nodejs"
  Normal  Pulled                 37s   kubelet, ip-192-168-18-63.ec2.internal  Successfully pulled image "brentley/ecsdemo-nodejs"
  Normal  Created                37s   kubelet, ip-192-168-18-63.ec2.internal  Created container
  Normal  Started                37s   kubelet, ip-192-168-18-63.ec2.internal  Started container
```

실패 소개
다음 명령을 실행하여 nodejs 애플리케이션에 SIGUSR1 신호를 보냅니다. 이 명령을 실행하여 도커 런타임의 애플리케이션 프로세스에 kill 신호를 보냅니다.

```
kubectl exec -it liveness-app -- /bin/kill -s SIGUSR1 1
```

15-20초 동안 기다린 후 포드를 설명하면 컨테이너를 종료하고 다시 시작하는 kubelet 작업을 알 수 있습니다.

```
Events:
  Type     Reason                 Age                From                                    Message
  ----     ------                 ----               ----                                    -------
  Normal   Scheduled              1m                 default-scheduler                       Successfully assigned liveness-app to ip-192-168-18-63.ec2.internal
  Normal   SuccessfulMountVolume  1m                 kubelet, ip-192-168-18-63.ec2.internal  MountVolume.SetUp succeeded for volume "default-token-8bmt2"
  Warning  Unhealthy              30s (x3 over 40s)  kubelet, ip-192-168-18-63.ec2.internal  Liveness probe failed: Get http://192.168.13.176:3000/health: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
  Normal   Pulling                0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  pulling image "brentley/ecsdemo-nodejs"
  Normal   Pulled                 0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  Successfully pulled image "brentley/ecsdemo-nodejs"
  Normal   Created                0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  Created container
  Normal   Started                0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  Started container
  Normal   Killing                0s                 kubelet, ip-192-168-18-63.ec2.internal  Killing container with id docker://liveness:Container failed liveness probe.. Container will be killed and recreated.
```

nodejs 애플리케이션이 SIGUSR1 신호로 디버그 모드에 들어갔을 때 상태 확인 핑에 응답하지 않았고 kubelet이 컨테이너를 종료했습니다. 컨테이너에 기본 다시 시작 정책이 적용되었습니다.

```
kubectl get pod liveness-app
```

출력은 다음과 같습니다.

```
NAME           READY     STATUS    RESTARTS   AGE
liveness-app   1/1       Running   1          12m
```

도전:
컨테이너 상태 확인의 상태를 어떻게 확인할 수 있습니까?

솔루션을 보려면 여기를 확장하십시오.

```
kubectl logs liveness-app
```

컨테이너가 충돌한 경우 플래그 kubectl logs가 있는 컨테이너의 이전 인스턴스화에서 로그를 검색 하는 데 사용할 수도 있습니다.--previous

```
kubectl logs liveness-app --previous
```

## 준비 프로브 구성
프로브 구성
다음 코드 블록을 실행하여 ~/environment/healthchecks/readiness-deployment.yaml 을 채웁니다 . readinessProbe 정의는 linux 명령이 상태 확인으로 구성될 수 있는 방법을 설명합니다. 빈 파일 /tmp/healthy 를 생성하여 준비 상태 프로브를 구성하고 동일한 파일을 사용하여 kubelet이 정상적인 포드만으로 배포를 업데이트하는 데 어떻게 도움이 되는지 이해합니다.

```
cat <<EoF > ~/environment/healthchecks/readiness-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-deployment
  template:
    metadata:
      labels:
        app: readiness-deployment
    spec:
      containers:
      - name: readiness-deployment
        image: alpine
        command: ["sh", "-c", "touch /tmp/healthy && sleep 86400"]
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 3
EoF
```

이제 준비 상태 프로브를 테스트하기 위한 배포를 생성합니다.

```
kubectl apply -f ~/environment/healthchecks/readiness-deployment.yaml
```

위의 명령은 처음에 설명한 대로 3개의 복제본과 준비 상태 프로브가 있는 배포를 만듭니다.

```
kubectl get pods -l app=readiness-deployment
```

출력은 아래와 유사합니다.

```
NAME                                    READY     STATUS    RESTARTS   AGE
readiness-deployment-7869b5d679-922mx   1/1       Running   0          31s
readiness-deployment-7869b5d679-vd55d   1/1       Running   0          31s
readiness-deployment-7869b5d679-vxb6g   1/1       Running   0          31s
```

또한 서비스가 이 배포를 가리킬 때 트래픽을 처리하는 데 모든 복제본을 사용할 수 있는지 확인하겠습니다.

```
kubectl describe deployment readiness-deployment | grep Replicas:
```

출력은 다음과 같습니다.

```
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
```

실패 소개
위의 3에서 포드 중 하나를 선택하고 아래와 같이 명령을 실행 하여 준비 상태 프로브를 실패하게 만드는 /tmp/healthy 파일 을 삭제합니다 .

```
kubectl exec -it <YOUR-READINESS-POD-NAME> -- rm /tmp/healthy
```

readyiness-deployment-7869b5d679-922mx 가 예제 클러스터에서 선택되었습니다. / tmp를 / 건강한 파일이 삭제되었습니다. 준비 확인을 통과하려면 이 파일이 있어야 합니다. 아래는 명령을 내린 후의 상태입니다.

```
kubectl get pods -l app=readiness-deployment
```

출력은 아래와 유사합니다.

```
NAME                                    READY     STATUS    RESTARTS   AGE
readiness-deployment-7869b5d679-922mx   0/1       Running   0          4m
readiness-deployment-7869b5d679-vd55d   1/1       Running   0          4m
readiness-deployment-7869b5d679-vxb6g   1/1       Running   0          4m
```

트래픽은 위 배포의 첫 번째 포드로 라우팅되지 않습니다. 준비 열은 이 포드에 대한 준비 상태 프로브가 통과하지 않았으므로 준비되지 않은 것으로 표시되었음을 확인합니다.
이제 서비스가 이 배포를 가리킬 때 트래픽을 처리하는 데 사용할 수 있는 복제본을 확인합니다.

```
kubectl describe deployment readiness-deployment | grep Replicas:
```

출력은 다음과 같습니다.

```
Replicas:               3 desired | 3 updated | 3 total | 2 available | 1 unavailable
```

포드에 대한 준비 상태 프로브가 실패하면 엔드포인트 컨트롤러는 포드와 일치하는 모든 서비스의 엔드포인트 목록에서 포드를 제거합니다.

도전:
어떻게 Pod를 Ready 상태로 복원하시겠습니까?

 솔루션을 보려면 여기를 확장하십시오.

Pod 이름으로 아래 명령을 실행하여 /tmp/healthy 파일 을 다시 생성 합니다. 포드가 프로브를 통과하면 준비된 것으로 표시되고 트래픽을 다시 수신하기 시작합니다.
```
kubectl exec -it <YOUR-READINESS-POD-NAME> -- touch /tmp/healthy
```

```
kubectl get pods -l app=readiness-deployment
```

## 대청소
Liveness Probe 예제는 HTTP 요청을 사용했고 Readiness Probe는 포드의 상태를 확인하는 명령을 실행했습니다. 에 설명된 대로 TCP 요청을 사용하여 동일한 작업을 수행할 수 있습니다.선적 서류 비치.

```
kubectl delete -f ~/environment/healthchecks/liveness-app.yaml
kubectl delete -f ~/environment/healthchecks/readiness-deployment.yaml
```


