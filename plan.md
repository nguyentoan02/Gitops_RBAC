# Kế hoạch chạy project

## 1. Mục tiêu tôi hiểu từ repo

Repo này không phải app local đơn lẻ để `npm start` hay `python app.py` là đủ. Đây là một demo GitOps/Kubernetes gồm:

- 1 Flask API ở `src/api`
- Argo Rollouts để canary deploy
- Prometheus + Alertmanager để scrape metrics và bắn alert
- Argo CD theo mô hình App of Apps để sync toàn bộ manifest

Mục tiêu chạy đúng nghĩa của repo là dựng một cluster local, cài Argo CD + Argo Rollouts + kube-prometheus-stack, rồi để Argo CD sync các app trong repo.

## 2. Những gì tôi đã kiểm tra

- README mô tả quy trình chạy bằng `minikube` + `kubectl`
- API image trong rollout đang dùng image public: `ghcr.io/Vuong-Bach/w10-api:0.0.1`
- `src/api/Dockerfile` và `src/api/app.py` tồn tại, nên có thể build image local nếu không muốn phụ thuộc image public
- `argocd/root.yaml` và các child application cần trỏ tới repo:
  - `https://github.com/nguyentoan02/Gitops_RBAC.git`
- Nhưng `git remote origin` của working copy hiện tại là:
  - `https://github.com/nguyentoan02/Gitops_RBAC.git`

## 3. Rủi ro / blocker cần review trước

### Rủi ro lớn nhất: Argo CD đang sync sai repo

Nếu chạy nguyên trạng:

- Argo CD sẽ không deploy từ thư mục local hiện tại
- Nó sẽ pull manifest từ repo được khai báo trong manifest Argo CD
- Kết quả có thể khác nội dung bạn đang review trong máy

Tôi coi đây là blocker số 1 trước khi chạy thật.

### Rủi ro 2: phụ thuộc mạng ngoài

Để chạy full stack, cluster cần kéo:

- manifest Argo CD từ GitHub
- Helm chart `argo-rollouts`
- Helm chart `kube-prometheus-stack`
- container image như `ghcr.io/Vuong-Bach/w10-api:0.0.1`

Nếu mạng chặn GitHub/GHCR/Helm repo thì plan cần đổi sang preload image/chart.

### Rủi ro 3: alert email chưa sẵn sàng

`app-alert/email-secret.yaml` chưa có file thật, chỉ có file mẫu. Phần email alert là optional, nhưng nếu muốn demo full thì phải tạo secret trước.

## 4. Kế hoạch chạy đề xuất

### Pha A - chốt nguồn deploy

Mục tiêu: đảm bảo Argo CD sync đúng repo cần chạy.

Phương án đề xuất:

1. Sửa toàn bộ `repoURL` trong `argocd/*.yaml` sang repo thật sẽ được Argo CD theo dõi
2. Repo đích đã chốt:
   - `https://github.com/nguyentoan02/Gitops_RBAC.git`
3. Nếu muốn test đúng code local đang mở, cần cập nhật:
   - `argocd/root.yaml`
   - `argocd/apps/app-common.yaml`
   - `argocd/apps/app-analysis.yaml`
   - `argocd/apps/app-alert.yaml`
   - `argocd/apps/app-api.yaml`
4. Chỉ sau khi chốt xong repoURL mới nên apply Argo CD root app

Kết quả mong đợi:

- Argo CD sync đúng source
- Tránh tình trạng “chạy được nhưng không phải code đang review”

### Pha B - chuẩn bị môi trường local

Mục tiêu: sẵn sàng một cluster local đủ để chạy demo.

Checklist:

1. Cài sẵn:
   - Docker Desktop
   - `kubectl`
   - `minikube`
   - `git`
2. Tạo cluster:
   - `minikube start -p w10 --driver=docker`
3. Chuyển context:
   - `kubectl config use-context w10`
4. Kiểm tra cluster sống:
   - `kubectl get nodes`

Kết quả mong đợi:

- Có cluster `w10`
- `kubectl` nói chuyện được với cluster

### Pha C - cài Argo CD

