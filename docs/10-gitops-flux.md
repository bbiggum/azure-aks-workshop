# 10. GitOps — Flux v2로 배포 자동화

<details>
<summary><strong>⚠️ Cloud Shell 세션이 만료된 경우 — 환경 변수 재설정</strong></summary>

```bash
export RESOURCE_GROUP="WorkshopDemo-RG"
export CLUSTER_NAME="workshop-demo"
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --overwrite-existing
```

</details>

## 개요

지금까지는 `kubectl apply`로 직접 배포했지만, 프로덕션 환경에서는 **선언적 배포 자동화**가 필수입니다.  
**GitOps**는 Git 저장소를 단일 진실 원천(Single Source of Truth)으로 사용하여 클러스터 상태를 선언적으로 관리하는 운영 방식입니다.
이 섹션에서는 AKS의 **매니지드 Flux v2** (Azure Arc 기반 GitOps 확장)를 활용하여
Git 저장소의 매니페스트 변경이 자동으로 클러스터에 반영되는 과정을 체험합니다.

> [!NOTE]
> **AKS는 두 가지 매니지드 GitOps 옵션을 제공합니다.**  
> 이 워크샵은 **GA 안정성 + 빠른 실습**을 위해 Flux v2를 선택했지만, 실무에선 둘 중 어느 쪽을 택할지 클러스터 수와 팀 구조에 따라 결정합니다.
>
> | 옵션 | 상태 | 주요 강점 |
> |---|---|---|
> | **Flux v2** (이 워크샵) | ✅ **GA** | 가볍고 안정적, Azure Policy 통합, 단일 클러스터에 최적 |
> | **Argo CD** | 🟡 **Preview** (`Microsoft.ArgoCD` 확장) | 강력한 웹 UI, 멀티 클러스터 중앙 관리(Fleet Manager 조합) |
>
> 상세 비교와 Argo CD 설치 참고 명령은 [10-8. (참고) Argo CD on AKS](#10-8-참고-argo-cd-on-aks--preview-옵션과-비교)에서 다룹니다.

### 이 섹션에서 배우는 것

- **GitOps 개념** — 명령형(imperative) vs 선언형(declarative) 배포의 차이
- **Flux v2 확장** — AKS에서 기본 지원하는 GitOps 컨트롤러 설정
- **자동 Sync** — Git 커밋 → 30초 내 클러스터 자동 반영
- **드리프트 복구** — 수동 변경을 감지하고 Git 상태로 자동 되돌리기
- **(참고) Flux v2 vs Argo CD** — 두 매니지드 옵션의 차이점과 선택 기준

### GitOps vs 전통적 배포 비교

| 항목 | `kubectl apply` (전통적) | GitOps (Flux v2) |
|------|------------------------|------------------|
| 배포 주체 | 사람 (CLI 실행) | Flux가 자동 감지 & 배포 |
| 상태 관리 | 명령형 (imperative) | 선언형 (declarative) |
| 이력 추적 | CLI 히스토리에 의존 | Git 커밋 이력으로 추적 |
| 드리프트 감지 | 수동 확인 | 자동 감지 & 복구 |
| 롤백 | `kubectl rollout undo` | Git revert → 자동 반영 |

### 핸즈온 시나리오

```mermaid
flowchart TD
    A["Git 변경"] --> B["Flux 감지 30초"]
    B --> C["AKS 자동 Sync"]
    C --> D["롤링 업데이트"]

    style A fill:#339af0,color:#fff
    style D fill:#51cf66,color:#fff
```

---

## 10-0. 사전 준비 — 개인 GitHub 저장소 구성

GitOps는 **"Git 저장소 → 클러스터" 단방향 동기화**가 핵심입니다. Flux가 polling할 URL은 **본인이 push 권한을 가진 저장소**여야 하며, 워크샵 진행 중 매니페스트를 직접 커밋·푸시하므로 각자 **개인 fork**가 필요합니다.

> [!NOTE]
> **여러 명이 같은 repo를 공유할 수 없는 이유**: push 권한 충돌, 한 명의 변경이 모두의 클러스터에 반영됨, 드리프트 실험 격리 불가. 1인 1 fork가 필수입니다.

### Step 1: 워크샵 저장소 Fork

브라우저에서 [https://github.com/bbiggum/azure-aks-workshop](https://github.com/bbiggum/azure-aks-workshop) 접속 → 우측 상단 **Fork** 버튼 클릭 → 본인 계정으로 복제합니다.

> 📸 **스크린샷**: Fork 화면 — `Owner`는 본인 GitHub 계정, `Repository name`은 그대로 두면 됨
>
> ![GitHub Fork 화면](images/github-fork.png)

### Step 2: Cloud Shell에서 본인 fork 연결 + 로컬 main 브랜치 준비

이전 절(`01-prerequisites`)에서 이미 `bbiggum/azure-aks-workshop` 을 clone 한 상태라고 가정합니다.

> [!NOTE]
> 01절에서 `git clone -b my-change ...` 형태로 받았다면 로컬에 **`main` 브랜치가 없는 상태**입니다(my-change 브랜치만 받음). 아래 절차의 `git fetch` + `checkout -B main origin/main` 단계가 이 함정을 해결해 줍니다.

```bash
# (a) 본인 GitHub 사용자명을 환경 변수로 (이후 모든 단계에서 재사용)
export GITHUB_USERNAME="<YOUR_GITHUB_USERNAME>"

cd ~/azure-aks-workshop

# (b) 기존 origin(bbiggum)을 upstream으로 이름 변경 — 나중에 원본 변경 동기화 시 사용
git remote rename origin upstream

# (c) 본인 fork를 새 origin으로 등록
git remote add origin https://github.com/$GITHUB_USERNAME/azure-aks-workshop.git

# (d) 본인 fork의 브랜치 정보 가져오기
git fetch origin

# (e) 로컬 main 브랜치를 본인 fork의 main 기준으로 생성/체크아웃
git checkout -B main origin/main

# (f) 확인 — origin 두 줄(본인 fork), upstream 두 줄(bbiggum) 모두 보여야 정상
git remote -v
git branch
```

기대 출력:

```
origin    https://github.com/<YOUR_GITHUB_USERNAME>/azure-aks-workshop.git (fetch)
origin    https://github.com/<YOUR_GITHUB_USERNAME>/azure-aks-workshop.git (push)
upstream  https://github.com/bbiggum/azure-aks-workshop (fetch)
upstream  https://github.com/bbiggum/azure-aks-workshop (push)

* main
  my-change
```

<details>
<summary><strong>처음부터 clone하는 경우 (01절을 건너뛴 분)</strong></summary>

```bash
export GITHUB_USERNAME="<YOUR_GITHUB_USERNAME>"

# 본인 fork를 직접 clone — origin이 자동으로 본인 fork가 됨
git clone https://github.com/$GITHUB_USERNAME/azure-aks-workshop.git
cd azure-aks-workshop

# 원본 repo를 upstream으로 등록 (선택 — 동기화 필요할 때만)
git remote add upstream https://github.com/bbiggum/azure-aks-workshop.git
```

</details>

> [!TIP]
> 워크샵 진행 중 원본 repo(`bbiggum/azure-aks-workshop`)에 업데이트가 있을 때 본인 fork에 반영하려면:
> ```bash
> git fetch upstream
> git checkout main
> git merge upstream/main           # 또는 git reset --hard upstream/main (본인 변경 폐기)
> git push origin main
> ```
> 또는 GitHub UI에서 본인 fork 페이지 → **`Sync fork`** 버튼 한 번이면 끝.

### Step 3: GitHub 인증 토큰(PAT) 준비 — 선택 사항

> [!IMPORTANT]
> **Azure Cloud Shell 사용자(대부분의 워크샵 환경)는 이 단계를 건너뛰어도 됩니다.**  
> Cloud Shell에는 **Git Credential Manager (GCM)** 가 사전 설치되어 있어, 첫 `git push` 시 자동으로 브라우저 OAuth 창이 열리고 본인의 GitHub/EMU SSO 세션으로 인증이 즉시 완료됩니다. 발급된 OAuth 토큰은 Cloud Shell이 자동으로 캐시하므로 별도의 PAT 입력이 필요 없습니다.
>
> 바로 [Step 4 — 빈 커밋 push 검증](#step-4-확인--빈-커밋으로-push-권한-검증)으로 이동하세요.

다음 경우에는 PAT이 필요합니다:

- 로컬 머신(macOS/Linux/Windows)에서 GCM 없이 진행하는 경우
- CI/CD 파이프라인 같은 **비대화형 환경**
- GCM이 거부하는 특수한 프라이빗 fork

Cloud Shell에서 만약 OAuth 자동 흐름이 동작하지 않거나(드물게 발생), 위 케이스에 해당한다면 아래로 PAT을 생성하세요.

1. [https://github.com/settings/tokens](https://github.com/settings/tokens) 접속 → **Generate new token (classic)**

   ![GitHub PAT 생성 화면](images/gen-pat.png)

2. 권한(Scopes): **`repo`** 체크 (전체) — fork가 public이어도 push에는 필요
3. **Expiration**: 워크샵 기간만큼 짧게 (예: 7일) 설정 — 보안상 권장

   ![Expiration · Scopes 설정 화면](images/gen-pat2.png)

4. 생성된 토큰을 안전한 곳에 복사 (한 번만 표시됨)

   ![생성된 PAT 토큰 화면](images/gen-pat3.png)

> [!TIP]
> 첫 push 시 사용자명/비밀번호 프롬프트가 뜨면 **비밀번호 자리에 PAT을 그대로 붙여넣으세요**. Cloud Shell은 인증 정보를 세션 동안만 캐시합니다.
>
> 토큰을 자주 입력하기 싫다면 git credential helper를 잠시 켭니다:
> ```bash
> git config --global credential.helper cache
> ```

### Step 4: 확인 — 빈 커밋으로 push 권한 검증

본격적인 변경 전에 **푸시 권한이 동작하는지 한 번 확인**합니다.

```bash
# 빈 커밋 후 push
git commit --allow-empty -m "chore: verify push access for GitOps workshop"
git push origin main
```

성공 메시지가 보이고 본인 GitHub fork의 commit 이력에 새 커밋이 나타나면 준비 완료입니다. 인증 오류가 나면 Step 3을 다시 확인하세요.

### 사전 준비 체크리스트

- [ ] 본인 GitHub 계정에 `azure-aks-workshop` fork 존재
- [ ] `git remote -v` → `origin`이 본인 fork, `upstream`이 `bbiggum/azure-aks-workshop`
- [ ] `git branch` → 로컬에 `main` 브랜치 존재 (현재 브랜치이면 더 좋음)
- [ ] (선택) Cloud Shell 외 환경 / GCM 미사용 시 PAT을 만들어 안전한 곳에 저장
- [ ] `git push origin main` 빈 커밋이 성공
- [ ] 본인 GitHub 저장소 페이지에서 새 커밋 확인

---

## 10-1. AKS GitOps 확장 (Flux v2) 활성화

AKS에서는 **Flux v2** 기반의 GitOps 확장을 기본 지원합니다.  
먼저 CLI 확장을 설치하고 클러스터에 GitOps를 활성화합니다.

```bash
# k8s-configuration CLI 확장 설치
az extension add --name k8s-configuration --upgrade
```

## 10-2. GitOps 구성 생성

Git 저장소를 클러스터에 연결하는 Flux 구성을 생성합니다.

```bash
# GitOps 매니페스트 디렉터리 생성
mkdir -p gitops-manifests
```

### store-front 전용 GitOps 매니페스트 준비

간단한 시나리오로 `store-front`의 이미지 태그를 GitOps로 관리합니다.

```bash
cat > gitops-manifests/store-front-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-front
  namespace: pets
spec:
  replicas: 2
  selector:
    matchLabels:
      app: store-front
  template:
    metadata:
      labels:
        app: store-front
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-front
          image: aksworkshopkoea6e.azurecr.io/store-front:ko
          ports:
            - containerPort: 8080
          env:
            - name: VUE_APP_ORDER_SERVICE_URL
              value: "http://order-service:3000/"
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
EOF
```

### Git 저장소에 매니페스트 푸시

> [!NOTE]
> 본인 fork로 remote 변경과 PAT 준비는 [10-0. 사전 준비](#10-0-사전-준비--개인-github-저장소-구성)에서 이미 완료되었어야 합니다. 아직이라면 먼저 10-0을 진행하세요.

> [!TIP]
> 실제 GitOps 운영에서는 애플리케이션 코드와 배포 매니페스트를 분리된 저장소로 관리하는 것이 모범 사례입니다.

```bash
# 워크샵에서는 현재 저장소의 gitops-manifests 디렉터리를 활용합니다
cd ~/azure-aks-workshop
git add gitops-manifests/
git commit -m "feat: add GitOps manifests for store-front"
git push origin main
```

## 10-3. Flux GitOps 구성 적용

AKS에 Flux 구성을 생성하여 Git 저장소를 연결합니다.

```bash
az k8s-configuration flux create \
  --name workshop-gitops \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --cluster-type managedClusters \
  --namespace flux-system \
  --scope cluster \
  --url https://github.com/$GITHUB_USERNAME/azure-aks-workshop \
  --branch main \
  --kustomization name=store-front path=./gitops-manifests prune=true sync_interval=30s
```

> [!NOTE]
> `$GITHUB_USERNAME` 변수는 [10-0 Step 2](#step-2-cloud-shell에서-본인-fork로-remote-변경)에서 설정한 것입니다. 변수가 비어 있다면 `export GITHUB_USERNAME="<본인 사용자명>"` 으로 다시 설정하세요.

> **프라이빗 저장소**인 경우 `--https-user`와 `--https-key` 옵션을 추가하세요:
> ```bash
> az k8s-configuration flux create \
>   ... \
>   --https-user <GITHUB_USERNAME> \
>   --https-key <GITHUB_PAT>
> ```

### 구성 상태 확인

```bash
az k8s-configuration flux show \
  --name workshop-gitops \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --cluster-type managedClusters \
  -o table
```

### Flux 컨트롤러 확인

```bash
# Flux 컴포넌트 확인
kubectl get pods -n flux-system
```

### 예상 출력

```
NAME                                       READY   STATUS    RESTARTS   AGE
fluxconfig-agent-6b8688c7d5-c4mmh          2/2     Running   0          110m
fluxconfig-controller-5f7b4878f4-r2scc     2/2     Running   0          110m
helm-controller-5579b4b4b6-ws6jd           1/1     Running   0          110m
kustomize-controller-568bcf7b57-6ssc9      1/1     Running   0          110m
notification-controller-66b7bfb489-2hdjw   1/1     Running   0          110m
source-controller-54f4cc9f5b-rj8lw         1/1     Running   1          110m
```

### Azure Portal에서 확인

CLI 외에 **Azure Portal**에서도 동일한 정보를 GUI로 볼 수 있습니다.

**Azure Portal → AKS 클러스터(`workshop-demo`) → 좌측 메뉴 `GitOps`** 에서 방금 만든 `workshop-gitops` 구성과 Sync 상태, 소스 저장소, Kustomization 목록 등을 확인합니다.

![AKS Portal GitOps 화면](images/flux-ui.png)

> [!NOTE]
> AKS Portal의 GitOps 블레이드는 **상태 조회 + 구성 편집/삭제** 위주이고, "지금 즉시 Sync" 버튼은 제공하지 않습니다. 즉시 reconcile이 필요하면 다음을 사용하세요:
> ```bash
> # 방법 1 — kubectl annotate (가장 호환성 좋음)
> kubectl annotate gitrepositories.source.toolkit.fluxcd.io -n flux-system workshop-gitops \
>   reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
>
> # 방법 2 — Flux CLI (curl -s https://fluxcd.io/install.sh | sudo bash 로 설치 후)
> flux reconcile source git workshop-gitops -n flux-system
> ```
> 그 외에는 기본 `sync_interval=30s` 가 알아서 처리하므로 보통은 그냥 기다리면 됩니다.

## 10-4. GitOps Sync 확인

Flux가 Git 저장소의 매니페스트를 클러스터에 동기화하는지 확인합니다.

```bash
# Kustomization 상태 확인
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -n flux-system
```

```
NAME                          AGE   READY     STATUS
workshop-gitops-store-front   14h   Unknown   Reconciliation in progress
```


```bash
# store-front Deployment 확인
kubectl get deployment store-front -n pets -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```

## 10-5. GitOps 워크플로우 체험 — replicas 변경

Git에서 매니페스트를 수정하면 Flux가 자동으로 클러스터에 반영합니다. 이 실습에서는 `store-front` 의 `replicas` 값을 변경하고 자동 sync 과정을 관찰합니다.

> [!WARNING]
> **07절에서 만든 HPA(`store-front-hpa`, min=2/max=10)와 충돌 주의.**  
> `store-front` Deployment의 `replicas` 필드를 Git에서 바꿔도, HPA가 CPU 사용률 기반으로 즉시 다시 자기가 원하는 값으로 덮어씁니다. virtual-customer가 부하를 생성 중이라면 replicas를 줄여도 HPA가 다시 늘립니다.
>
> **이 실습 직전에 HPA를 잠시 비활성화하세요:**
> ```bash
> kubectl delete hpa store-front-hpa -n pets
> ```
> 실습 종료 후 복원:
> ```bash
> kubectl apply -f workshop-manifests/55-hpa-store.yaml
> ```
>
> 프로덕션 베스트 프랙티스로는 HPA가 관리하는 Deployment의 `replicas` 필드는 GitOps에서 제외(또는 ignore patch)하는 게 일반적입니다.

### Step 1: 매니페스트에서 replicas 수 변경

```bash
# replicas를 2 → 4로 변경
sed -i 's/replicas: 2/replicas: 4/' gitops-manifests/store-front-deployment.yaml
```

### Step 2: Git에 커밋 & 푸시

```bash
cd ~/azure-aks-workshop
git add gitops-manifests/store-front-deployment.yaml
git commit -m "scale: store-front replicas 2 → 4"
git push origin main
```

### Step 3: 자동 Sync 관찰

```bash
# Flux가 변경을 감지하고 적용하는 과정 관찰 (30초 내 반영)
kubectl get pods -n pets -l app=store-front -w
```

> [!NOTE]
> ⏱ `sync_interval=30s`로 설정했으므로 최대 30초 내에 변경이 반영됩니다. 즉시 보고 싶다면 [10-3 NOTE](#azure-portal에서-확인)에 있는 `kubectl annotate gitrepositories ...` 명령으로 강제 reconcile.

### 예상 결과

```
NAME                          READY   STATUS    RESTARTS   AGE
store-front-88f94db58-rgnsv   1/1     Running   0          12m
store-front-88f94db58-spbqd   1/1     Running   0          84s    ← 새로 추가
store-front-88f94db58-vx6zr   1/1     Running   0          84s    ← 새로 추가
store-front-88f94db58-z76q2   1/1     Running   0          12m
```

### (선택) 역방향 — 4 → 2 로 줄이기

```bash
sed -i 's/replicas: 4/replicas: 2/' gitops-manifests/store-front-deployment.yaml
git commit -am "scale: store-front replicas 4 → 2"
git push origin main
```

30초 후 Pod이 4개 → 2개로 줄어듭니다. HPA를 다시 활성화하기 전까지는 이 값이 유지됩니다.

> [!TIP]
> **이미지 태그 변경(예: `:latest` → `:v2`) 데모를 보고 싶다면**, 매니페스트의 `image:` 필드를 sed로 바꾸고 동일한 add/commit/push 흐름을 따르면 됩니다. 그 경우 Pod이 rolling update로 한 번에 1~2개씩 교체되는 모습을 볼 수 있고, HPA와 충돌하지 않습니다.

## 10-6. 드리프트 감지 & 자동 복구

GitOps의 핵심 장점 중 하나는 **드리프트(drift) 자동 복구**입니다.  
누군가 `kubectl`로 직접 변경해도 Flux가 Git 상태로 자동 되돌립니다.

### 실험: 수동 변경 후 복구 관찰

```bash
# 수동으로 replicas를 1로 축소
kubectl scale deployment/store-front -n pets --replicas=1

# Pod 수 확인 (일시적으로 1개)
kubectl get pods -n pets -l app=store-front

# 30초 후 Flux가 자동으로 4개로 복구
kubectl get pods -n pets -l app=store-front -w
```

> Flux가 Git 저장소의 `replicas: 4`와 클러스터 상태가 다른 것을 감지하여 자동으로 복구합니다.

## 10-7. (선택) 정리

```bash
# Flux GitOps 구성 삭제
az k8s-configuration flux delete \
  --name workshop-gitops \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --cluster-type managedClusters \
  --yes

# replicas를 원래 값으로 복구
sed -i 's/replicas: 4/replicas: 2/' gitops-manifests/store-front-deployment.yaml
kubectl apply -f workshop-manifests/aks-store-all-in-one-ko.yaml
```

## 10-8. (참고) Argo CD on AKS — Preview 옵션과 비교

이 워크샵은 **Flux v2**(AKS 매니지드 GA)를 사용했지만, GitOps의 또 다른 양대 산맥인 **Argo CD**도 AKS의 매니지드 확장으로 **Preview** 단계에 와 있습니다. 실무에서 두 도구가 어떻게 다른지 알아두면 도구 선택에 도움이 됩니다.

### AKS 매니지드 형태 비교

| 항목 | Flux v2 (이 워크샵에서 사용) | Argo CD |
|---|---|---|
| **AKS 매니지드 GA 상태** | ✅ **GA** | 🟡 **Preview** (`Microsoft.ContainerService/ArgoCDPreview` feature) |
| **Azure 리소스 종류** | `Microsoft.KubernetesConfiguration/fluxConfigurations` | `Microsoft.KubernetesConfiguration/extensions` (type: `Microsoft.ArgoCD`) |
| **설치 명령** | `az k8s-configuration flux create ...` | `az k8s-extension create --extension-type Microsoft.ArgoCD ...` |
| **Portal 통합** | AKS → 좌측 메뉴 `GitOps` (전용 블레이드) | 클러스터에 Argo CD UI를 LoadBalancer/Ingress로 노출 |
| **Azure Policy 통합** | ✅ 정책으로 GitOps 구성 강제 가능 | 🟡 Preview 단계 |
| **Microsoft 공식 지원 (SR)** | ✅ | 🟡 Preview는 best-effort |
| **워크샵 적합도** | ✅ 5~10분에 데모 완료 | UI/SSO 설정 시간 더 필요 |

### 도구 자체 기능 비교 (매니지드 여부와 무관)

| 항목 | Flux v2 | Argo CD |
|---|---|---|
| **UI** | ❌ 없음 (CLI/대시보드 별도 — `flux` CLI, Weave GitOps Dashboard) | ✅ **강력한 웹 UI** (시각적 Sync 트리, Diff, Rollback) |
| **멀티 클러스터** | 클러스터마다 Flux 설치 | **중앙 Argo CD 1개로 여러 클러스터 관리** (Hub-Spoke) |
| **앱 모델** | `Kustomization`, `HelmRelease` (선언적) | `Application` CRD (선언적 + UI에서 조작 가능) |
| **이미지 자동 업데이트** | ✅ `ImageUpdateAutomation` 내장 | ❌ `ArgoCD Image Updater` 별도 |
| **알림 통합** | `Notification Controller` (네이티브, Slack/Teams/PagerDuty 등) | `ArgoCD Notifications` |
| **러닝 커브** | 낮음 (CRD 적음, 명령형 흐름) | 약간 높음 (Application, AppProject, RBAC 개념) |
| **운영 부담** | 적음 (Controller 5개, 모두 stateless) | 보통 (server, repo-server, redis, dex 등 컴포넌트 다수) |
| **CNCF 상태** | Graduated (2022) | Graduated (2022) |
| **사용자 규모** | 더 적음 | 더 많음 (특히 대기업 DevOps팀에서 사실상 표준) |
| **AKS Fleet Manager 통합** | 가능 | **특히 강점** — 수십~수백 AKS를 중앙 Argo CD로 관리 |

### 언제 어느 쪽을 선택?

**Flux v2가 유리한 경우**
- AKS 매니지드 확장으로 GA 안정성을 원할 때
- **단일 클러스터 + 단순한 GitOps 흐름**
- 운영팀이 적고 추가 컴포넌트를 두기 부담스러울 때
- 이미지 태그 자동 업데이트가 핵심 요구사항
- Azure Policy로 GitOps 구성을 정책화하고 싶을 때

**Argo CD가 유리한 경우**
- **다수 클러스터를 중앙 1개에서 시각적으로 관리** 하고 싶을 때 (AKS Fleet Manager와 조합)
- 개발자에게 **UI로 sync/diff/rollback 권한**을 주고 싶을 때
- **App-of-Apps, ApplicationSet** 같은 고급 패턴이 필요할 때
- 이미 ArgoCD가 사내 표준일 때 (생태계 도구가 더 많음)

### Argo CD on AKS 설치 (참고용 — 실습 X)

> [!IMPORTANT]
> 아래 명령은 **참고용**입니다. Preview feature 등록 + AKS 확장 설치 + UI 노출 + SSO 연동 등 단계가 많아 워크샵에선 실행하지 않습니다.

```bash
# 1) Preview feature 등록 (구독당 1회)
az feature register --namespace Microsoft.ContainerService --name ArgoCDPreview
az provider register --namespace Microsoft.ContainerService

# 2) AKS에 Argo CD 매니지드 확장 설치
az k8s-extension create \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --cluster-type managedClusters \
  --name argocd \
  --extension-type Microsoft.ArgoCD \
  --auto-upgrade true

# 3) Argo CD 서버 Pod 확인 (argocd 네임스페이스)
kubectl get pods -n argocd

# 4) UI 접근 — Port-forward (실무에선 Ingress + AAD SSO 권장)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 5) 초기 admin 비밀번호
kubectl get secret -n argocd argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d ; echo
```

> Argo CD 정식 도입 시에는 [공식 가이드](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/) 의 `Application`/`AppProject` CRD 작성 방식, RBAC, SSO 연동을 함께 학습하세요.

### 한 줄 요약

> **이 워크샵에서 Flux를 택한 건 "GA + 빠른 데모" 때문이고, ArgoCD가 더 나쁜 도구라서가 아닙니다. 실무에선 클러스터 수, UI 요구, 팀 구조에 따라 둘 다 정답이 될 수 있습니다.**

## 핵심 개념 정리

```mermaid
flowchart TD
    A["Git Push"] --> B["Flux 감지"]
    B --> C["클러스터 Apply"]
    C --> D["동기화 완료 ✅"]
    D --> E["kubectl 수동 변경"]
    E --> F["Flux 자동 복구"]
    F --> D

    style A fill:#339af0,color:#fff
    style D fill:#51cf66,color:#fff
    style E fill:#ff6b6b,color:#fff
    style F fill:#fcc419,color:#333
```

## 점검 체크리스트

- [ ] `kubectl get pods -n flux-system` — Flux 컨트롤러 3개 Running
- [ ] `kubectl get kustomizations -n flux-system` — Ready=True
- [ ] Git에서 replicas 변경 → 30초 내 클러스터 반영 확인
- [ ] `kubectl scale` 수동 변경 → Flux가 자동 복구하는지 확인

---

| | |
|:---|---:|
| [⬅️ 09. 모니터링 & 트러블슈팅](09-monitoring-troubleshooting.md) | [11. 정리 ➡️](11-cleanup.md) |
