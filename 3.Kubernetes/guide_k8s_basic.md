# Kubernetes 기초  

개발한 구독관리 서비스를 컨테이너 이미지로 빌드하고 AKS에 배포하면서 쿠버네티스에 대해 이해합니다.  

## 목차
- [Kubernetes 기초](#kubernetes-기초)
  - [목차](#목차)
  - [ingress controller 추가](#ingress-controller-추가)
  - [사전준비](#사전준비)
  - [네임스페이스 생성](#네임스페이스-생성)
    - [Database 설치](#database-설치)
  - [Backend Application 실행파일 빌드](#backend-application-실행파일-빌드)
  - [컨테이너 이미지 빌드](#컨테이너-이미지-빌드)
    - [Backend Application](#backend-application)
    - [Front Application](#front-application)
  - [컨테이너 이미지 푸시](#컨테이너-이미지-푸시)
    - [ACR로그인](#acr로그인)
    - [Backend Application 이미지 푸시](#backend-application-이미지-푸시)
    - [Frontend Application 이미지 푸시](#frontend-application-이미지-푸시)
  - [Kubernetes Manifest 준비](#kubernetes-manifest-준비)
    - [ALLOWED\_ORIGINS값 셋팅](#allowed_origins값-셋팅)
    - [매니페스트 이해](#매니페스트-이해)
    - [Manifest 점검 및 실행](#manifest-점검-및-실행)
    - [manifest 실행](#manifest-실행)
    - [정상 배포 확인](#정상-배포-확인)
    - [테스트](#테스트)

---

## ingress controller 추가
쿠버네티스 설치 시 Ingress Controller가 기본으로 설치 안되기 때문에 먼저 그거부터 설치해야 합니다.  
Ingress Controller 중 가장 많이 사용하는 nginx ingress controller를 추가합니다.  

작업 디렉토리를 먼저 만듭니다.  
```
mkdir -p ~/install/ingress-controller && cd ~/install/ingress-controller 
```

helm으로 설치할 겁니다.  
따라서 helm repository 부터 추가해야겠죠?  
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update 
```

아래 내용으로 ingress-values.yaml 파일을 만듭니다.  
appProtocol 옵션을 비활성해야 제대로 생성이 됩니다.  
```
controller:
  replicaCount: 1
  service:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: /healthz
    loadBalancerIP: ""
    
    #해당 포트가 어떤 애플리케이션 프로토콜을 사용하는지 명시적으로 지정하는 옵션 비활성화
    #targetPort를 named port("http", "https")로 매핑하려고 시도해서 
    #Ingress Nginx Controller pod의 container port는 숫자(80, 443)로 정의되어 있어서 매핑이 실패 
    appProtocol: false  
    
  config:
    use-forwarded-headers: "true"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

```

ingress-basic 네임 스페이스에 설치 합니다.    
```
helm upgrade -i ingress-nginx -f ingress-values.yaml \
-n ingress-basic --create-namespace ingress-nginx/ingress-nginx 
```

제대로 생성되었는지 체크해 봅니다.   
```
k get svc -n ingress-basic
k get po -n ingress-basic
```

잠깐 ingress controller가 어떻게 외부의 요청을 내부로 연결할까요?  
ingress controller 파드는 준실시간으로 ingress object들의 설정을 읽어 내부의 nginx.conf파일에 업데이트 합니다.  
외부에서는 ingress-nginx-controller의 L/B IP로 접근합니다.  
이 트래픽 요청의 Host와 경로에 따라 적절한 서비스 오브젝트로 proxying합니다.    
결론적으로 외부의 트래픽을 내부로 전달하는 것은 인그레스 오브젝트가 아니라 인그레스 컨트롤러 파드드인것입니다.  
![](images/2025-02-16-13-58-13.png)

내친김에 ingress controller pod의 nginx.conf 내용도 볼까요? 
아직 ingress 오브젝트를 만들지 않았기 때문에 지금은 실습 못하지만,  
이 실습이 완료된 후에 한번 직접 확인해 보십시오.  
```
k get po -n ingress-basic

k exec -it {ingress controller pod}  -n ingress-basic -- bash
ingress-nginx-controller-5d9dcdb7b8-dx65z:/etc/nginx$ cat nginx.conf | more
```

스페이스를 눌러 내려가다 보면 아래 예와 같은 설정이 있는걸 확인할 수 있을겁니다.  
보시면 아시겠죠? ingress 오브젝트 'backend-ingress'의 설정이 그대로 반영되어 있습니다.   
```
## start server _
server {
    ...
    location ~* "^/recommend(/|$)(.*)" {

        set $namespace      "${ID}-lifesub-ns";
        set $ingress_name   "backend-ingress";
        set $service_name   "recommend";
        set $service_port   "80";
        set $location_path  "/recommend(/|${literal_dollar})(.*)";
        ...
 
        # Custom Response Headers
        rewrite "(?i)/recommend(/|$)(.*)" /$2 break;
        proxy_pass http://upstream_balancer;
    }
    ...
}
```

| [Top](#목차) |

---

## 사전준비 

1.lifesub-ns의 리소스 삭제  
AKS의 자원이 부족할 수 있으므로 이전에 실습한 lifesub-ns의 모든 리소스부터 정리하겠습니다.    
```
helm delete member mysub recommend -n lifesub-ns
k delete pvc --all -n lifesub-ns
k delete deploy --all -n lifesub-ns
k delete cm --all -n lifesub-ns
k delete secret --all -n lifesub-ns
k delete ns lifesub-ns
```

2.실행을 위한 전역변수 셋팅  
```
export ID={본인ID}

```

3.실습을 위한 소스 클론  
```
cd ~/workspace
git clone https://github.com/cna-bootcamp/lifesub.git -b k8s
git clone https://github.com/cna-bootcamp/lifesub-web.git
```
만약 기존에 있다면 지우거나 옮기시고 하십시오.  

| [Top](#목차) |

---

## 네임스페이스 생성
```bash
k create ns ${ID}-lifesub-ns 2>/dev/null || true
kubens {ID}-lifesub-ns
```

### Database 설치
각 백엔드 서비스를 위한 PostgreSQL DB를 설치합니다.  
역시 Helm 차트로 설치합니다.  

Helm repository를 추가하고요. 
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

미리 만들어진 shell파일을 이용하여 설치합니다.  
```
cd ~/workspace/lifesub
chmod +x deployment/database/deploy_db.sh
./deployment/database/deploy_db.sh ${ID}-lifesub-ns
```

파드와 서비스가 잘 생성되었는지 확인합니다.  
```
k get po 
k get svc
```

| [Top](#목차) |

---

## Backend Application 실행파일 빌드    

```bash
# lifesub 디렉토리에서 수행
cd ~/workspace/lifesub

# member 서비스
./gradlew :member:clean :member:build -x test

# mysub 서비스
./gradlew :mysub-infra:clean :mysub-infra:build -x test

# recommend 서비스
./gradlew :recommend:clean :recommend:build -x test
```

## 컨테이너 이미지 빌드  
### Backend Application
```bash
# lifesub 디렉토리에서 수행
cd ~/workspace/lifesub

# member 서비스
docker build \
  --build-arg BUILD_LIB_DIR="member/build/libs" \
  --build-arg ARTIFACTORY_FILE="member.jar" \
  -f container/Dockerfile \
  -t ${ID}cr.azurecr.io/lifesub/member:1.0.0 .

# mysub 서비스
docker build \
  --build-arg BUILD_LIB_DIR="mysub-infra/build/libs" \
  --build-arg ARTIFACTORY_FILE="mysub.jar" \
  -f container/Dockerfile \
  -t ${ID}cr.azurecr.io/lifesub/mysub:1.0.0 .

# recommend 서비스
docker build \
  --build-arg BUILD_LIB_DIR="recommend/build/libs" \
  --build-arg ARTIFACTORY_FILE="recommend.jar" \
  -f container/Dockerfile \
  -t ${ID}cr.azurecr.io/lifesub/recommend:1.0.0 .
```

### Front Application 
아시다시피 프론트엔드는 클라이언트 디바이스(브라우저나 모바일 단말기)에서 백엔드를 호출합니다.  
그래서 각 백엔드 서비스의 호출 주소를 지정해 줘야 합니다.   
가장 간단한 방법은 백엔드 서비스를 LoadBalancer 타입으로 만들어 External IP를 지정하면 되겠죠?   
  
좀 더 세련된 방법이 없을까요?   
Ingress를 이용하는 방법이 더 나은 방법입니다.  왜일까요?   
백엔드 서비스를 외부에 직접 노출하지 않고 Ingress를 통해서 연결하니 확장성도 좋고 보안 측면에서도 더 좋기 때문입니다.     
  
혹시 프론트엔드도 파드이고 백엔드도 파드이니 백엔드의 서비스 오브젝트를 통해 통신하면 되지않아? 
라고 생각하시는 분이 있을 수 있습니다.   
안됩니다.  왜냐하면 프론트엔드와 백엔드가 서버 TO 서버 통신이 아니라 클라이언트 TO 서버 통신이기 때문입니다.   
'클라이언트 디바이스 -> Proxy서버 -> 백엔드 서비스'의 구조로 바꿔줘야 가능합니다.  
Proxy서버의 역할을 하는게 결국 Ingress Controller입니다.     

그래서 전략은 이렇습니다.  
- 프론트엔드는 Ingress Controller Service(ingress-nginx-controller)의 L/B IP를 호출   
- 호출 시 경로를 다르게 함. 예) http://{Ingress Controller Service L/B IP}/member   
- Ingress로 각 경로마다 Rewriting하여 각 백엔드의 서비스 객체를 연결. 예) /member/api/... -> http://member/api/...  
  
그럼 이 전략대로 각 백엔드 서비스의 주소를 셋팅하여 프론트엔드 이미지를 만들어 봅시다.   
아래와 같이 하면 됩니다.  
```bash
# lifesub-web 디렉토리에서 수행
cd ~/workspace/lifesub-web

ingress_host=$(kubectl get svc ingress-nginx-controller -n ingress-basic -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
echo "Ingress Host: ${ingress_host}"  
docker build \
  --build-arg PROJECT_FOLDER="." \
  --build-arg REACT_APP_MEMBER_URL="http://${ingress_host}/member" \
  --build-arg REACT_APP_MYSUB_URL="http://${ingress_host}/mysub" \
  --build-arg REACT_APP_RECOMMEND_URL="http://${ingress_host}/recommend" \
  --build-arg BUILD_FOLDER="container" \
  --build-arg EXPORT_PORT="18080" \
  -f container/Dockerfile \
  -t ${ID}cr.azurecr.io/lifesub/lifesub-web:1.0.0 .
```

| [Top](#목차) |

---

## 컨테이너 이미지 푸시  
### ACR로그인 
먼저 ACR 로그인 정보를 확인하고 로그인 합니다.  
```bash
# ACR 로그인
ACR_NAME="${ID}cr"
ACR_SERVER="${ACR_NAME}.azurecr.io"
ACR_USERNAME=$(az acr credential show -n ${ACR_NAME} --query "username" -o tsv)
ACR_PASSWORD=$(az acr credential show -n ${ACR_NAME} --query "passwords[0].value" -o tsv)

# Docker 로그인
echo ${ACR_PASSWORD} | docker login ${ACR_SERVER} -u ${ACR_USERNAME} --password-stdin
```

### Backend Application 이미지 푸시  
```bash
# member 서비스
docker push ${ID}cr.azurecr.io/lifesub/member:1.0.0

# mysub 서비스
docker push ${ID}cr.azurecr.io/lifesub/mysub:1.0.0

# recommend 서비스
docker push ${ID}cr.azurecr.io/lifesub/recommend:1.0.0
```

### Frontend Application 이미지 푸시  

```
docker push ${ID}cr.azurecr.io/lifesub/lifesub-web:1.0.0
```

| [Top](#목차) |

---

## Kubernetes Manifest 준비  
쿠버네티스 매니페스트 파일들은 이미 deployment 디렉토리 하위에 생성되어 있습니다.  
백엔드는 IntelliJ에서 오픈하여 확인하고, 프론트엔드는 VSCode에서 오픈하여 확인해 보십시오.   
각 매니페스트에 대한 설명은 조금 이따 하고 골치 아픈것 먼저 해결합시다.  
  
### ALLOWED_ORIGINS값 셋팅  
바로 CORS 설정을 위한 환경변수 'ALLOWED_ORIGINS' 셋팅입니다.   
메니페스트 'lifesub/deployment/manifest/configmaps/common-config.yaml'에 정의되어 있습니다.  
이 변수에는 프론트엔드의 주소가 들어가야 합니다.  
프론테엔드의 주소는 프론트엔드 서비스 객체의 L/B IP겠죠?   
즉, 매니페스트를 보면 lifesub-web이라는 서비스 객체의 External IP입니다.  
이 IP를 알려면 lifesub-web 서비스 객체를 배포할 수 밖에 없습니다.  
그리고 External IP를 구해서 'common-config.yaml'의 ALLOWED_ORIGINS값을 바꿔줘야 합니다.     
  
아래는 이 과정을 자동으로 처리하는 명령들입니다.    
```
cd ~/workspace
# lifesub-web service 생성
kubectl apply -f lifesub-web/deployment/manifest/services/lifesub-web-service.yaml

# lifesub-web의 External IP가 할당될 때까지 대기
echo "Waiting for LoadBalancer IP..."
while [ -z "$web_host" ]; do
  web_host=$(kubectl get svc lifesub-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
  if [ -z "$web_host" ]; then
    echo -n "."
    sleep 2
  fi
done
echo "LoadBalancer IP: ${web_host}"

cat > lifesub/deployment/manifest/configmaps/common-config.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: common-config
data:
  JPA_DDL_AUTO: update
  JPA_SHOW_SQL: "true"
  ALLOWED_ORIGINS: "http://localhost:18080,http://localhost:18081,http://${web_host}"
EOF

# 반영 확인
echo -e "\nVerifying configuration:"
echo "Web Service IP: ${web_host}"
echo -n "ALLOWED_ORIGINS: "
grep "ALLOWED_ORIGINS" lifesub/deployment/manifest/configmaps/common-config.yaml
```

### 매니페스트 이해
쿠버네티스 주요 리소스의 흐름 기억하시죠?  
Ingress -> Service -> Pod (ConfigMap, Secret, PVC 마운트)   
그리고 Pod를 관리하는 5가지 종류의 Workload Controller가 있습니다.  
Deployment, StatefulSet, DaemonSet, Job, CronJob이 그것이죠.   
  




---


### Manifest 점검 및 실행
DB Service 이름 확인:
```bash
kubectl get svc
```

### manifest 실행

기존 파드 삭제
```
cd ~/workspace
k delete -f lifesub/deployment/manifest/deployments/
k delete -f lifesub-web/deployment/manifest/deployments/
```

확인된 DB Service 이름으로 ConfigMap의 DB Host 수정 후 manifest 실행
```bash
cd ~/workspace

# Ingreess 생성
kubectl apply -f lifesub/deployment/manifest/ingresses/

# ConfigMap 생성
kubectl apply -f lifesub/deployment/manifest/configmaps/

# Secret 생성
kubectl apply -f lifesub/deployment/manifest/secrets/

# Backend 서비스 배포
kubectl apply -f lifesub/deployment/manifest/deployments/
kubectl apply -f lifesub/deployment/manifest/services/

# Frontend 서비스 배포
kubectl apply -f lifesub-web/deployment/manifest/deployments/
kubectl apply -f lifesub-web/deployment/manifest/services/
```

### 정상 배포 확인
```bash
# Service 상태 확인
kubectl get svc

# Pod 상태 확인
kubectl get pods -w

```

### 테스트
Frontend Service의 External IP로 접속하여 서비스 동작 확인

```
web_host=$(kubectl get svc lifesub-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
echo "테스트 주소: http://${web_host}"
```

| [Top](#목차) |

---
