# 애플리케이션의 컨테이너화와 Kubernetes 배포

## 🎯 프로젝트 목적

- 애플리케이션을 **Docker 이미지**로 빌드하고 Docker Hub에 Push  
- Kubernetes에 **배포 설정(Deployment / Service / Ingress)** 을 적용해 클러스터에 배포  
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

## 시스템 아키텍처

<p align="center">
<img width="720" height="532" alt="image" src="https://github.com/user-attachments/assets/66e6178d-12f8-4f20-ad69-8563c92736cf" />
</p>
<br>

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
  <img src="https://i.postimg.cc/Znx1V9Nv/image.png" alt="Docker 이미지 빌드 & 태그 예시" width="500">
</p>

---

## 3) Docker Hub에 이미지 Push

```bash
docker login
docker push ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION}
```
<p align="center">
  <img src="https://i.postimg.cc/Hx9gPPDL/image.png" alt="스크린샷" width="500">
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
  <img width="500" height="250" alt="image" src="https://github.com/user-attachments/assets/9f27ecae-3dba-429a-9192-7e0c425c890d" />
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
<img width="700" height="325" alt="image" src="https://github.com/user-attachments/assets/f5daf8af-3300-42d7-8aae-2981f8a69a97" />
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
<img width="700" height="200" alt="image" src="https://github.com/user-attachments/assets/e82f980b-570e-4105-9ac3-7bdfa24d36c5" />
</p>

- 브라우저에서 `http://gmg.local` 접속  
- index.html의 내용(“접속 성공”)이 보이면 성공 ✅

<p align="center">
<img width="700" height="300" alt="image" src="https://github.com/user-attachments/assets/5baf35eb-16b8-40a7-92b2-bf71262a939a" />
</p>


## 🛠️ 트러블 슈팅

### 1. **현상**
- Ingress 리소스를 배포했으나, 외부 요청이 정상적으로 Service로 전달되지 않음.
- 브라우저 접속 및 curl 요청 시 404 Not Found 또는 연결 불가 현상 발생.
  
### 2. **원인**
- Ingress 리소스에서 **backend service의 name**과 실제 Service 리소스의 **metadata.name**이 일치하지 않았음
- Ingress 리소스에서 지정한 **servicePort name/number**와 Service 리소스에서 정의한 **port name/number**가 불일치했음
    
    요약: Ingress Controller는 host/path → Service 매핑 시 name과 port 매칭을 엄격하게 검사하므로, 오탈자나 불일치가 있으면 트래픽을 라우팅하지 못함
    
### 3. **해결 방법**
- Ingress의 service.name 값을 Service의 metadata.name과 동일하게 수정
- Ingress service.port 를 Service 정의와 동일하게 맞춤

---

### 1. 현상

- Ingress 리소스를 배포했으나, curl http://<노드IP>:<포트> 형식으로 접근 시 응답이 오지 않음.
- http://<서비스-ClusterIP>:<포트> 접근 또한 외부에서는 불가능.
- 반대로 curl http://<Ingress에 정의한 Host>/... 형식으로 도메인 기반 요청 시에는 정상 응답.

### 2. 원인

- Service 타입을 ClusterIP로 설정했기 때문에 클러스터 내부에서만 접근 가능함.
- Ingress는 L7 라우터 역할을 하며, 외부 → 내부 트래픽은 Ingress Controller의 LoadBalancer/NodePort 포트로 들어와야만 서비스까지 도달함.
- 따라서 외부에서 직접 ClusterIP:Port로 호출하면 응답이 없는 것이 정상 동작임.

  요약: Ingress를 쓸 경우 외부 접근은 반드시 Ingress Controller의 진입점(도메인/NodePort/LoadBalancer)으로만 가능하고, Service의 ClusterIP는 내부 전용이므로 외부 IP:Port 호출은 실패한다.

### 3. **해결 방법**
- Service 리소스를 Clusterip가 아닌 NodePort로 선언하여 curl http://<노드IP>:<포트> 형식으로 접근 
