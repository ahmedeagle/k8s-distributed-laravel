# CI/CD Setup for Laravel Kubernetes Deployment

## Overview
Automated pipeline: `laravel-docker` → ECR → `laravel-gitops-k8s` → ArgoCD → EKS

## Setup Instructions

### 1. Configure Secrets in `laravel-docker` Repository

Go to `laravel-docker` → Settings → Secrets and variables → Actions

Add these secrets:
```
AWS_ACCESS_KEY_ID          # Your AWS Access Key
AWS_SECRET_ACCESS_KEY      # Your AWS Secret Key
GITOPS_REPO_TOKEN          # GitHub Personal Access Token with repo scope
```

### 2. Configure Secrets in `laravel-gitops-k8s-Copy` Repository

Go to `laravel-gitops-k8s-Copy` → Settings → Secrets and variables → Actions

Add these secrets:
```
AWS_ACCESS_KEY_ID          # Your AWS Access Key (same as above)
AWS_SECRET_ACCESS_KEY      # Your AWS Secret Key (same as above)
```

### 3. Create GitHub Personal Access Token

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Give it a name: `GitOps CI/CD`
4. Select scopes:
   - ✅ `repo` (Full control of private repositories)
   - ✅ `workflow` (Update GitHub Action workflows)
5. Generate token and copy it
6. Add it as `GITOPS_REPO_TOKEN` secret in `laravel-docker` repository

### 4. Update Repository Names (if needed)

If your GitOps repository name is different, update this line in `laravel-docker/.github/workflows/build-and-push-ecr.yml`:

```yaml
https://api.github.com/repos/${{ github.repository_owner }}/laravel-gitops-k8s/dispatches
```

Change `laravel-gitops-k8s` to your actual repo name.

## How It Works

### Automatic Flow

```
1. Developer pushes code to laravel-docker/main
   ↓
2. GitHub Actions triggers automatically
   ↓
3. Builds Docker image
   ↓
4. Pushes to AWS ECR with auto-generated tag
   Format: YYYYMMDD-HHMMSS-<git-sha>
   Example: 20251116-143022-a1b2c3d
   ↓
5. Triggers GitOps repository via webhook
   ↓
6. GitOps workflow updates deployment YAMLs
   ↓
7. Commits and pushes changes to main
   ↓
8. ArgoCD detects change and syncs to EKS
   ↓
9. Kubernetes rolls out new pods with new image
```

### Manual Trigger

#### Option 1: From laravel-docker repo
```bash
# Go to Actions tab → Build and Push to ECR → Run workflow
# Optional: Add custom tag suffix
```

#### Option 2: From laravel-gitops-k8s-Copy repo
```bash
# Go to Actions tab → Auto Update on Laravel Docker Changes
# Input full ECR image URI
# Example: 123456.dkr.ecr.us-east-1.amazonaws.com/laravel-app:20251116-143022-a1b2c3d
```

## Files Created

### In `laravel-gitops-k8s-Copy`:
- `.github/workflows/build-and-deploy.yml` - Full CI/CD (alternative)
- `.github/workflows/update-image.yml` - Update deployment manifests

### In `laravel-docker`:
- `.github/workflows/build-and-push-ecr.yml` - Build and push to ECR

## Workflow Features

### 🏗️ Build and Push (laravel-docker)
- ✅ Auto-generates unique tags with timestamp + git SHA
- ✅ Pushes to ECR with tag and `:latest`
- ✅ Triggers GitOps update automatically
- ✅ Manual trigger support with custom tag suffix
- ✅ Detailed summary in GitHub Actions UI

### 🔄 Update Manifests (laravel-gitops-k8s-Copy)
- ✅ Updates `laravel-deployment.yaml`
- ✅ Updates `laravel-worker-deployment.yaml`
- ✅ Auto-commits and pushes changes
- ✅ Triggers ArgoCD sync automatically

## Testing the Pipeline

### Test End-to-End:

1. Make a change in `laravel-docker`:
```bash
cd /home/aemam/wsl-projects/laravel-docker
echo "// Test change" >> routes/web.php
git add .
git commit -m "test: Pipeline test"
git push
```

2. Watch the magic happen:
```bash
# Monitor laravel-docker actions
# Wait for build completion (~5 min)
# Check laravel-gitops-k8s-Copy for new commit
# Watch ArgoCD sync
# Check pods rolling update
kubectl get pods -n laravel-app -w
```

3. Verify new image:
```bash
kubectl describe pod -n laravel-app | grep Image:
```

## Troubleshooting

### Issue: GitOps update not triggered
**Solution**: Check `GITOPS_REPO_TOKEN` secret in laravel-docker repo

### Issue: ECR push failed
**Solution**: Verify AWS credentials and ECR repository exists:
```bash
aws ecr describe-repositories --repository-names laravel-app
```

### Issue: ArgoCD not syncing
**Solution**: Check ArgoCD auto-sync is enabled:
```bash
kubectl get application laravel-app -n argocd -o yaml | grep automated
```

Enable auto-sync:
```bash
kubectl patch application laravel-app -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

## Rollback

### Quick rollback to previous image:
```bash
# Get previous image tag from git history
cd /home/aemam/wsl-projects/laravel-gitops-k8s-Copy
git log --oneline -5 base/laravel/laravel-deployment.yaml

# Revert to specific commit
git revert <commit-hash>
git push

# Or manually update and commit
vim base/laravel/laravel-deployment.yaml
# Change image tag to previous version
git add base/laravel/laravel-deployment.yaml
git commit -m "rollback: Revert to previous image"
git push
```

## Benefits

✅ **No Manual Steps**: Push code → Everything automated  
✅ **Traceable**: Every deployment has git commit + image tag  
✅ **Rollback Ready**: Git history = deployment history  
✅ **Multi-Environment**: Easy to extend for dev/staging/prod  
✅ **Audit Trail**: All changes tracked in Git  

## Next Steps

1. ✅ Commit the workflows to Git
2. ✅ Configure GitHub secrets
3. ✅ Test the pipeline
4. 🔜 Add Slack/Discord notifications
5. 🔜 Add deployment approvals for production
6. 🔜 Add automated tests before build
7. 🔜 Add multi-environment support (dev/staging/prod)
