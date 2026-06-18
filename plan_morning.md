# Plan W10 Morning: RBAC + Admission Policy

## Muc tieu tong

Hoan thanh 3 phan bat buoc cua lab:

1. RBAC: tao 3 role cho `alice`, `bob`, `carol` qua GitOps.
2. Gatekeeper: cai OPA Gatekeeper va enforce 4 constraint chan manifest xau.
3. Custom policy: tu viet 1 `ConstraintTemplate` + 1 `Constraint` bang Rego.

Tat ca phai di qua git va ArgoCD, khong `kubectl apply` tay de nop bai.

## Buoc 0: Chuan bi

- Fork repo platform.
- Sua `argocd/root.yaml` de `repoURL` tro ve repo fork cua ban.
- Kiem tra cac file trong `argocd/apps/`:
  - Moi `Application` deu co `spec.source.repoURL` rieng.
  - Neu ban fork repo thi cac child app nay cung phai tro ve repo fork cua ban, neu khong ArgoCD van lay manifest tu repo cu.
- Xac nhan platform W9 dang `Synced/Healthy` truoc khi them RBAC va Gatekeeper.

## Phan 1: RBAC qua GitOps

### Muc tieu can dat

- `alice` duoc CRUD workload chi trong namespace `demo`.
- `bob` duoc xem va thao tac pod toan cum.
- `carol` chi duoc doc toan cum.
- Moi thay doi di qua git + ArgoCD, khong apply tay.

### File can tao / sua

- Tao thu muc `rbac/`
- Tao `rbac/roles.yaml`
- Tao `rbac/rolebindings.yaml`
- Tao `argocd/apps/rbac.yaml`

### Cach lam tung buoc

#### Buoc 1: Xac dinh loai role can dung

- `alice` chi song trong `demo` nen dung `Role` namespaced.
- `bob` va `carol` la quyen toan cum nen dung `ClusterRole`.
- User that su duoc gan quyen thong qua `RoleBinding` hoac `ClusterRoleBinding`.

#### Buoc 2: Tao `rbac/roles.yaml`

Trong file nay khai bao 3 role:

- `Role` ten `developer` trong namespace `demo`
  - Cho phep `get/list/watch/create/update/patch/delete`
  - Tren:
    - `pods`
    - `services`
    - `deployments.apps`
- `ClusterRole` ten `sre`
  - Cho phep thao tac tren `pods` toan cum
  - Quyen dung:
    - `get/list/watch/create/update/patch/delete`
- `ClusterRole` ten `viewer`
  - Chi doc toan cum
  - Quyen dung:
    - `get/list/watch`
  - Co the cap cho:
    - `pods`
    - `services`
    - `namespaces`
    - `deployments`
    - `replicasets`

#### Buoc 3: Tao `rbac/rolebindings.yaml`

Trong file nay khai bao 3 binding:

- `RoleBinding` ten `alice-developer` trong namespace `demo`
  - `subjects.kind: User`
  - `subjects.name: alice`
  - `roleRef` tro toi `Role/developer`
- `ClusterRoleBinding` ten `bob-sre`
  - bind user `bob` vao `ClusterRole/sre`
- `ClusterRoleBinding` ten `carol-viewer`
  - bind user `carol` vao `ClusterRole/viewer`

#### Buoc 4: Tao `argocd/apps/rbac.yaml`

Tao 1 child app de ArgoCD doc thu muc `rbac/`:

- `metadata.name: rbac`
- `spec.source.path: rbac`
- `spec.source.repoURL`: trung voi repo dang dung
- `destination.namespace: demo`
- bat `automated.prune` va `selfHeal`

Neu ban dang lam tren repo fork, file nay cung phai tro dung `repoURL` cua repo fork.

#### Buoc 5: Commit va push

- `git add rbac argocd/apps/rbac.yaml plan_morning.md`
- `git commit -m "Add RBAC lab manifests"`
- `git push`

#### Buoc 6: ArgoCD sync

- Mo ArgoCD va kiem tra app `root`
- Xac nhan child app `rbac` duoc tao
- Cho `rbac` ve trang thai `Synced/Healthy`

Neu `rbac` khong xuat hien:

- Kiem tra `root` da sync lai chua
- Kiem tra `argocd/apps/rbac.yaml` da nam trong branch dung chua
- Kiem tra `repoURL` co dang tro sai repo khong

#### Buoc 7: Test quyen bang impersonation

Sau khi ArgoCD sync xong, chay 4 lenh `kubectl auth can-i` de xac nhan authorization.

### Luu y ky thuat

