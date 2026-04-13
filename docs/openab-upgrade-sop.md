# OpenAB Version Upgrade SOP

| | |
|---|---|
| **Document Version** | 1.2 |
| **Last Updated** | 2026-04-13 |

## Environment Reference

| Item | Details |
|---|---|
| Deployment Method | Kubernetes (Helm Chart) |
| Deployment Name | `openab-kiro` |
| Pod Label | `app.kubernetes.io/instance=openab` |
| Helm Repo (GitHub Pages) | `https://openabdev.github.io/openab` |
| Helm Repo (OCI) | `oci://ghcr.io/openabdev/charts/openab` |
| Image Registry | `ghcr.io/openabdev/openab` |
| Git Repo | `github.com/openabdev/openab` |
| Agent Config | `/home/agent/.kiro/agents/default.json` |
| Steering Files | `/home/agent/.kiro/steering/` |
| PVC Mount Path | `/home/agent` (Helm); `.kiro` / `.local/share/kiro-cli` (raw k8s) |
| KUBECONFIG | `~/.kube/config` (must be set explicitly — default k3s config has insufficient permissions) |
| Namespace | `default` (adjust to match your actual deployment namespace) |

> ⚠️ The local kubectl defaults to reading `/etc/rancher/k3s/k3s.yaml`, which will result in a permission denied error. Before running any command, always set:
> ```bash
> export KUBECONFIG=~/.kube/config
> ```

> 💡 **Namespace setup (recommended):** If OpenAB is deployed in a non-default namespace, set the following at the start of your session to avoid having to append `-n <namespace>` to every command:
> ```bash
> export NS=openab          # replace with your actual namespace
> export KUBECONFIG=~/.kube/config
> alias kubectl="kubectl -n $NS"
> alias helm="helm -n $NS"
> ```
> All `kubectl` and `helm` commands in this SOP assume either the default namespace or that the above aliases are in effect.

---

## Upgrade Process Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Pre-Upgrade Preparation                  │
│  Check version info → Read Release Notes → Announce outage  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                          Backup                              │
│  Helm values / Agent config / Steering / hosts.yml / PVC    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Upgrade Execution (2 Phases)                │
│                                                             │
│  Step 1: Pre-release Validation                             │
│    helm upgrade --version x.x.x-beta.1                     │
│    └─ Discord functional test                               │
│         ├─ Pass ──────────────────────────┐                 │
│         └─ Fail → Wait for beta.2, retry  │                 │
│                                           ▼                 │
│  Step 2: Promote to Stable                                  │
│    helm upgrade --version x.x.x                            │
│    └─ kubectl rollout status                               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Post-Upgrade Verification                  │
│  Pod Ready? → Version check → Log check → Discord E2E test  │
│       │                                                     │
│       ├─ All pass → Send completion notice ✅               │
│       └─ Issues   → Proceed to rollback ↓                  │
└────────────────────────┬────────────────────────────────────┘
                         │ (on failure)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                       Rollback                               │
│                                                             │
│  Diagnose symptom                                           │
│   ├─ Pod won't start    → helm rollback <REVISION>          │
│   ├─ Broken / Pod OK    → rollback or hotfix                │
│   ├─ Config lost        → restore from backup               │
│   └─ Bot unresponsive   → kubectl rollout restart → rollback │
│                                                             │
│  Rollback done → Re-run verification → Send rollback notice │
└─────────────────────────────────────────────────────────────┘
```

---

## I. Pre-Upgrade Preparation

### 1. Check Current Version

> ℹ️ OpenAB is a pre-compiled Rust binary shipped inside a Docker image. There is **no source code or git repository** inside the container — version information must be retrieved from Helm or the image tag.

```bash
# Get the current running Pod
POD=$(kubectl get pod -l app.kubernetes.io/instance=openab -o jsonpath='{.items[0].metadata.name}')

# Check deployed Helm chart version and image tag
helm list -f openab
helm status openab

# Check the actual image the Pod is running (including tag / SHA)
kubectl get deployment openab-kiro \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

# Check latest releases on GitHub
# Visit https://github.com/openabdev/openab/releases

# List available versions from the Helm repo (requires repo to be added first — see Section III)
helm search repo openab --versions
```

### 2. Read the Release Notes

- Go to `https://github.com/openabdev/openab/releases/tag/<target-version>`
- Pay special attention to:
  - Breaking changes
  - Helm Chart values changes
  - Added or deprecated environment variables
  - Any migration steps

