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
  <img src="https://i.postimg.cc/sgpP0tDS/3-drawio-1.png" alt="Image Build & Push & Host Flow Diagram" width="800">
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


| 단계 | 내용 |
|------|------|
| **리소스 적용** | ```docker build -t ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION} .``` |
| **실행 결과** | <p align="center"><img width="500" height="250" alt="리소스 적용 결과" src="https://github.com/user-attachments/assets/9f27ecae-3dba-429a-9192-7e0c425c890d" /></p> |

---

## 3) Docker Hub에 이미지 Push

| 단계 | 내용 |
|------|------|
| **리소스 적용** | ```docker login``` / ```docker push ${YOUR_DOCKERHUB_ID}/${APP_NAME}:${APP_VERSION}``` |
| **실행 결과** | <p align="center"><img src="https://i.postimg.cc/Hx9gPPDL/image.png" /></p> |

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

| 단계 | 내용 |
|------|------|
| **리소스 적용** | ```kubectl apply -f gmg-ingressdeploysvc.yaml``` / ```kubectl apply -f gmg-clusterip.yaml``` |
| **실행 결과** | <p align="center"><img width="500" height="250" alt="image" src="https://github.com/user-attachments/assets/9f27ecae-3dba-429a-9192-7e0c425c890d" /></p> |

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

| 단계 | 내용 |
|------|------|
| **리소스 적용** | ```kubectl apply -f gmg-ingress.yaml``` / ```kubectl describe ingress gmg-ingress``` / ``` kubectl get ingress ``` |
| **실행 결과** | <p align="center"><img width="700" height="325" alt="image" src="https://github.com/user-attachments/assets/f5daf8af-3300-42d7-8aae-2981f8a69a97" /></p> |

---



## 7) 외부 접속 확인

- Windows `hosts` 파일에 Ingress Controller IP와 `gmg.local` 매핑  

| 단계 | 내용 |
|------|------|
| **리소스 적용** | ```<INGRESS_IP> gmg.local``` |
| **실행 결과** | <p align="center"><img width="700" height="325" alt="image" src="https://github.com/user-attachments/assets/e82f980b-570e-4105-9ac3-7bdfa24d36c5" /></p> |


- 브라우저에서 `http://gmg.local` 접속  
- index.html의 내용(“접속 성공”)이 보이면 성공 ✅

| 단계 | 내용 |
|------|------|
| **실행 결과** | <p align="center"><img width="700" height="325" alt="image" src="https://github.com/user-attachments/assets/5baf35eb-16b8-40a7-92b2-bf71262a939a" /></p> |


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