- `alice` bi gioi han trong 1 namespace nen dung `Role`.
- `bob` va `carol` la pham vi toan cum nen dung `ClusterRole`.
- `viewer` tuyet doi khong duoc co `create/update/delete`.
- Binding cho user thi dung `subjects.kind: User`.
- `kubectl auth can-i --as <user>` chi kiem authorization, khong can authentication that.
- Trong repo nay cac app ArgoCD dang hardcode `repoURL`, day la diem de sai nhat khi fork repo.

### Nghiem thu RBAC

4 lenh sau phai cho ket qua dung:

```bash
kubectl auth can-i create deploy -n demo --as alice
# yes

kubectl auth can-i create deploy -n kube-system --as alice
# no

kubectl auth can-i get pods -A --as bob
# yes

kubectl auth can-i delete nodes --as carol
# no
```

### Ket qua hien tai da duoc tao trong repo nay

- `rbac/roles.yaml`
- `rbac/rolebindings.yaml`
- `argocd/apps/rbac.yaml`

Khi review lai sau nay, chi can doi chieu 3 file tren voi muc tieu role va 4 lenh nghiem thu la du.

## Phan 2: Gatekeeper + 4 constraint

### Viec can lam

- Cai Gatekeeper controller qua GitOps:
  - Tao `argocd/apps/gatekeeper.yaml`.
  - Dat sync-wave som de controller len truoc.
- Su dung `ConstraintTemplate` tu `gatekeeper-library`, khong can tu viet Rego cho 4 luat nay.
- Tao thu muc `gatekeeper/constraints/`.
- Dat trong do:
  - cac `ConstraintTemplate` can dung,
  - 4 `Constraint`,
  - 1 app/manifests neu repo cua ban can de ArgoCD quan ly phan nay.
- Dam bao thu tu ap dung:
  1. controller
  2. `ConstraintTemplate`
  3. `Constraint`

### 4 luat bat buoc enforce

1. Cam image tag `:latest`.
2. Bat buoc co `resources.limits`.
3. Cam `runAsUser: 0`.
4. Cam `hostNetwork: true`.

### Cach lam an toan

- Ban dau dat `enforcementAction: warn` de audit.
- Kiem tra resource hien tai cua platform co dang vi pham khong.
- Dac biet kiem tra rollout `api`:
  - image da pin version,
  - co `resources.limits`,
  - khong `runAsUser: 0`.
- Sua workload cua platform neu can.
- Sau khi sach loi moi chuyen sang `enforcementAction: deny`.

### Nghiem thu Gatekeeper

Thu deploy cac manifest test, ket qua phai nhu sau:

- Pod dung image `:latest` -> reject
- Pod thieu `resources.limits` -> reject
- Pod co `runAsUser: 0` -> reject
- Pod co `hostNetwork: true` -> reject
- Pod hop le (version pinned + limits + non-root) -> pass

## Phan 3: Custom policy bat buoc

### Chon 1 de bai

- Reject `Deployment` neu `replicas > 5`
- Bat buoc moi workload co label `owner`
- Chi cho image tu registry cua ban, vi du `ghcr.io/<ban>/...`

### Viec can lam

- Tu viet 1 `ConstraintTemplate` bang Rego.
- Tao 1 `Constraint` dung template do.
- Dua vao repo de ArgoCD sync.
- Tao 1 manifest vi pham de test reject.
- Tao 1 manifest hop le de test pass.
- Commit va push day du.

## Deliverable can co trong repo

```text
rbac/                   # 3 role + 3 binding
gatekeeper/constraints/ # 4 constraint + custom policy
argocd/apps/*.yaml      # app cho rbac va gatekeeper
```

## Checklist nop bai

- [ ] `argocd/root.yaml` da tro ve repo fork cua minh
- [ ] Platform W9 van `Synced/Healthy`
- [ ] Da tao du 3 role va 3 binding qua GitOps
- [ ] 4 lenh `kubectl auth can-i` cho ket qua dung
- [ ] Gatekeeper da cai qua ArgoCD
- [ ] 4 constraint bat buoc da duoc enforce
- [ ] 4 manifest vi pham bi reject
- [ ] 1 manifest hop le duoc pass
- [ ] Da tu viet 1 custom `ConstraintTemplate`
- [ ] Custom policy co test reject va pass
- [ ] Moi thay doi da commit/push
- [ ] ArgoCD dang `Synced`

## Thu tu lam de nhanh nhat

1. Chot repo fork va xac nhan W9 xanh.
2. Lam RBAC va test bang `kubectl auth can-i`.
3. Cai Gatekeeper.
4. Bat `warn` cho 4 constraint, sua workload nao dang vi pham.
5. Chuyen sang `deny`, test reject/pass.
6. Lam custom policy.
7. Commit lan cuoi, sync ArgoCD, chup ket qua nghiem thu.
