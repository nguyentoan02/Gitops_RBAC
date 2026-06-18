# Plan Afternoon - Part 1 Runbook

## Muc tieu

Hoan thanh phan 1 cua bai lab:

- Secret DB khong nam trong Git
- Secret duoc lay tu AWS Secrets Manager qua External Secrets Operator
- App doc secret qua mounted volume `/secrets/password`
- Khi secret doi tren AWS, Kubernetes secret `db-secret` cap nhat lai trong `<= 60s`
- App thay duoc secret moi ma khong can sua code hay hard-code password

## Nhung gi da duoc sua trong repo

### File moi

- `k8s-eso/secret-store.yaml`
- `k8s-eso/external-secret.yaml`
- `argocd/apps/eso.yaml`

### File da sua

- `src/api/app.py`
- `app-api/rollout.yaml`
- `.github/workflows/validate.yml`

## Logic da implement

### 1. App da doc DB secret tu file

Trong `src/api/app.py`:

- Them env `DB_PASSWORD_PATH`, mac dinh `/secrets/password`
- Route `/` tra ve:
  - `db_status`
  - `db_password_loaded`
- Them route debug `/db-secret`
- App khong crash neu secret chua co; no se tra `SECRET_NOT_FOUND`

### 2. Rollout da mount secret vao pod

Trong `app-api/rollout.yaml`:

- Mount secret `db-secret` vao thu muc `/secrets`
- Set env `DB_PASSWORD_PATH=/secrets/password`
- Secret volume dang de `optional: true`

Ly do de `optional: true`:

- Pod van len duoc ngay ca khi `ExternalSecret` chua sync xong
- Sau khi ESO tao `db-secret`, file secret se xuat hien trong volume

### 3. GitOps manifests da co cho ESO objects

Trong `k8s-eso/`:

- `secret-store.yaml`: noi AWS Secrets Manager qua secret `aws-credentials`
- `external-secret.yaml`: dong bo AWS secret `prod/db/password` thanh K8s secret `db-secret`

Trong `argocd/apps/eso.yaml`:

- ArgoCD se sync thu muc `k8s-eso/` vao namespace `demo`

## Dieu kien truoc khi chay

Ban can co:

- Cluster dang chay
- `kubectl` dang tro dung context cluster
- ArgoCD da cai va `argocd/root.yaml` da duoc apply
- External Secrets Operator controller da duoc cai vao cluster
- AWS account/co quyen tao va doc secret trong Secrets Manager
- Image local `gitops-rbac-api:0.0.1` build duoc va load vao cluster local

## Luu y ve AWS tren may nay

Khi toi kiem tra:

```powershell
aws sts get-caller-identity
```

lenh dang fail voi loi proxy:

```text
Failed to connect to proxy URL: "http://127.0.0.1:9"
```

Truoc khi chay cac lenh AWS, ban can:

- xoa proxy sai
- hoac cau hinh proxy dung
- hoac provide lai AWS environment/profile hop le

Kiem tra nhanh:

```powershell
Get-ChildItem Env:HTTP_PROXY
Get-ChildItem Env:HTTPS_PROXY
Get-ChildItem Env:AWS_PROFILE
aws configure list
aws sts get-caller-identity
```

Neu `HTTP_PROXY`/`HTTPS_PROXY` dang tro toi `127.0.0.1:9`, bo no di:

```powershell
Remove-Item Env:HTTP_PROXY -ErrorAction SilentlyContinue
Remove-Item Env:HTTPS_PROXY -ErrorAction SilentlyContinue
```

## B1 - Cai External Secrets Operator controller

Phan nay la dependency bat buoc. Repo hien da co `SecretStore` va `ExternalSecret`, nhung neu cluster chua co controller thi se khong reconcile duoc.

Co the cai nhanh bang manifest:

```powershell
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.5/deploy/crds/bundle.yaml
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/v0.10.5/deploy/operator.yaml
```

