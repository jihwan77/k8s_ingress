# 애플리케이션의 컨테이너화와 Kubernetes 배포

> **목표:**  
> 애플리케이션을 **컨테이너 이미지**로 빌드하고, Docker Hub에 Push한 뒤  
> **Kubernetes Deployment / Service / Ingress** 리소스를 이용해 클러스터에 배포하여  
> 외부에서 접근 가능한 환경을 구축합니다.

---

## 1) index.html 생성

프로젝트 루트 디렉토리에 간단한 `index.html` 파일을 생성합니다.  

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

이후 Dockerfile을 통해 이 HTML 파일을 컨테이너 이미지에 포함시켜 배포합니다.

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

# 애플리케이션 JAR 파일 복사 (예시)
COPY SpringAppSample2-0.0.1-SNAPSHOT.jar app.jar

# 애플리케이션 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**이미지 빌드 & 태그**

```bash
docker build -t ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION} .
```

<p align="center">
  <img src="https://i.postimg.cc/Znx1V9Nv/image.png" alt="Docker 이미지 빌드 & 태그 예시" width="800">
</p>

---

## 3) Docker Hub에 이미지 Push

```bash
docker login
docker push ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION}
```
<p align="center">
  <img src="https://i.postimg.cc/Hx9gPPDL/image.png" alt="스크린샷" width="800">
</p>


---

## 4) Kubernetes 매니페스트 작성 (Deployment, Service)

### 4-1) `gmg-ingressdeploysvc.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gmg-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring
  template:
    metadata:
      labels:
        app: spring
    spec:
      containers:
      - name: gmg-spring
        image: minkyoung2/gmg:latest
        ports:
        - containerPort: 80
```

### 4-2) `gmg-clusterip.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gmg-service
spec:
  type: ClusterIP
  selector:
    app: spring
  ports:
    - port: 80
      targetPort: 80
```

**리소스 적용**

```bash
kubectl apply -f gmg-ingressdeploysvc.yaml
kubectl apply -f gmg-clusterip.yaml
```

---

## 5) Ingress Controller 설치

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

설치 완료 후 Ingress Controller Pod와 Service 상태를 확인합니다.

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## 6) Ingress 리소스 작성 및 적용

### `gmg-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gmg-ingress
  namespace: default  
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: gmg.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gmg-service
            port:
              number: 80
```

**적용**

```bash
kubectl apply -f gmg-ingress.yaml
kubectl get ingress
```

---

## 7) 외부 접속 확인

- Windows `hosts` 파일에 Ingress Controller IP와 `gmg.local` 매핑  
  ```
  <INGRESS_IP> gmg.local
  ```

- 브라우저에서 `http://gmg.local` 접속  
- index.html의 내용(“접속 성공”)이 보이면 성공 ✅
