# HELM

이 튜토리얼은 Helm v3용으로 업데이트되었습니다. 버전 3에서는 Tiller 구성 요소가 제거되어 운영이 간소화되고 보안이 향상되었습니다.

Helm v2에서 v3으로 마이그레이션해야 하는 경우 [공식 문서를 보려면 여기를 클릭하십시오](https://helm.sh/blog/migrate-from-helm-v2-to-helm-v3/).

[Helm](https://helm.sh) 은 여러 Kubernetes 리소스를 Chart 라는 단일 논리적 배포 단위로 패키징하는 Kubernetes용 패키지 관리자입니다 . 차트는 쉽게 만들고, 버전을 지정하고, 공유하고, 게시할 수 있습니다.

이 장에서 다룰 내용은 [Helm 설치](https://www.eksworkshop.com/beginner/060\_helm/helm\_intro). 설치가 완료되면 Helm을 사용하여 [간단한 nginx 웹 서버 배포](https://www.eksworkshop.com/beginner/060\_helm/helm\_nginx), 그리고 [더 정교한 마이크로서비스](https://www.eksworkshop.com/beginner/060\_helm/helm\_micro).

![](../Beginner/images/helm-logo.svg)

### Helm 소개

Helm은 여러 Kubernetes 리소스를 Chart 라는 단일 논리적 배포 단위로 패키징하는 Kubernetes용 패키지 관리자 및 애플리케이션 관리 도구입니다.

Helm은 다음을 수행하는 데 도움이 됩니다.

간단하고(하나의 명령) 반복 가능한 배포 달성 다른 애플리케이션 및 서비스의 특정 버전을 사용하여 애플리케이션 종속성 관리 여러 배포 구성 관리: 테스트, 스테이징, 프로덕션 및 기타 애플리케이션 배포 중 사후/사전 배포 작업 실행 업데이트/롤백 및 테스트 애플리케이션 배포

### HELM CLI 설치

Helm CLI 설치 Helm 구성을 시작하기 전에 먼저 상호 작용할 명령줄 도구를 설치해야 합니다. 이렇게 하려면 다음을 실행합니다.

```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

버전을 확인할 수 있습니다

```
helm version --short
```

첫 번째 차트 저장소를 구성해 보겠습니다. 차트 리포지토리는 Linux에서 친숙할 수 있는 APT 또는 yum 리포지토리 또는 macOS의 Homebrew용 Taps와 유사합니다.

stable리포지토리를 다운로드하여 다음으로 시작할 수 있습니다.

```
helm repo add stable https://charts.helm.sh/stable
```

이것이 설치되면 설치할 수 있는 차트를 나열할 수 있습니다.

```
helm search repo stable
```

마지막으로 helm명령에 대한 Bash 완료를 구성해 보겠습니다 .

```
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```

### Helm으로 nginx 배포

이 장에서는 Helm에 대해 더 자세히 알아보고 다음 단계를 통해 nginx 웹 서버를 설치하는 방법을 보여줍니다.

차트 저장소 업데이트 차트 저장소 검색 Bitnami 저장소 추가 비트나미/nginx 설치 정리

### 차트 저장소 업데이트

Helm은 패키징 형식을 사용합니다. 차트. 차트는 Kubernetes 리소스를 설명하는 파일 및 템플릿 모음입니다.

차트는 독립형 웹 서버(우리가 만들 예정)와 같은 것을 설명하는 단순할 수 있지만, 예를 들어 웹 서버, 데이터베이스, 프록시 등

를 통해 Kubernetes 리소스를 수동으로 설치하는 대신 kubectlHelm을 사용하여 오타나 기타 운영자 오류의 가능성이 적은 미리 정의된 차트를 더 빠르게 설치할 수 있습니다.

차트 저장소는 업데이트 및 새로운 추가로 인해 자주 변경됩니다. 이러한 모든 변경 사항으로 Helm의 로컬 목록을 최신 상태로 유지하려면 때때로 다음을 실행해야 합니다.저장소 업데이트 명령.

Helm의 로컬 차트 목록을 업데이트하려면 다음을 실행하세요.

```
# first, add the default repository, then update
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

다음과 유사한 내용이 표시되어야 합니다.

```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

다음으로 nginx 웹 서버 차트를 검색합니다.

### 차트 저장소 검색

이제 저장소 차트 목록이 업데이트되었으므로 차트 검색.

모든 차트를 나열하려면:

```
helm search repo
```

다음과 유사한 결과가 출력되어야 합니다.

```
NAME                                    CHART VERSION   APP VERSION                     DESCRIPTION
stable/acs-engine-autoscaler            2.2.2           2.1.1                           Scales worker...
stable/aerospike                        0.3.2           v4.5.0.5                        A Helm chart...
...
```

우리가 추가한 모든 차트 목록을 덤프한 출력을 볼 수 있습니다. 어떤 경우에는 유용할 수 있지만 훨씬 더 유용한 검색에는 키워드 인수가 포함됩니다. 다음으로 다음을 검색합니다 nginx.

```
helm search repo nginx
```

그 결과:

```
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
stable/nginx-ingress            1.30.3          0.28.0          An nginx Ingress ...
stable/nginx-ldapauth-proxy     0.1.3           1.13.5          nginx proxy ...
stable/nginx-lego               0.3.1                           Chart for...
stable/gcloud-endpoints         0.1.2           1               DEPRECATED Develop...
...
```

이 새로운 차트 목록은 명령에 nginx 인수를 전달했기 때문에 nginx에만 해당됩니다 helm search repo.

명령에 대한 추가 정보를 찾을 수 있습니다. [여기](https://helm.sh/docs/helm/helm\_search\_repo/).

### BITNAMI 저장소 추가

마지막 슬라이드에서 nginx는 기본 Helm Chart 리포지토리를 통해 다양한 제품을 제공하지만 nginx 독립 실행형 웹 서버는 그 중 하나가 아님을 확인했습니다.

빠른 웹 검색 후 nginx 독립 실행형 웹 서버용 차트가 있음을 발견했습니다. [Bitnami 차트 저장소](https://github.com/bitnami/charts).

Bitnami 차트 저장소를 검색 가능한 차트의 로컬 목록에 추가하려면:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

완료되면 모든 Bitnami 차트를 검색할 수 있습니다.

```
helm search repo bitnami
```

결과:

```
NAME                                    CHART VERSION   APP VERSION             DESCRIPTION
bitnami/bitnami-common                  0.0.8           0.0.8                   Chart with...
bitnami/apache                          4.3.3           1.10.9                  Chart for Apache...
bitnami/cassandra                       5.0.2           3.11.6                  Apache Cassandra...
...
```

다시 nginx 검색

```
helm search repo nginx
```

이제 두 저장소 모두에서 더 많은 nginx 옵션이 표시됩니다.

```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/nginx                           5.1.6           1.16.1          Chart for the nginx server
bitnami/nginx-ingress-controller        5.3.4           0.29.0          Chart for the nginx Ingress...
stable/nginx-ingress                    1.30.3          0.28.0          An nginx Ingress controller ...
```

또는 nginx에 대해서만 Bitnami 저장소를 검색할 수도 있습니다.

```
helm search repo bitnami/nginx
```

Bitnami에서 nginx로 좁힙니다.

```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/nginx                           5.1.6           1.16.1          Chart for the nginx server
bitnami/nginx-ingress-controller        5.3.4           0.29.0          Chart for the nginx Ingress...
```

지난 두 번의 검색 모두에서

```
bitnami/nginx
```

검색 결과로. 이것이 바로 우리가 찾고 있는 것이므로 Helm을 사용하여 EKS 클러스터에 설치하겠습니다.

### BITNAMI/NGINX 설치

Bitnami 독립 실행형 nginx 웹 서버 설치 차트에는 다음을 사용하는 것이 포함됩니다. 키 설치 명령.

Helm 차트는 Kubernetes 클러스터 내부에 여러 번 설치할 수 있습니다. Chart의 각 설치는 다른 목적에 맞게 사용자 정의할 수 있기 때문입니다.

이러한 이유로 설치에 고유한 이름을 제공하거나 Helm에 이름 생성을 요청해야 합니다.

도전: Helm을 사용하여 bitnami/nginx 차트를 배포하는 방법은 무엇입니까?

힌트 : 사용 helm에 유틸리티를 차트와 이름을 지정 는 Kubernetes 배포. 상담 installbitnami/nginxmywebserver키 설치 설명서를 참조하거나 helm install --help명령을 실행 하여 구문을 파악하십시오.

솔루션을 보려면 여기를 확장하십시오.

```
helm install mywebserver bitnami/nginx
```

이 명령을 실행하면 다음과 유사한 배포 상태, 개정판, 네임스페이스 등에 대한 정보가 출력에 포함됩니다.

```
NAME: mywebserver
LAST DEPLOYED: Tue Feb 18 22:02:13 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the NGINX URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w mywebserver-nginx'

  export SERVICE_IP=$(kubectl get svc --namespace default mywebserver-nginx --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "NGINX URL: http://$SERVICE_IP/"
```

기본 Kubernetes 서비스, 포드 및 배포를 검토하려면 다음을 실행하세요.

```
kubectl get svc,po,deploy
```

다음 kubectl명령 예에서 이러한 각 개체 DESIRED및 CURRENT값이 일치 하는 데 1\~2분 정도 걸릴 수 있습니다 . 첫 번째 시도에서 일치하지 않으면 몇 초 기다렸다가 명령을 다시 실행하여 상태를 확인합니다.

이 출력에 표시된 첫 번째 객체는 전개. Deployment 개체는 다른 버전의 응용 프로그램의 롤아웃(및 롤백)을 관리합니다.

다음 명령을 실행하여 이 배포 개체를 더 자세히 검사할 수 있습니다.

```
kubectl describe deployment mywebserver
```

Chart에 의해 생성된 다음 객체는 다음과 같습니다. 현물 상환 지불. Pod는 하나 이상의 컨테이너 그룹입니다.

Pod 개체가 성공적으로 배포되었는지 확인하기 위해 다음 명령을 실행할 수 있습니다.

```
kubectl get pods -l app.kubernetes.io/name=nginx
```

다음과 유사한 출력이 표시되어야 합니다.

```
NAME                                 READY     STATUS    RESTARTS   AGE
mywebserver-nginx-85985c8466-tczst   1/1       Running   0          10s
```

이 차트가 생성하는 세 번째 객체는 서비스. 서비스를 통해 Elastic Load Balancer(ELB)를 통해 인터넷에서 이 nginx 웹 서버에 연결할 수 있습니다.

이 서비스의 전체 URL을 얻으려면 다음을 실행하십시오.

```
kubectl get service mywebserver-nginx -o wide
```

다음과 유사한 결과가 출력되어야 합니다.

```
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP
mywebserver-nginx   LoadBalancer   10.100.223.99   abc123.amazonaws.com
```

EXTERNAL-IP 의 값을 복사하여 웹 브라우저에서 새 탭을 열고 붙여넣습니다.

ELB 및 관련 DNS 이름을 사용할 수 있게 되는 데 몇 분 정도 걸릴 수 있습니다. 오류가 발생하면 1분 정도 기다렸다가 다시 로드를 누르세요.

서비스가 온라인 상태가 되면 다음과 유사한 환영 메시지가 표시됩니다.

![](../Beginner/images/welcome\_to\_nginx.png)

축하합니다! 이제 nginx 독립형 웹 서버를 EKS 클러스터에 성공적으로 배포했습니다!

## 정리

Helm 차트가 생성한 모든 개체를 제거하려면 다음을 [Helm 제거](https://helm.sh/docs/helm/helm\_uninstall/)를 사용할 수 있습니다.

애플리케이션을 제거하기 전에 다음을 통해 실행 중인 [Helm list](https://helm.sh/docs/helm/helm\_list/)항목을 확인할 수 있습니다.

```
helm list
```

mywebserver가 설치되었음을 보여주는 아래와 유사한 출력이 표시되어야 합니다.

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
mywebserver     default         1               2020-02-18 22:02:13.844416354 +0100 CET deployed        nginx-5.1.6     1.16.1
```

정말 재미있었어요. HTTP를 주고받는 좋은 시간을 보냈지만 이제 이 배포를 제거할 때입니다. 제거하려면:

```
helm uninstall mywebserver
```

그리고 다음과 같은 결과가 나와야 합니다.

```
release "mywebserver" uninstalled
```

kubectl은 또한 우리의 포드와 서비스를 더 이상 사용할 수 없음을 보여줍니다.

```
kubectl get pods -l app.kubernetes.io/name=nginx
kubectl get service mywebserver-nginx -o wide
```

페이지 새로고침을 통해 웹 브라우저를 통해 서비스에 액세스하려고 하는 것과 같습니다.

이렇게 하면 정리가 완료됩니다.

## Helm을 사용하여 예제 마이크로서비스 배포

이 장에서는 .NET을 사용하여 모든 작업을 수동으로 수행하는 대신 사용자 지정 Helm 차트를 사용하여 마이크로서비스를 배포하는 방법을 보여줍니다 kubectl.

차트 템플릿 작업에 대한 [Helm Docs](https://docs.helm.sh/chart\_template\_guide/) 자세한 내용

### 차트 만들기

Helm 차트의 구조는 다음과 유사합니다.

```
/eksdemo
  /Chart.yaml  # a description of the chart
  /values.yaml # defaults, may be overridden during install or upgrade
  /charts/ # May contain subcharts
  /templates/ # the template files themselves
  ...
```

이 템플릿 을 따라 다음 명령을 사용하여 eksdemo 라는 새 차트를 만듭니다 .

```
cd ~/environment
helm create eksdemo
```

### 기본값 사용자 정의

새로 생성된 eksdemo 디렉터리를 보면 여러 파일과 디렉터리를 볼 수 있습니다. 특히 /templates 디렉토리 내부에 다음이 표시됩니다.

deployment.yaml: Kubernetes 배포를 만들기 위한 기본 매니페스트 \_helpers.tpl: 차트 전체에서 재사용할 수 있는 템플릿 도우미를 넣는 곳 ingress.yaml: 서비스에 대한 Kubernetes 수신 객체를 생성하기 위한 기본 매니페스트 NOTES.txt: 차트의 "도움말 텍스트"입니다. 이것은 사용자가 helm install을 실행할 때 표시됩니다. serviceaccount.yaml: 서비스 계정 생성을 위한 기본 매니페스트. service.yaml: 배포를 위한 서비스 엔드포인트를 만들기 위한 기본 매니페스트 tests/: 차트 테스트가 포함된 폴더 우리는 실제로 우리 자신의 파일을 만들 것이므로 이러한 상용구 파일을 삭제할 것입니다.

```
rm -rf ~/environment/eksdemo/templates/
rm ~/environment/eksdemo/Chart.yaml
rm ~/environment/eksdemo/values.yaml
```

다음 코드 블록을 실행하여 차트를 설명하는 새 Chart.yaml 파일을 만듭니다.

```
cat <<EoF > ~/environment/eksdemo/Chart.yaml
apiVersion: v2
name: eksdemo
description: A Helm chart for EKS Workshop Microservices application
version: 0.1.0
appVersion: 1.0
EoF
```

다음으로 각 마이크로 서비스에 대한 매니페스트 파일을 템플릿 디렉토리에 servicename .yaml 로 복사합니다.

```
#create subfolders for each template type
mkdir -p ~/environment/eksdemo/templates/deployment
mkdir -p ~/environment/eksdemo/templates/service

# Copy and rename frontend manifests
cp ~/environment/ecsdemo-frontend/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/frontend.yaml
cp ~/environment/ecsdemo-frontend/kubernetes/service.yaml ~/environment/eksdemo/templates/service/frontend.yaml

# Copy and rename crystal manifests
cp ~/environment/ecsdemo-crystal/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/crystal.yaml
cp ~/environment/ecsdemo-crystal/kubernetes/service.yaml ~/environment/eksdemo/templates/service/crystal.yaml

# Copy and rename nodejs manifests
cp ~/environment/ecsdemo-nodejs/kubernetes/deployment.yaml ~/environment/eksdemo/templates/deployment/nodejs.yaml
cp ~/environment/ecsdemo-nodejs/kubernetes/service.yaml ~/environment/eksdemo/templates/service/nodejs.yaml
```

템플릿 디렉토리의 모든 파일은 템플릿 엔진을 통해 전송됩니다. 현재 그대로 Kubernetes에 전송되는 일반 YAML 파일입니다.

하드 코딩된 값을 템플릿 지시문으로 교체 template directives하드 코딩된 값을 제거하여 더 많은 사용자 정의를 가능하게 하기 위해 일부 값을 로 바꾸겠습니다 .

Cloud9 편집기에서 \~/environment/eksdemo/templates/deployment/frontend.yaml을 엽니다.

다음 단계는 frontend.yaml , crystal.yaml 및 nodejs.yaml 에 대해 별도로 완료해야 합니다 .

에서 복제본: 1을spec 찾아 다음으로 바꿉니다 .

```
replicas: {{ .Values.replicas }}
```

에서 spec.template.spec.containers.image이미지를 아래 표의 올바른 템플릿 값으로 바꿉니다.

| 파일이름          | 값                                                             |
| ------------- | ------------------------------------------------------------- |
| frontend.yaml | - 이미지: \{{ .Values.frontend.image \}}:\{{ .Values.version \}} |
| crystal.yaml  | - 이미지: \{{ .Values.crystal.image \}}:\{{ .Values.version \}}  |
| nodejs.yaml   | - 이미지: \{{ .Values.nodejs.image \}}:\{{ .Values.version \}}   |

템플릿 기본값으로 values.yaml 파일을 만듭니다. 다음 코드 블록을 실행하여 template directives기본값 을 채웁니다 .

```
cat <<EoF > ~/environment/eksdemo/values.yaml
# Default values for eksdemo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Release-wide Values
replicas: 3
version: 'latest'

# Service Specific Values
nodejs:
  image: brentley/ecsdemo-nodejs
crystal:
  image: brentley/ecsdemo-crystal
frontend:
  image: brentley/ecsdemo-frontend
EoF
```

### EKSDEMO 차트 배포

테스트 실행 플래그를 사용하여 템플릿 테스트 차트를 실제로 배포하지 않고 차트의 구문과 유효성을 테스트하기 위해 --dry-run플래그를 사용합니다 .

다음 명령은 차트를 설치하지 않고 렌더링된 템플릿을 빌드하고 출력합니다.

```
helm install --debug --dry-run workshop ~/environment/eksdemo
```

템플릿에서 생성한 값이 올바른지 확인합니다.

차트 배포 이제 템플릿을 테스트했으므로 설치해 보겠습니다.

```
helm install workshop ~/environment/eksdemo
```

우리가 이전에 본 것과 유사합니다. nginx Helm 차트 예제, 명령 출력에는 다음과 유사한 배포 상태, 개정판, 네임스페이스 등에 대한 정보가 포함됩니다.

```
NAME: workshop
LAST DEPLOYED: Tue Feb 18 22:11:37 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

기본 서비스, 포드 및 배포를 검토하려면 다음을 실행합니다.

```
kubectl get svc,po,deploy
```

## 서비스 테스트

eksdemo 차트가 생성한 서비스를 테스트하려면 차트를 배포할 때 생성된 ELB 끝점의 이름을 가져와야 합니다.

```
kubectl get svc ecsdemo-frontend -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"; echo
```

해당 주소를 복사하여 브라우저의 새 탭에 붙여넣습니다. 다음과 유사한 내용이 표시되어야 합니다.

![](../Beginner/images/micro\_example.png)

### 롤백

배포 중에 실수가 발생하고 실수가 발생하면 Helm을 사용하여 쉽게 실행 취소하거나 이전에 배포된 버전으로 "롤백"할 수 있습니다.

주요 변경 사항으로 데모 애플리케이션 차트 업데이트 values.yaml을 열고 아래의 이미지 이름 nodejs.image을 brentley/ecsdemo-nodejs-non-existing 으로 수정 합니다. 이 이미지가 존재하지 않으므로 배포가 중단됩니다.

업데이트된 데모 애플리케이션 차트를 배포합니다.

```
helm upgrade workshop ~/environment/eksdemo
```

롤링 업그레이드는 새 이미지로 새 nodejs 포드를 생성하여 시작됩니다. 새 ecsdemo-nodejsPod는 존재하지 않는 이미지를 가져오는 데 실패해야 합니다. 오류 kubectl get pods를 보려면 실행 하십시오 ImagePullBackOff.

```
kubectl get pods
```

```
NAME                               READY   STATUS             RESTARTS   AGE
ecsdemo-crystal-844d84cb86-56gpz   1/1     Running            0          23m
ecsdemo-crystal-844d84cb86-5vvcg   1/1     Running            0          23m
ecsdemo-crystal-844d84cb86-d2plf   1/1     Running            0          23m
ecsdemo-frontend-6df6d9bb9-dpcsl   1/1     Running            0          23m
ecsdemo-frontend-6df6d9bb9-lzlwh   1/1     Running            0          23m
ecsdemo-frontend-6df6d9bb9-psg69   1/1     Running            0          23m
ecsdemo-nodejs-6fdf964f5f-6cnzl    1/1     Running            0          23m
ecsdemo-nodejs-6fdf964f5f-fbcjv    1/1     Running            0          23m
ecsdemo-nodejs-6fdf964f5f-v88jn    1/1     Running            0          23m
ecsdemo-nodejs-7c6575b56c-hrrsp    0/1     ImagePullBackOff   0          15m
```

실행 helm status workshop하여 LAST DEPLOYED타임스탬프 를 확인합니다 .

```
helm status workshop
```

```
LAST DEPLOYED: Tue Feb 18 22:14:00 2020
NAMESPACE: default
STATUS: deployed
...
```

이것은 의 마지막 항목과 일치해야 합니다. helm history workshop helm history workshop 실패한 업그레이드 롤백 이제 애플리케이션을 이전 작업 릴리스 버전으로 롤백할 것입니다.

먼저 Helm 릴리스 버전을 나열합니다.

```
helm history workshop
```

그런 다음 이전 애플리케이션 개정으로 롤백합니다(모든 개정으로도 롤백할 수 있음).

```
# rollback to the 1st revision
helm rollback workshop 1
```

다음을 사용 workshop하여 릴리스 상태를 확인하십시오.

```
helm status workshop
```

오류가 사라졌는지 확인

```
kubectl get pods
```

```
NAME                               READY   STATUS             RESTARTS   AGE
ecsdemo-crystal-844d84cb86-56gpz   1/1     Running            0          23m
ecsdemo-crystal-844d84cb86-5vvcg   1/1     Running            0          23m
ecsdemo-crystal-844d84cb86-d2plf   1/1     Running            0          23m
ecsdemo-frontend-6df6d9bb9-dpcsl   1/1     Running            0          23m
ecsdemo-frontend-6df6d9bb9-lzlwh   1/1     Running            0          23m
ecsdemo-frontend-6df6d9bb9-psg69   1/1     Running            0          23m
ecsdemo-nodejs-6fdf964f5f-6cnzl    1/1     Running            0          23m
ecsdemo-nodejs-6fdf964f5f-fbcjv    1/1     Running            0          23m
ecsdemo-nodejs-6fdf964f5f-v88jn    1/1     Running            0          23m
```

## 대청소

워크샵 릴리스를 삭제하려면 다음을 실행하십시오.

```
helm uninstall workshop
```