Sau do verify:

```powershell
kubectl get pods -n external-secrets
kubectl get crd | findstr external-secrets
```

Pass neu:

- co namespace/controller cua `external-secrets`
- co CRD `secretstores.external-secrets.io`
- co CRD `externalsecrets.external-secrets.io`

## B2 - Tao AWS secret tren Secrets Manager

Secret ma manifests dang dung la:

- Name: `prod/db/password`
- Region: `ap-southeast-1`

Tao moi:

```powershell
aws secretsmanager create-secret `
  --name prod/db/password `
  --secret-string "MyS3cr3tP@ss" `
  --region ap-southeast-1
```

Neu secret da ton tai, update:

```powershell
aws secretsmanager update-secret `
  --secret-id prod/db/password `
  --secret-string "MyS3cr3tP@ss" `
  --region ap-southeast-1
```

Verify:

```powershell
aws secretsmanager get-secret-value `
  --secret-id prod/db/password `
  --region ap-southeast-1 `
  --query SecretString `
  --output text
```

## B3 - Tao K8s secret chua AWS credentials

`k8s-eso/secret-store.yaml` dang doi mot secret Kubernetes ten `aws-credentials` trong namespace `demo`.

Tao secret:

```powershell
kubectl create secret generic aws-credentials `
  --namespace demo `
  --from-literal=access-key-id=YOUR_AWS_ACCESS_KEY_ID `
  --from-literal=secret-access-key=YOUR_AWS_SECRET_ACCESS_KEY
```

Verify:

```powershell
kubectl get secret aws-credentials -n demo
```

Quan trong:

- Khong commit access key vao Git
- Access key can co quyen doc `prod/db/password`

## B4 - Build image local de chay app moi

Repo dang dung image local trong `app-api/rollout.yaml`, nen can build lai image de co code doc secret moi.

```powershell
docker build -t gitops-rbac-api:0.0.1 src/api
minikube image load gitops-rbac-api:0.0.1 -p w10
```

Neu khong dung minikube profile `w10`, doi lai ten profile cho dung.

## B5 - Dua manifests moi vao cluster

### Cach 1 - Neu dang dung ArgoCD App of Apps

Commit/push cac file moi:

```powershell
git add src/api/app.py app-api/rollout.yaml k8s-eso argocd/apps/eso.yaml .github/workflows/validate.yml plan_afternoon.md
git commit -m "feat: add ESO secret sync for DB password"
git push origin master
```

Sau do verify app `eso` duoc root app nhat vao:

```powershell
kubectl get applications -n argocd
argocd app sync eso
argocd app wait eso --health --sync
```

### Cach 2 - Neu ban muon apply tay de test nhanh truoc

```powershell
kubectl apply -f k8s-eso/secret-store.yaml
kubectl apply -f k8s-eso/external-secret.yaml
kubectl apply -f app-api/service.yaml
kubectl apply -f app-api/servicemonitor.yaml
kubectl apply -f app-api/rollout.yaml
```

Neu namespace `demo` chua co:

```powershell
kubectl apply -f app-common/demo-namespace.yaml
```

## B6 - Verify ESO da sync secret

Kiem tra cac object:

```powershell
kubectl get secretstore -n demo
kubectl get externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl describe externalsecret db-password -n demo
```

Doc password tu K8s secret:

```powershell
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}"
```

Gia tri tra ve la base64. Tren PowerShell co the decode:

```powershell
$b64 = kubectl get secret db-secret -n demo -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b64))
```

Pass neu:

- `SecretStore` ton tai
- `ExternalSecret` co `READY=True`
- `db-secret` da duoc tao
- Password giai ma ra la `MyS3cr3tP@ss`

## B7 - Verify app da doc duoc secret

Kiem tra pod:

```powershell
kubectl get pods -n demo -l app=api
```

Port-forward service:

```powershell
kubectl port-forward -n demo svc/api 8080:80
```

Trong terminal khac:

```powershell
curl http://127.0.0.1:8080/
curl http://127.0.0.1:8080/db-secret
```

Ket qua mong doi:

```json
{
  "ok": true,
  "version": "v0.0.1",
  "db_status": "connected",
  "db_password_loaded": true
}
```

Va:

```json
{
  "password_path": "/secrets/password",
  "password_found": true,
  "password_preview": "MyS3c..."
}
```

## B8 - Test secret rotation <= 60s

Xem gia tri hien tai:

```powershell
$b64 = kubectl get secret db-secret -n demo -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b64))
```

Cap nhat secret tren AWS:

```powershell
aws secretsmanager update-secret `
  --secret-id prod/db/password `
  --secret-string "NewP@ss123" `
  --region ap-southeast-1
```

