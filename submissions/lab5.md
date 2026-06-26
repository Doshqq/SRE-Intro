# Lab 5 Submission

## Task 1 — CI Pipeline + ArgoCD Setup

### 1. GitHub Actions Run (Green Check)
Link to successful CI run: https://github.com/Doshqq/SRE-Intro/actions/runs/28209614694

### 2. Container Images Pushed to ghcr.io
The CI pipeline successfully built and pushed all three container images:
- ghcr.io/doshqq/quickticket-gateway:0ba009f93d10b859884bb6a07768fc569029542d
- ghcr.io/doshqq/quickticket-events:0ba009f93d10b859884bb6a07768fc569029542d
- ghcr.io/doshqq/quickticket-payments:0ba009f93d10b859884bb6a07768fc569029542d

### 3. ArgoCD Application Status
```
Name:               argocd/quickticket
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8443/applications/quickticket
Source:
- Repo:             https://github.com/Doshqq/SRE-Intro.git
  Target:           
  Path:             k8s
SyncWindow:         Sync Allowed
Sync Policy:        Automated
Sync Status:        Synced to  (009de97)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME      STATUS  HEALTH   HOOK  MESSAGE
       Service     default    gateway   Synced  Healthy        service/gateway unchanged
       Service     default    payments  Synced  Healthy        service/payments unchanged
       Service     default    postgres  Synced  Healthy        service/postgres unchanged
       Service     default    events    Synced  Healthy        service/events unchanged
       Service     default    redis     Synced  Healthy        service/redis unchanged
apps   Deployment  default    postgres   Synced  Healthy        deployment.apps/postgres unchanged
apps   Deployment  default    payments   Synced  Healthy        deployment.apps/payments unchanged
apps   Deployment  default    redis      Synced  Healthy        deployment.apps/redis unchanged
apps   Deployment  default    events     Synced  Healthy        deployment.apps/events unchanged
apps   Deployment  default    gateway    Synced  Healthy        deployment.apps/gateway configured
```

### 4. Git Change Synced to Cluster
Added version label "v2" to gateway deployment. After pushing to Git and syncing with ArgoCD:
```
kubectl get deployment gateway -o jsonpath='{.metadata.labels.version}'
v2
```

### 5. Answer: What happens if someone manually runs `kubectl edit` on a resource managed by ArgoCD?
ArgoCD will detect the drift between the Git repository state and the actual cluster state. The resource will show as "OutOfSync" in the ArgoCD UI. ArgoCD will then try to reconcile the state by reapplying the configuration from Git, effectively reverting the manual changes made with `kubectl edit`. This is the core principle of GitOps - Git is the single source of truth.

## Task 2 — Rollback via GitOps

### 1. ArgoCD Status After Bad Deploy
```
Name:               argocd/quickticket
Sync Status:        Synced to  (f7c200d)
Health Status:      Progressing

GROUP  KIND        NAMESPACE  NAME      STATUS  HEALTH       HOOK  MESSAGE
apps   Deployment  default    gateway    Synced  Progressing        deployment.apps/gateway configured
```

### 2. Pod Status After Bad Deploy
```
NAME                        READY   STATUS             RESTARTS   AGE
gateway-6bf7d9d68b-449vj    0/1     ImagePullBackOff   0          19s
```

### 3. Git Log Showing Deploy + Revert
```
009de97 Revert "feat: deploy new gateway version"
f7c200d feat: deploy new gateway version
318748b ci: update image tags to 1a2b8303a4f63a2a6492f7e5b553174d19a049cf
```

### 4. ArgoCD Status After Revert
```
Name:               argocd/quickticket
Sync Status:        Synced to  (009de97)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME      STATUS  HEALTH   HOOK  MESSAGE
apps   Deployment  default    gateway    Synced  Healthy        deployment.apps/gateway configured
```

### 5. Answer: How long from `git revert` + push to pods being healthy again?
Approximately 40 seconds from `git revert` + push to pods being healthy again. This includes:
- Time for ArgoCD to detect the Git change (automated sync policy)
- Time for ArgoCD to sync the new state to the cluster
- Time for Kubernetes to terminate the bad pod and start a new one with the correct image

## Bonus Task — Automated Image Tag Update

### 1. Updated CI Workflow
The CI workflow now includes automated image tag update:
```yaml
- name: Update image tags in manifests
  run: |
    SHA=${{ github.sha }}
    ACTOR_LOWER=$(echo "${{ github.actor }}" | tr '[:upper:]' '[:lower:]')
    sed -i "s|image: ghcr.io/.*/quickticket-gateway:.*|image: ghcr.io/${ACTOR_LOWER}/quickticket-gateway:${SHA}|" k8s/gateway.yaml
    sed -i "s|image: ghcr.io/.*/quickticket-events:.*|image: ghcr.io/${ACTOR_LOWER}/quickticket-events:${SHA}|" k8s/events.yaml
    sed -i "s|image: ghcr.io/.*/quickticket-payments:.*|image: ghcr.io/${ACTOR_LOWER}/quickticket-payments:${SHA}|" k8s/payments.yaml

- name: Commit and push manifest update
  run: |
    git config user.name "github-actions"
    git config user.email "github-actions@github.com"
    git add -f k8s/
    git diff --cached --quiet || git commit -m "ci: update image tags to ${{ github.sha }}"
    git push
```

The workflow also includes a condition to skip CI-triggered commits:
```yaml
if: "!startsWith(github.event.head_commit.message, 'ci:')"
```

### 2. Git Log Showing Code Commit → CI Tag-Update Commit
```
318748b ci: update image tags to 1a2b8303a4f63a2a6492f7e5b553174d19a049cf
1a2b830 feat: deploy new gateway version
a34dc9e ci: update image tags to 45039ded8a39506856ad8693a5adb2f27eae3677
```

### 3. ArgoCD Syncing Auto-Updated Tag
ArgoCD successfully synced the auto-updated image tags without manual intervention. The automated sync policy detected the commit from GitHub Actions and applied the new image tags to the cluster automatically.
