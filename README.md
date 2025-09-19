# 애플리케이션의 컨테이너화와 Kubernetes 배포

## 🎯 프로젝트 목적

- 애플리케이션을 **Docker 이미지**로 빌드하고 Docker Hub에 Push  
- Kubernetes에 **배포 설정(Deployment / Service / Ingress)**을 적용해 클러스터에 배포  
- **Ingress와 도메인 매핑을 통해 외부 브라우저에서 접속 가능**한 환경 구축  

---

## 🌠 진행 과정

1. **Spring Boot 프로젝트에 `index.html` 추가**
2. Dockerfile을 생성하고 **이미지 빌드 → Docker Hub Push**  
3. Kubernetes에 **배포용 설정 파일 적용**  
4. **NGINX Ingress Controller 설치**  
5. Ingress 설정을 적용해 도메인 연결  
6. Ingress Controller IP와 도메인 매핑  
7. 브라우저에서 `http://gmg.local` 접속

---

## 워크플로우 시각화

> 소스 → 컨테이너 이미지 빌드 → Docker Hub 푸시 → Kubernetes 배포(Deployment/Service) → Ingress 노출 → gmg.local로 외부 접속까지의 End-to-End 흐름.

<br>

<p align="center">
  <img src="https://i.postimg.cc/V6sdLtFD/Image-Build-Push-Host-Flow-Diagram.png" alt="Image Build & Push & Host Flow Diagram" width="800">
</p>
<br>

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

<p align="center">
  <img width="700" height="300" alt="image" src="https://github.com/user-attachments/assets/9f27ecae-3dba-429a-9192-7e0c425c890d" />
</p>

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
kubectl describe ingress gmg-ingress
kubectl get ingress
```
<p align="center">
<img width="720" height="325" alt="image" src="https://github.com/user-attachments/assets/f5daf8af-3300-42d7-8aae-2981f8a69a97" />
</p>

<p align="center">
<img width="700" height="200" alt="image" src="https://github.com/user-attachments/assets/c31cd05a-645f-42b1-80f0-71ab7378e879" />

---

## 7) 외부 접속 확인

- Windows `hosts` 파일에 Ingress Controller IP와 `gmg.local` 매핑  
  ```
  <INGRESS_IP> gmg.local
  ```
<p align="center">
<img width="600" height="200" alt="image" src="https://github.com/user-attachments/assets/e82f980b-570e-4105-9ac3-7bdfa24d36c5" />
</p>

- 브라우저에서 `http://gmg.local` 접속  
- index.html의 내용(“접속 성공”)이 보이면 성공 ✅

<p align="center">
<img width="720" height="306" alt="image" src="https://github.com/user-attachments/assets/5baf35eb-16b8-40a7-92b2-bf71262a939a" />
</p>