### 3. Check Node Resources

> Skipping this step risks the new Pod getting stuck in `Pending` if the node lacks capacity.

```bash
# Check allocatable resources on all nodes
kubectl describe nodes | grep -A 5 "Allocatable:"

# Check current resource requests across the cluster
kubectl top nodes

# Confirm the new image size has not changed significantly
# (check the release notes for any resource requirement changes)
```

### 4. Announce the Upgrade

> ⚠️ **Downtime is expected during every upgrade.** The deployment strategy is `Recreate` because the PVC is ReadWriteOnce, which does not support RollingUpdate. The old Pod is terminated before the new one starts — the Discord bot will be unavailable during this window, and this is expected behaviour.

- Notify all users via Discord channel / Slack / email:
  - Scheduled upgrade time and estimated downtime (typically 1–3 minutes)
  - Scope of impact (Discord bot will be offline during the upgrade)
  - Emergency contact

---

## II. Backup

### Backup Checklist

| Item | Command | Notes |
|---|---|---|
| Helm values | `helm get values openab -o yaml > openab-values-backup-$(date +%Y%m%d).yaml` | Current Helm deployment parameters |
| Agent config | `kubectl cp $POD:/home/agent/.kiro/agents/default.json ./backup-default.json` | Custom agent settings (model, prompt, tools, etc.) |
| Steering files | `kubectl cp $POD:/home/agent/.kiro/steering/ ./backup-steering/` | Steering docs such as IDENTITY.md |
| Skills | `kubectl cp $POD:/home/agent/.kiro/skills/ ./backup-skills/` | Custom agent skills (if any; skip if path does not exist) |
| hosts.yml | `kubectl cp $POD:/home/agent/.config/gh/hosts.yml ./backup-hosts.yml` | GitHub CLI credentials (including multi-account configs) |
| Full Secret export | `kubectl get secret openab-kiro -o yaml > ./backup-secret.yaml` | ⚠️ **SENSITIVE** — contains Discord token, STT key, and other credentials. Store securely, **never commit to version control**. Consider encrypting with `gpg` or [`age`](https://github.com/FiloSottile/age) before storing. |
| STT API Key | `kubectl get secret openab-kiro -o jsonpath='{.data.stt-api-key}' \| base64 -d > ./backup-stt-api-key.txt` | Required only if STT is enabled (`stt.enabled: true`) |
| PVC data | See "PVC Backup" section below | Persistent data mounted to the Pod (⚠️ manual step required) |
| Helm release history | `helm history openab > openab-helm-history-$(date +%Y%m%d).txt` | Useful reference for rollback |

### One-Click Backup Script

```bash
#!/bin/bash
# Note: set -e is intentionally omitted.
# Failures are recorded per step and reported at the end,
# so that a single failure does not prevent remaining items from being backed up.

export KUBECONFIG=~/.kube/config

BACKUP_DIR="openab-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

POD=$(kubectl get pod -l app.kubernetes.io/instance=openab -o jsonpath='{.items[0].metadata.name}')
if [ -z "$POD" ]; then
  echo "❌ OpenAB Pod not found. Aborting backup." && exit 1
fi

# Pre-check: kubectl cp directory operations require tar inside the container
if ! kubectl exec "$POD" -- which tar > /dev/null 2>&1; then
  echo "❌ tar not found in container. kubectl cp of directories will fail."
  echo "   Use 'kubectl exec' with a tar pipe instead, or use VolumeSnapshot for PVC backup."
  exit 1
fi

FAILED_STEPS=()

backup_step() {
  local desc="$1"; shift
  echo "=== $desc ==="
  if ! "$@"; then
    echo "⚠️  Failed: $desc (continuing with remaining steps...)"
    FAILED_STEPS+=("$desc")
  fi
}

backup_step "Backup Helm values"    bash -c "helm get values openab -o yaml > $BACKUP_DIR/values.yaml"
backup_step "Backup Agent config"   kubectl cp "$POD:/home/agent/.kiro/agents/" "$BACKUP_DIR/agents/"
backup_step "Backup Steering files" kubectl cp "$POD:/home/agent/.kiro/steering/" "$BACKUP_DIR/steering/"
kubectl cp "$POD:/home/agent/.kiro/skills/" "$BACKUP_DIR/skills/" 2>/dev/null \
  || echo "⚠️ skills/ directory not found, skipping"
backup_step "Backup hosts.yml"      kubectl cp "$POD:/home/agent/.config/gh/hosts.yml" "$BACKUP_DIR/hosts.yml"
backup_step "Backup full Secret"    bash -c "kubectl get secret openab-kiro -o yaml > $BACKUP_DIR/secret.yaml"
backup_step "Backup Helm history"   bash -c "helm history openab > $BACKUP_DIR/helm-history.txt"

echo ""
echo "=== Backup Summary: $BACKUP_DIR ==="
ls -la "$BACKUP_DIR/"

if [ ${#FAILED_STEPS[@]} -gt 0 ]; then
  echo ""
  echo "⚠️  The following backup steps FAILED:"
  for step in "${FAILED_STEPS[@]}"; do
    echo "   - $step"
  done
  echo ""
  echo "   Review the failures above before proceeding with the upgrade."
  echo "   A failed backup step means the corresponding data is NOT protected."
else
  echo "✅ All backup steps completed successfully."
fi

echo ""
echo "🔐 SECURITY REMINDER: $BACKUP_DIR/secret.yaml contains sensitive credentials"
echo "   (Discord token, STT key, etc.). Do NOT commit this file."
echo "   Consider encrypting it before storing:"
echo "     gpg --symmetric $BACKUP_DIR/secret.yaml"
echo "     # or: age -p -o $BACKUP_DIR/secret.yaml.age $BACKUP_DIR/secret.yaml"
```

### Verify the Backup

```bash
# Check for empty files in the backup directory
find $BACKUP_DIR -type f -empty

# Confirm values.yaml is readable
cat $BACKUP_DIR/values.yaml | head -20
```

### PVC Backup (⚠️ Manual Step)

> This step must be performed manually based on your PVC type. It cannot be automated.

```bash
# 1. List PVCs mounted to the Pod
kubectl get pod $POD -o jsonpath='{range .spec.volumes[*]}{.name}{"\t"}{.persistentVolumeClaim.claimName}{"\n"}{end}'

# 2. Option A: VolumeSnapshot (recommended — requires CSI driver support)
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: openab-pvc-snapshot-$(date +%Y%m%d)
spec:
  volumeSnapshotClassName: <your-snapshot-class>
  source:
    persistentVolumeClaimName: <pvc-name>
EOF

# 3. Option B: kubectl cp (suitable for small data volumes only)
#
# ⚠️ Size check before proceeding — kubectl cp may timeout or OOM on large datasets.
# Recommended limit: < 500 MB. For larger volumes, use Option A (VolumeSnapshot).
kubectl exec $POD -- du -sh <pvc-mount-path>
# If the output is within an acceptable range, proceed:
kubectl cp $POD:<pvc-mount-path> $BACKUP_DIR/pvc-data/
```

---

## III. Upgrade Execution

### Pre-check: Confirm Helm Repo is Configured

```bash
# GitHub Pages (stable releases — recommended for most cases)
helm repo add openab https://openabdev.github.io/openab
helm repo update

# List available versions
helm search repo openab --versions

# Or query via OCI Registry
helm show chart oci://ghcr.io/openabdev/charts/openab --version <target-version>
```

### Step 1: Pre-release Validation (Required)

> ⚠️ Per project convention, **a stable release must be preceded by a validated pre-release**. Do not skip directly to Step 2.
>
> **When can Step 1 be skipped?** Only if the release notes for the target stable version explicitly contain `pre-release-validated: true`, indicating that the corresponding pre-release has already been validated in another environment (e.g. a staging cluster). In all other cases, run Step 1 first.

```bash
BACKUP_VALUES="<backup-dir>/values.yaml"  # e.g. openab-backup-20260413-070000/values.yaml

# Dry-run the pre-release version first
helm upgrade openab openab/openab \
  --version <target-version>-beta.1 \
  -f "$BACKUP_VALUES" \
  --dry-run

# Deploy the pre-release
helm upgrade openab openab/openab \
  --version <target-version>-beta.1 \
  -f "$BACKUP_VALUES"

kubectl rollout status deployment/openab-kiro --timeout=300s

# Run functional tests in the Discord channel
# Verify basic conversation and tool calls work as expected
# If issues are found, wait for beta.2 and repeat this step
```

### Step 2: Promote to Stable

```bash
BACKUP_VALUES="<backup-dir>/values.yaml"

# Dry-run the stable version
helm upgrade openab openab/openab \
  --version <target-version> \
  -f "$BACKUP_VALUES" \
  --dry-run

# Deploy stable (short downtime is expected)
helm upgrade openab openab/openab \
  --version <target-version> \
  -f "$BACKUP_VALUES"

# Wait for the Pod to be ready
kubectl rollout status deployment/openab-kiro --timeout=300s
```

### Post-Upgrade Verification

```bash
POD=$(kubectl get pod -l app.kubernetes.io/instance=openab -o jsonpath='{.items[0].metadata.name}')

# 1. Check Pod status (must be Running and READY)
kubectl get pod -l app.kubernetes.io/instance=openab
kubectl wait --for=condition=Ready pod/$POD --timeout=120s

# 2. Verify version (from Helm and image tag — no source code in the container)
helm list -f openab
kubectl get deployment openab-kiro \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

# 3. Confirm the openab process is running
kubectl exec $POD -- pgrep -x openab

# 4. Check logs for errors (ERROR / WARN / panic)
kubectl logs deployment/openab-kiro --tail=100 | grep -iE "error|warn|panic|fatal"

# 5. Verify steering files and agent config are still present
#    (PVC content should survive upgrades, but this confirms it)
kubectl exec $POD -- ls /home/agent/.kiro/steering/
kubectl exec $POD -- cat /home/agent/.kiro/agents/default.json | head -5
#    If either path is missing, restore from backup (see Section IV: Restore Custom Config)

# 6. Discord E2E verification (final check)
# → Send a test message in the Discord channel
# → Confirm the bot responds and conversation works correctly
```

### Completion Notice

- Once all verifications pass, notify users:
  - Upgrade complete, service restored
  - New version number and summary of key changes
  - Contact channel for reporting any issues

---

## IV. Rollback

### Decision Tree

```
Issue detected after upgrade
         │
         ▼
    Pod status?
         │
         ├─ CrashLoopBackOff / Pending ──→ helm rollback <REVISION>
         │
         ├─ Running, but functionality broken
         │         │
         │         ├─ Quick fix possible (e.g. config error) ──→ hotfix (engineer)
         │         └─ Root cause unclear ────────────────────→ helm rollback <REVISION>
         │
         ├─ Running, but bot is unresponsive
         │         │
         │         └─ kubectl rollout restart deployment/openab-kiro
         │                   │
         │                   ├─ Recovers after restart ──→ Continue monitoring
         │                   └─ Still unresponsive      ──→ helm rollback <REVISION>
         │
         └─ Running, but custom config is missing
                   │
                   └─ Restore config from backup → kubectl rollout restart
```

| Symptom | Action |
|---|---|
| Pod fails to start (CrashLoopBackOff) | Helm rollback |
| Functionality broken, Pod is running | Rollback or hotfix  |
| Custom config lost | Restore config files from backup |
| Bot unresponsive | Restart Pod first; rollback if it persists |

### Helm Rollback

```bash
# 1. View release history
helm history openab

# 2. Roll back to a previous revision
helm rollback openab <REVISION>

# 3. Wait for the Pod to be ready
kubectl rollout status deployment/openab-kiro --timeout=300s

# 4. Confirm rollback succeeded
kubectl get pod -l app.kubernetes.io/instance=openab

# 5. Run full post-rollback verification (same as post-upgrade verification)
POD=$(kubectl get pod -l app.kubernetes.io/instance=openab -o jsonpath='{.items[0].metadata.name}')
kubectl wait --for=condition=Ready pod/$POD --timeout=120s
kubectl exec $POD -- pgrep -x openab
kubectl logs deployment/openab-kiro --tail=100 | grep -iE "error|warn|panic|fatal"
# → Send a test message in the Discord channel to confirm the bot responds
```

### Restore Custom Config

```bash
POD=$(kubectl get pod -l app.kubernetes.io/instance=openab -o jsonpath='{.items[0].metadata.name}')

# Restore agent config
kubectl cp ./backup-default.json $POD:/home/agent/.kiro/agents/default.json

# Restore steering files
# ⚠️ kubectl cp directory behaviour varies across versions — trailing slash matters.
# Use the tar pipe method below to avoid accidentally creating a nested directory
# (e.g. steering/steering/) which can happen with some kubectl versions.
kubectl exec $POD -- mkdir -p /home/agent/.kiro/steering
tar c -C ./backup-steering . | kubectl exec -i $POD -- tar x -C /home/agent/.kiro/steering

# Restore GitHub CLI credentials
kubectl cp ./backup-hosts.yml $POD:/home/agent/.config/gh/hosts.yml

# Restart Pod to apply changes
kubectl rollout restart deployment/openab-kiro
```
