# Plan chay va kiem thu 1 kich ban

## Muc tieu

Chay full demo GitOps/Kubernetes va chi kiem thu 1 kich ban:

- deploy thanh cong
- `ERROR_RATE = "0"`
- rollout di het 10% -> 50% -> 100%
- `AnalysisRun` pass

Toi chon kich ban nay vi no it rui ro nhat, de xac nhan toan bo duong chay co ban da dung:

- Argo CD sync duoc repo
- Argo Rollouts hoat dong
- Prometheus scrape duoc metrics
- AnalysisTemplate danh gia thanh cong

## Dieu kien dau vao

Can co san:

- Docker Desktop
- `minikube`
- `kubectl`
- `git`
- mang ra ngoai de keo chart/image/repo

Repo da duoc chot:

- `https://github.com/nguyentoan02/Gitops_RBAC.git`
- branch `master`

Email alert da cau hinh local:

- `app-alert/email-secret.yaml`

## Kich ban duy nhat can chay

### Buoc 1 - Tao cluster

```bash
minikube start -p w10 --driver=docker
kubectl config use-context w10
kubectl get nodes
```

Ket qua mong doi:

- cluster `w10` len thanh cong
- `kubectl get nodes` tra ve node `Ready`

### Buoc 2 - Cai Argo CD

```bash
kubectl create ns argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

Ket qua mong doi:

- `argocd-server` chay `successfully rolled out`

### Buoc 3 - Deploy App of Apps

```bash
kubectl apply -f argocd/root.yaml
```

Ket qua mong doi:

- cac app `common`, `kube-prometheus-stack`, `argo-rollouts`, `analysis`, `alert`, `api` duoc tao

Theo doi:

```bash
kubectl get applications -n argocd
kubectl get pods -n argo-rollouts
kubectl get pods -n monitoring
kubectl get pods -n demo
```

### Buoc 4 - Apply secret email

```bash
kubectl apply -f app-alert/email-secret.yaml
```

Neu namespace `monitoring` chua ton tai, cho app `kube-prometheus-stack` sync xong roi moi apply lai lenh tren.

Ket qua mong doi:

- secret `alertmanager-email` duoc tao trong namespace `monitoring`

### Buoc 5 - Xac nhan rollout thanh cong

File [app-api/rollout.yaml](D:\W10\temp\app-api\rollout.yaml:1) hien dang de:

- `ERROR_RATE = "0"`

Kiem tra rollout va analysis:

```bash
kubectl get rollout api -n demo
kubectl get rollout api -n demo -w
kubectl get analysisrun -n demo
kubectl get pods -n demo -l app=api
```

Ket qua mong doi:

- rollout tang qua cac step canary
- `AnalysisRun` co trang thai thanh cong
- rollout dat trang thai `Healthy`
- pod API chay on dinh

### Buoc 6 - Kiem tra metrics

```bash
kubectl run test-query --image=curlimages/curl:latest --rm -i --restart=Never -n monitoring -- curl -s "http://kube-prometheus-stack-prometheus.monitoring.svc:9090/api/v1/query?query=api:success_rate:5m"
```

Ket qua mong doi:

- query tra ve gia tri gan `1`

## Tieu chi pass

Buoi chay duoc coi la pass khi:

1. Argo CD tao day du child applications
2. Rollout `api` len thanh cong trong namespace `demo`
3. `AnalysisRun` pass
4. Prometheus query `api:success_rate:5m` tra ve ket qua hop le

## Neu co loi, check theo thu tu nay

1. `kubectl get applications -n argocd`
2. `kubectl get pods -n argocd`
3. `kubectl get pods -n argo-rollouts`
4. `kubectl get pods -n monitoring`
5. `kubectl get pods -n demo`
6. `kubectl describe rollout api -n demo`
7. `kubectl describe analysisrun -n demo <ten-analysisrun>`

## Ket luan

Khong can chay nhieu kich ban luc nay. Chi can chay 1 kich ban thanh cong voi `ERROR_RATE = "0"` de xac nhan duong nen cua he thong da dung. Sau khi kich ban nay pass, moi nen chuyen sang test rollback hoac alert SLO.
