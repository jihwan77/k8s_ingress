# 애플리케이션의 컨테이너화와 Kubernetes 배포

> **목표:**  
> 애플리케이션을 **컨테이너 이미지**로 만들어 레지스트리에 공유하고,  
> **Kubernetes Deployment / Service / Ingress**를 통해 클러스터에 배포하여  
> 외부에서 접근 가능한 환경을 구축합니다.

---

## 1) index.html 생성

- 프로젝트 루트 디렉토리에 `index.html` 파일을 생성 
- 이후 Dockerfile을 통해 이 HTML 파일을 컨테이너 이미지에 포함시켜 배포하게 될 것

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Insert title here</title>
</head>
<body>
  <h3>접속 성공</h3>
</body>
</html>
```

---

## 2) Dockerfile 작성 → 이미지 빌드

**프로젝트 루트에 `Dockerfile` 생성**

```dockerfile
# Base Image 설정
FROM openjdk:17-slim

# curl 설치 (slim 이미지에는 기본적으로 없음)
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# 작업 디렉토리 설정
WORKDIR /app

# 애플리케이션 JAR 파일 복사
COPY SpringAppSample2-0.0.1-SNAPSHOT.jar app.jar

# 애플리케이션 실행 (exec 방식 사용)
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**이미지 빌드 & 태그**

```bash
docker build -t ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION} .
```
<br>

<p align="center">
  <img src="https://i.postimg.cc/Znx1V9Nv/image.png" alt="Docker 이미지 빌드 & 태그 예시" width="800">
</p>

---

## 3) Docker Hub에 이미지 Push

```bash
docker login
docker push ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION}
```

> **Tip:** `latest` 태그도 함께 관리 가능
```bash
docker tag ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION} ${YOUR_DOCKERHUB_ID}/${APP_NAME}:latest
docker push ${YOUR_DOCKERHUB_ID}/${APP_NAME}:latest
```

---

## 4) Kubernetes 선언형 매니페스트 작성 (Deployment, Service)

### 4-1) `gmg-ingressdeploysvc.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  namespace: NAMESPACE
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: YOUR_DOCKERHUB_ID/APP_NAME:APP_VERSION
          ports:
            - containerPort: 80
```

### 4-2) `k8s/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
  namespace: NAMESPACE
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - name: http
      port: 80
      targetPort: 80
```

적용:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get all -n NAMESPACE
```

---

## 5) Ingress Controller 설치

### Minikube 예시
```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

### Helm 예시
```bash
kubectl create ns ingress-nginx --dry-run=client -o yaml | kubectl apply -f -

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx

kubectl -n ingress-nginx get svc,pods
```

---

## 6) Ingress 리소스 작성 및 적용

### `k8s/ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: NAMESPACE
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: HOST_NAME
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

적용:

```bash
kubectl apply -f k8s/ingress.yaml
kubectl get ingress -n NAMESPACE
```

Windows hosts 파일에 매핑:
```
<INGRESS_IP>   app.local
```

---

## 7) 외부 접속 확인

- Windows 브라우저에서 `http://app.local` 접속  
- index.html의 내용(“드디어 고망고”)이 보이면 성공 ✅

---

## 8) 점검 포인트

```bash
kubectl get pods -n NAMESPACE -o wide
kubectl get svc -n NAMESPACE
kubectl describe ingress app-ingress -n NAMESPACE
```

- Pod 3개가 모두 Ready 상태인지 확인  
- Service가 ClusterIP로 정상 연결됐는지 확인  
- Ingress 라우팅이 올바른지 확인  

---

## 파일 구조 예시

```
.
├─ index.html
├─ Dockerfile
├─ k8s/
│  ├─ deployment.yaml
│  ├─ service.yaml
│  └─ ingress.yaml
└─ README.md
```
