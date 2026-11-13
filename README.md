# githubactions_gitops

Repo GitOps: **nguồn sự thật duy nhất** cho những gì chạy trên Kubernetes — image nào, tag nào,
bao nhiêu replicas, port nào, probe/resources/rolling update ra sao. ArgoCD theo dõi repo này và
tự đồng bộ vào cluster. Code ứng dụng nằm ở repo khác:

| App | Repo code | Image | Thư mục |
|---|---|---|---|
| node-app | [githubactions](https://github.com/kahnpott9r/githubactions) | `ghcr.io/kahnpott9r/githubactions` | [apps/node-app](apps/node-app) |
| java-app | [githubactions_java](https://github.com/kahnpott9r/githubactions_java) | `ghcr.io/kahnpott9r/githubactions_java` | [apps/java-app](apps/java-app) |

## Cấu trúc

```
apps/
  node-app/
    namespace.yaml       # namespace "todo"
    deployment.yaml      # replicas, port 3000, probe /api/health, rolling update
    service.yaml         # NodePort 30000
    kustomization.yaml   # images[].newTag  <- CI cập nhật
  java-app/
    deployment.yaml      # replicas, port 8080, probe /actuator/health, rolling update
    service.yaml         # NodePort 30080
    kustomization.yaml   # images[].newTag  <- CI cập nhật
argocd/
  applications.yaml      # 2 ArgoCD Application (auto sync, prune, self-heal)
```

| | node-app | java-app |
|---|---|---|
| Replicas | 2 | 2 |
| Container port | 3000 | 8080 |
| Service | NodePort 30000 | NodePort 30080 |
| Rolling update | maxSurge 1, maxUnavailable 0 | maxSurge 1, maxUnavailable 0 |
| Requests / limits | 50m, 64Mi / 128Mi | 100m, 256Mi / 512Mi |

## Luồng CI/CD

```
push main (githubactions / githubactions_java)
  └─ CI: test ─▶ docker build & push ghcr.io/...:sha-<abc1234>
       └─ job update-gitops: sửa apps/<app>/kustomization.yaml (newTag: sha-abc1234)
            └─ commit + push vào repo này
                 └─ ArgoCD phát hiện thay đổi ─▶ rolling update trên cluster
```

Tag dùng `sha-<7 ký tự commit>` (bất biến) thay vì `latest`, nên mỗi lần đổi image là một commit
trong repo này → biết chính xác version nào đang chạy, rollback bằng `git revert`.

### Secret cần cấu hình

Ở **cả 2 repo** `githubactions` và `githubactions_java`, thêm secret `GITOPS_REPO_TOKEN`:
fine-grained PAT chỉ cấp cho repo `githubactions_gitops`, quyền **Contents: Read and write**.
Thiếu secret này, job `update-gitops` chỉ in cảnh báo và bỏ qua.

### Đổi cấu hình bằng tay

Sửa replicas / port / resources trực tiếp trong `apps/*/deployment.yaml`, mở PR. Workflow
[validate.yml](.github/workflows/validate.yml) chạy `kubectl kustomize` để kiểm tra manifest.

Kiểm tra local:

```bash
kubectl kustomize apps/node-app
kubectl kustomize apps/java-app
```

## Cài vào cluster

```bash
kubectl apply -f argocd/applications.yaml
```

Nếu image GHCR là private, cluster cần `imagePullSecret` (hoặc đổi package sang public trong
GitHub → Packages → Package settings). Nếu repo này private, thêm repo vào ArgoCD
(Settings → Repositories) bằng PAT quyền Contents: Read-only.