1. Tạo namespace:
   - `kubectl create ns argocd`
2. Cài manifest chuẩn của Argo CD
3. Chờ `argocd-server` lên
4. Lấy mật khẩu admin ban đầu
5. Port-forward UI nếu cần review trực quan

Kết quả mong đợi:

- Namespace `argocd` hoạt động
- Truy cập được Argo CD UI/API

### Pha D - deploy App of Apps

1. Apply:
   - `kubectl apply -f argocd/root.yaml`
2. Theo dõi các child application được tạo:
   - `common`
   - `kube-prometheus-stack`
   - `argo-rollouts`
   - `analysis`
   - `alert`
   - `api`
3. Kiểm tra thứ tự sync wave:
   - wave `-1`: common
   - wave `0`: infrastructure
   - wave `1`: analysis + alert
   - wave `2`: api

Kết quả mong đợi:

- Argo CD tạo đủ child apps
- Hạ tầng monitoring/rollout lên trước khi app API sync

### Pha E - xác minh runtime

1. Kiểm tra namespace:
   - `demo`
   - `monitoring`
   - `argo-rollouts`
2. Kiểm tra rollout:
   - `kubectl get rollout api -n demo`
3. Kiểm tra pod API:
   - `kubectl get pods -n demo -l app=api`
4. Kiểm tra ServiceMonitor và metric scrape
5. Kiểm tra AnalysisRun:
   - `kubectl get analysisrun -n demo`

Kết quả mong đợi:

- API pod chạy
- `ServiceMonitor` được Prometheus nhận
- `AnalysisRun` sinh ra khi rollout chạy

### Pha F - test các kịch bản demo

#### Kịch bản 1: rollout thành công

1. Set `ERROR_RATE = "0"` trong `app-api/rollout.yaml`
2. Commit + push repo mà Argo CD đang theo dõi
3. Quan sát canary đi qua 10% -> 50% -> 100%

Kết quả mong đợi:

- Analysis pass
- Rollout promote thành công

#### Kịch bản 2: rollout thất bại

1. Set `ERROR_RATE = "0.15"`
2. Commit + push
3. Theo dõi `AnalysisRun` fail
4. Kiểm tra rollback

Kết quả mong đợi:

- Success rate dưới ngưỡng `0.90`
- Rollout không promote full, hoặc bị rollback

#### Kịch bản 3: bắn alert SLO

1. Set `ERROR_RATE = "0.10"`
2. Commit + push
3. Đợi Prometheus rule đánh giá trong vài phút
4. Nếu đã cấu hình email secret, kiểm tra mailbox

Kết quả mong đợi:

- Canary vẫn có thể pass
- Rule `SLOViolation` fire vì success rate dưới `0.95`

## 5. Phương án chạy tối thiểu nếu chỉ muốn verify code API

Nếu chưa muốn dựng full Kubernetes stack, có thể chạy phần API riêng:

1. Build image từ `src/api/Dockerfile`, hoặc
2. Chạy Flask app local với env:
   - `VERSION`
   - `ERROR_RATE`
3. Kiểm tra các endpoint:
   - `/`
   - `/healthz`
   - `/metrics`

Giá trị của phương án này:

- verify được app Python
- verify được metrics endpoint
- nhưng không verify được GitOps, Rollouts, AnalysisTemplate, Alertmanager

## 6. Thứ tự thực thi tôi đề xuất

1. Sửa hoặc xác nhận lại `repoURL` trong Argo CD manifests
2. Dựng `minikube`
3. Cài Argo CD
4. Apply `argocd/root.yaml`
5. Theo dõi sync và fix lỗi image/chart nếu có
6. Cấu hình email secret nếu cần demo alert
7. Chạy các kịch bản thay đổi `ERROR_RATE`

## 7. Review points tôi muốn bạn xác nhận

1. Bạn muốn tôi chuẩn bị plan theo hướng:
   - chỉ chạy API local
   - hay chạy full GitOps/Kubernetes demo
2. Bạn có muốn tôi tạo luôn file secret local cho Gmail App Password sau khi bạn chuẩn bị password không?