Cho khoang 60 giay vi `refreshInterval: 1m`:

```powershell
Start-Sleep -Seconds 70
```

Doc lai secret tren cluster:

```powershell
$b64 = kubectl get secret db-secret -n demo -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b64))
```

Kiem tra app:

```powershell
curl http://127.0.0.1:8080/db-secret
curl http://127.0.0.1:8080/
```

Pass neu:

- Kubernetes secret doi tu `MyS3cr3tP@ss` sang `NewP@ss123`
- `/db-secret` van cho `password_found=true`
- `/` van cho `db_status=connected`
- khong can doi lai manifest
- khong can hard-code password vao repo

## B9 - Troubleshooting

### 1. `ExternalSecret` khong READY

Kiem tra:

```powershell
kubectl describe externalsecret db-password -n demo
kubectl logs -n external-secrets deploy/external-secrets
```

Thuong do:

- sai AWS credentials
- sai region
- secret `prod/db/password` chua ton tai
- cluster chua cai ESO controller

### 2. Pod app len nhung `password_found=false`

Kiem tra:

```powershell
kubectl exec -n demo deploy/api -- ls /secrets
kubectl get secret db-secret -n demo -o yaml
```

Thuong do:

- `db-secret` chua duoc tao
- key trong secret khong phai `password`

### 3. AWS CLI van loi proxy

Kiem tra:

```powershell
Get-ChildItem Env:HTTP_PROXY
Get-ChildItem Env:HTTPS_PROXY
```

Neu can, xoa env proxy roi chay lai `aws sts get-caller-identity`.

## Tieu chi pass phan 1

Pass khi tat ca dieu kien sau dung:

- ESO controller dang chay trong cluster
- `aws-credentials` ton tai trong namespace `demo`
- AWS Secrets Manager co secret `prod/db/password`
- `SecretStore` + `ExternalSecret` deploy thanh cong
- `db-secret` duoc tao trong namespace `demo`
- API doc duoc secret qua file `/secrets/password`
- Khi AWS secret doi, cluster secret cap nhat trong `<= 60s`
- App van truy cap duoc secret moi ma khong hard-code password vao Git

## Nhung gi toi chua the tu chay het tren may nay

Toi da hoan thanh phan code/manifests cho part 1, nhung chua the chay end-to-end vi hien con thieu hoac dang bi chan:

- AWS CLI hien fail do proxy `127.0.0.1:9`
- Toi chua co gia tri that cua:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
- Toi chua xac nhan duoc cluster cua ban dang chay va da co ESO controller hay chua

## Neu ban muon toi chay tiep den muc verify that

Ban can cung cap mot trong cac nhom thong tin sau:

1. AWS access key va secret key co quyen Secrets Manager, hoac profile AWS da config san va dung duoc
2. Xac nhan cluster dang dung la gi:
   - minikube
   - Docker Desktop Kubernetes
   - cluster khac
3. Xac nhan da cai hay chua cai External Secrets Operator
4. Neu may dang dung proxy, cho toi biet cach AWS CLI can duoc cau hinh de ra ngoai
