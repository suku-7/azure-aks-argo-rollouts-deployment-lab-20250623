# Model
## azure-aks-argo-rollouts-deployment-lab-20250623
https://labs.msaez.io/#/courses/cna-full/2c7ffd60-3a9c-11f0-833f-b38345d437ae/deploy-my-app-2024

## Azure AKS 환경에서 Argo Rollouts를 사용하여 카나리(Canary) 및 블루/그린(Blue/Green) 배포하는 실습
- Argo Rollouts 컨트롤러를 Kubernetes 클러스터에 설치하고, Rollout 객체를 사용하여 NGINX 애플리케이션에 대한 카나리 배포를 정의 및 실행합니다.
- kubectl argo rollouts CLI 툴과 대시보드를 사용하여 카나리 배포의 점진적인 트래픽 전환 과정을 실시간으로 모니터링하고 새로운 이미지로 업데이트합니다.
- 블루/그린 배포 전략으로 Rollout 객체를 재구성하고, 활성(Active) 및 미리 보기(Preview) 서비스를 통해 수동으로 트래픽을 전환하는 과정을 실습합니다.

## 사전 준비
Azure 계정 및 구독, Gitpod 워크스페이스, Spring Boot 애플리케이션 코드

![스크린샷 2025-06-23 160058](https://github.com/user-attachments/assets/7dcf3f87-4efa-494a-8ed4-7600b9284cb5)
![스크린샷 2025-06-23 161332](https://github.com/user-attachments/assets/7f90f48f-1af1-4a86-a2a9-9199bf95d635)
![스크린샷 2025-06-23 163436](https://github.com/user-attachments/assets/b4a4da79-9c18-4e78-a295-d0ebea60322b)
![스크린샷 2025-06-23 163514](https://github.com/user-attachments/assets/f3fa301c-ef37-4052-a186-69220a720016)
![스크린샷 2025-06-23 163828](https://github.com/user-attachments/assets/7e3e1286-cca5-4336-8ded-52124b6c9af0)
![스크린샷 2025-06-23 163843](https://github.com/user-attachments/assets/07c0d1d1-451d-4049-880b-a280a6400dff)
![스크린샷 2025-06-23 163900](https://github.com/user-attachments/assets/a3a54d6c-0a18-4161-9382-8852c508ae0e)
![스크린샷 2025-06-23 164505](https://github.com/user-attachments/assets/b837eef8-c5c7-4a0d-99a7-3866e604796b)
![스크린샷 2025-06-23 164512](https://github.com/user-attachments/assets/4a0f27ac-1376-4be9-9c84-6075aa23a0d7)
![스크린샷 2025-06-23 164812](https://github.com/user-attachments/assets/79b3b182-a600-451b-9943-8b5d9a793b81)
![스크린샷 2025-06-23 164858](https://github.com/user-attachments/assets/2bd20efc-aed4-4c01-b9ff-3b200c5aee30)
![스크린샷 2025-06-23 165315](https://github.com/user-attachments/assets/57e690a6-0747-4f53-91be-dcd490b795fa)
![스크린샷 2025-06-23 171122](https://github.com/user-attachments/assets/49dedc69-85bc-4017-a2fd-45c7bb3696ea)

---

## 실습 단계별 상세 설명

0. 초기 환경 설정 및 준비
--- 터미널 1 (메인 작업) ---
```
# Gitpod 환경 초기화 (재실행 시 필요한 스크립트)
./init.sh

# Java SDK 설치/업그레이드 (선택 사항, 필요 시)
sdk install java

# Azure 로그인 및 AKS 클러스터 자격 증명 설정 (클러스터 연결 확인)
az login --use-device-code
# (프롬프트에 따라 구독 선택 및 인증 진행)
az aks get-credentials --resource-group a071098-rsrcgrp --name a071098-aks

# 현재 네임스페이스 목록 확인
kubectl get ns

# shop 네임스페이스가 있다면 삭제하여 이전 리소스 정리 (선택 사항)
kubectl delete ns shop 2>/dev/null || true
```
1. Argo Rollouts 컨트롤러 설치
--- 터미널 1 (메인 작업) ---
```
# Argo Rollouts 컨트롤러를 위한 네임스페이스 생성
kubectl create ns argo-rollouts

# 네임스페이스 생성 확인
kubectl get ns

# Argo Rollouts 컨트롤러 배포
# (GitHub 공식 릴리즈에서 최신 install.yaml 파일을 다운로드하여 적용합니다.)
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Argo Rollouts 컨트롤러 Pod들이 정상적으로 실행되는지 확인
kubectl get all -n argo-rollouts
```
2. Argo Rollout 객체 생성 (카나리 배포 전략)
--- 터미널 1 (메인 작업) ---
```
# 카나리 배포 전략이 정의된 Rollout 및 Service YAML 파일 생성
# 이 YAML은 NGINX 애플리케이션을 카나리 방식으로 배포하며,
# 트래픽을 10%, 20%, 30%, 40%로 점진적으로 증가시키며 10초씩 일시 정지합니다.
# Service는 LoadBalancer 타입으로 NGINX Pod에 트래픽을 라우팅합니다.
cat <<EOF > rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: example-rollout
spec:
  replicas: 10 # 총 NGINX Pod 개수
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.10 # 초기 배포할 이미지 버전
        ports:
        - containerPort: 80
  minReadySeconds: 30 # Pod가 Ready 상태로 유지되어야 하는 최소 시간
  revisionHistoryLimit: 3 # 유지할 이전 버전의 Rollout 이력 개수
  strategy:
    canary: # 카나리 배포 전략 설정
      maxSurge: "25%" # 새로운 Pod 생성 시 허용할 최대 추가 Pod 비율
      maxUnavailable: 0 # Pod 업데이트 중 허용할 최대 사용 불가능 Pod 비율
      steps: # 트래픽 전환 및 일시 정지 단계
      - setWeight: 10 # 트래픽 10%를 신규 Pod로 전환
      - pause:
          duration: 10s # 10초 동안 현재 상태 유지
      - setWeight: 20
      - pause:
          duration: 10s
      - setWeight: 30
      - pause:
          duration: 10s
      - setWeight: 40
      - pause:
          duration: 10s
---
apiVersion: "v1"
kind: "Service"
metadata:
  name: "nginx" # Rollout에서 참조할 서비스 이름
  labels:
    app: "nginx"
spec:
  ports:
    -
      port: 80
      targetPort: 80
  selector:
    app: "nginx" # Rollout의 Pod와 동일한 레이블을 선택
  type: "LoadBalancer" # 외부에서 접근 가능한 LoadBalancer 서비스
EOF
```
3. 카나리 배포 실행 및 확인
--- 터미널 1 (메인 작업) ---
```
# 정의한 Rollout 및 Service 객체 배포
kubectl apply -f rollout.yaml

# 배포된 모든 Kubernetes 리소스 확인
kubectl get all

# Argo Rollout 객체 상태 확인
# DESIRED, CURRENT, UP-TO-DATE, AVAILABLE 필드를 통해 배포 진행 상황을 모니터링합니다.
kubectl get rollout
```
4. Argo CLI 및 대시보드 설치
--- 터미널 1 (메인 작업) ---
```
# kubectl-argo-rollouts CLI 플러그인 다운로드 및 실행 권한 부여
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64

# CLI 툴을 시스템 PATH에 추가
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# 설치된 버전 확인
kubectl argo rollouts version

# Argo Rollouts 대시보드 웹 서비스를 로컬 포트로 포워딩하여 실행
# 이 명령을 실행하면 대시보드 웹 페이지로 접속할 수 있는 URL이 표시됩니다.
# Gitpod 환경에서는 'PORTS' 탭을 통해 자동으로 웹 미리보기 URL이 열릴 수 있습니다.
kubectl argo rollouts dashboard

# NGINX 서비스의 외부 IP 주소 확인 (웹 브라우저 접속용)
# LoadBalancer 타입 서비스 'nginx'의 EXTERNAL-IP를 확인합니다.
kubectl get svc
# 예시 출력:
# nginx    LoadBalancer    10.0.80.233    20.249.112.139    80:32572/TCP
```
5. 새로운 이미지로 카나리 배포 및 모니터링
--- 터미널 1 (메인 작업) ---
(Argo Rollouts 대시보드를 열어두고 실시간 변화를 관찰합니다.)
```
# NGINX Rollout의 이미지를 새로운 버전(jinyoung/app:blue)으로 업데이트
# 이 명령이 실행되면 카나리 배포 전략에 따라 트래픽이 점진적으로 전환됩니다.
kubectl argo rollouts set image example-rollout nginx=jinyoung/app:blue

# (옵션) 서비스의 외부 IP를 통해 지속적으로 요청을 보내면서 응답 변화 확인
# 웹 브라우저에서 LoadBalancer IP로 접속하거나, 터미널에서 `watch http [EXTERNAL-IP]` 명령 사용
# 예시: watch http http://20.249.112.139/

# (추가 예시) ACR(Azure Container Registry)에 올린 이미지로 업데이트
# kubectl argo rollouts set image example-rollout nginx=a071098.azurecr.io/order:v1
```
6. 블루/그린 배포 전략으로 전환 및 실행
--- 터미널 1 (메인 작업) ---
```
# 현재 Rollout 객체 및 관련 서비스 삭제 (블루/그린 테스트를 위해 초기화)
kubectl delete rollout example-rollout
kubectl delete service/nginx
kubectl get all # 삭제 확인
```
```
# 블루/그린 배포 전략이 정의된 Rollout 및 서비스 YAML 파일 생성
# 이 YAML은 ActiveService와 PreviewService를 분리하여 Blue/Green 배포를 설정합니다.
# autoPromotionEnabled: false 설정으로 수동 프로모션(트래픽 전환)을 유도합니다.
cat <<EOF > rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: example-rollout
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.10 # 초기 버전 (Blue)
        ports:
        - containerPort: 80
  strategy:
    blueGreen: # 블루/그린 배포 전략 설정
      activeService: nginx-active # 현재 서비스 중인 Pod들을 가리키는 서비스
      previewService: nginx-preview # 새 버전 Pod들을 가리키는 미리보기 서비스
      autoPromotionEnabled: false  # 자동 트래픽 전환 비활성화 (수동 승인 필요)
      previewReplicaCount: 2 # 미리보기 Pod의 초기 개수
---
apiVersion: "v1"
kind: "Service"
metadata:
  name: "nginx-active" # 활성 서비스
  labels:
    app: "nginx"
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: "nginx" # Rollout의 Pod를 선택
  type: "LoadBalancer" # 외부에서 접근 가능한 서비스
---
apiVersion: "v1"
kind: "Service"
metadata:
  name: "nginx-preview" # 미리보기 서비스 (처음엔 트래픽을 받지 않음)
  labels:
    app: "nginx"
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: "nginx" # Rollout의 Pod를 선택
  type: "LoadBalancer" # 외부에서 접근 가능한 서비스 (또는 ClusterIP로 내부에서만 접근 가능하게 설정 가능)
EOF
```
```
# 블루/그린 배포가 정의된 Rollout 및 Service 객체 배포
kubectl apply -f rollout.yaml

# 배포된 Rollout 상태 확인 (새로운 Pod가 Preview 상태로 준비됩니다.)
kubectl get rollout
```
7. 블루/그린 배포 트래픽 전환 (Promote)
--- 터미널 1 (메인 작업) ---
```
# 새로운 이미지(예: nginx:1.20.0 또는 jinyoung/app:blue)로 업데이트
# 이 시점에서 Preview Service를 통해 새로운 버전에 접근할 수 있지만, Active Service는 여전히 이전 버전을 가리킵니다.
kubectl argo rollouts set image example-rollout nginx=nginx:1.20.0 # 새 이미지로 변경

# Argo Rollouts 대시보드를 통해 현재 상태를 확인합니다.
# 수동으로 트래픽을 새로운 버전(Preview)으로 전환 (Promotion)
# 이 명령을 실행하면 ActiveService가 새로운 버전의 Pod들을 가리키게 됩니다.
kubectl argo rollouts promote example-rollout

# 트래픽 전환 후 Rollout 상태 확인
kubectl get rollout
```
