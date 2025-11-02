# SOPS + ArgoCD Test Setup

This directory contains a test setup to verify that ArgoCD with SOPS support is working correctly with GPG-encrypted secrets.

## Directory Structure

```
sops-tests/
├── .sops.yaml                    # SOPS configuration file
├── manifests/                    # Plain (unencrypted) manifests
│   ├── test-secret.yaml         # Plain secret (for reference)
│   ├── test-configmap.yaml      # ConfigMap
│   └── test-pod.yaml            # Test pod that uses the secret
├── encrypted/                    # Encrypted manifests
│   └── test-secret.enc.yaml     # SOPS-encrypted secret
├── kustomization.yaml           # Kustomize configuration
├── argocd-application.yaml      # ArgoCD Application manifest
└── README.md                    # This file
```

## Prerequisites

1. ArgoCD installed with SOPS support
2. GPG key configured in ArgoCD (key fingerprint: `41DB93BCEEA30326`)
3. kubectl access to your cluster
4. SOPS CLI installed locally

## GPG Key Information

The test uses the following GPG key:
- **Key ID**: `41DB93BCEEA30326`
- **Full Fingerprint**: `4E307EDF57A9B6A79A22F9C041DB93BCEEA30326`
- **Identity**: anthrax.garmo.local (k8s) <anthrax@garmo.local>

## Quick Start

### 1. Verify SOPS Configuration

The `.sops.yaml` file configures SOPS to:
- Encrypt all `.yaml` files
- Only encrypt the `data` and `stringData` fields (keeps metadata readable)
- Use the GPG key `41DB93BCEEA30326`

### 2. Test SOPS Encryption/Decryption Locally

```bash
# View encrypted secret
cat encrypted/test-secret.enc.yaml

# Decrypt and view the secret
sops -d encrypted/test-secret.enc.yaml

# Re-encrypt after making changes to the plain secret
sops --encrypt manifests/test-secret.yaml > encrypted/test-secret.enc.yaml
```

### 3. Verify ArgoCD GPG Key Import

Make sure the GPG key is imported into ArgoCD:

```bash
# Export your GPG private key
gpg --export-secret-keys --armor 41DB93BCEEA30326 > /tmp/gpg-key.asc

# Import into ArgoCD (if not already done)
kubectl create secret generic sops-gpg \
  --from-file=sops.asc=/tmp/gpg-key.asc \
  -n argocd

# Verify ArgoCD repo-server has the key
kubectl exec -n argocd deploy/argocd-repo-server -- gpg --list-keys

# Clean up the exported key
rm /tmp/gpg-key.asc
```

### 4. Deploy with kubectl (Manual Test)

Test the encrypted secret directly with kubectl:

```bash
# Decrypt and apply the secret
sops -d encrypted/test-secret.enc.yaml | kubectl apply -f -

# Apply other manifests
kubectl apply -f manifests/test-configmap.yaml
kubectl apply -f manifests/test-pod.yaml

# Check if the pod can read the secrets
kubectl logs sops-test-pod

# Expected output:
# Username: admin
# Password: supersecretpassword123
# API Key: sk-1234567890abcdef

# Cleanup
kubectl delete pod sops-test-pod
kubectl delete secret test-sops-secret
kubectl delete configmap test-sops-configmap
```

### 5. Deploy with ArgoCD

#### Option A: Using kubectl

1. Update the `repoURL` in `argocd-application.yaml` with your Git repository
2. Push this directory to your Git repository
3. Apply the ArgoCD application:

```bash
kubectl apply -f argocd-application.yaml
```

#### Option B: Using ArgoCD CLI

```bash
# Login to ArgoCD
argocd login <argocd-server>

# Create the application
argocd app create sops-test-app \
  --repo https://github.com/YOUR_USERNAME/YOUR_REPO.git \
  --path sops-tests/encrypted \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 6. Verify ArgoCD Deployment

```bash
# Check application status
kubectl get applications -n argocd
argocd app get sops-test-app

# Check if resources are created
kubectl get secrets test-sops-secret
kubectl get configmap test-sops-configmap
kubectl get pod sops-test-pod

# Verify the secret was decrypted correctly
kubectl get secret test-sops-secret -o jsonpath='{.data.username}' | base64 -d
# Should output: admin

# Check pod logs
kubectl logs sops-test-pod
```

## Test Scenarios

### Scenario 1: Basic Encryption/Decryption
- Encrypt a secret with SOPS
- Decrypt it manually to verify
- Status: ✅ (completed during setup)

### Scenario 2: Manual kubectl Deployment
- Apply the encrypted secret using kubectl + SOPS
- Verify pod can access the decrypted values
- Tests: Local SOPS setup works

### Scenario 3: ArgoCD Sync
- Push encrypted manifests to Git
- Let ArgoCD sync and deploy
- Verify ArgoCD can decrypt and apply secrets
- Tests: ArgoCD SOPS integration works end-to-end

### Scenario 4: Update Secret
```bash
# Edit the plain secret
vim manifests/test-secret.yaml

# Re-encrypt
sops --encrypt manifests/test-secret.yaml > encrypted/test-secret.enc.yaml

# Commit and push
git add encrypted/test-secret.enc.yaml
git commit -m "Update secret"
git push

# ArgoCD should automatically detect and sync the change
argocd app sync sops-test-app
```

## Troubleshooting

### ArgoCD Can't Decrypt Secrets

1. Check if GPG key is properly imported:
```bash
kubectl exec -n argocd deploy/argocd-repo-server -- gpg --list-secret-keys
```

2. Check ArgoCD logs:
```bash
kubectl logs -n argocd deploy/argocd-repo-server
```

3. Verify SOPS plugin is installed:
```bash
kubectl exec -n argocd deploy/argocd-repo-server -- sops --version
```

### Secret Not Decrypted in Cluster

1. Verify the secret exists:
```bash
kubectl get secret test-sops-secret -o yaml
```

2. Check if the secret is still encrypted (it should be decrypted):
```bash
kubectl get secret test-sops-secret -o jsonpath='{.data.username}' | base64 -d
```

### Pod Can't Access Secret Values

1. Check pod status:
```bash
kubectl describe pod sops-test-pod
```

2. Verify secret keys match:
```bash
kubectl get secret test-sops-secret -o jsonpath='{.data}' | jq
```

## Cleanup

```bash
# Delete ArgoCD application
kubectl delete -f argocd-application.yaml
# or
argocd app delete sops-test-app

# Manually delete resources if needed
kubectl delete pod sops-test-pod
kubectl delete secret test-sops-secret
kubectl delete configmap test-sops-configmap
```

## Managing Secrets

### Editing Encrypted Secrets

#### Adding or Modifying Secret Values

SOPS provides an editor that automatically decrypts, allows editing, and re-encrypts the file:

```bash
# Edit an encrypted file in your default editor
sops encrypted/test-secret.enc.yaml

# Use a specific editor
EDITOR=vim sops encrypted/test-secret.enc.yaml
```

When you save and exit, SOPS automatically re-encrypts the file with the same keys.

#### Adding a New Key/Value to a Secret

**Method 1: Using SOPS Editor** (Recommended)

```bash
# Open the encrypted file for editing
sops encrypted/test-secret.enc.yaml

# In the editor, add your new key under stringData:
# stringData:
#   username: admin
#   password: supersecretpassword123
#   database-url: postgresql://user:pass@localhost:5432/mydb
#   api-key: sk-1234567890abcdef
#   new-key: new-secret-value  # <-- Add this line

# Save and exit - SOPS will re-encrypt automatically
```

**Method 2: Update Plain Secret and Re-encrypt**

```bash
# 1. Edit the plain secret file
vim manifests/test-secret.yaml

# 2. Add your new key under stringData:
# stringData:
#   username: admin
#   password: supersecretpassword123
#   database-url: postgresql://user:pass@localhost:5432/mydb
#   api-key: sk-1234567890abcdef
#   new-key: new-secret-value  # <-- Add this

# 3. Re-encrypt the entire file
sops --encrypt manifests/test-secret.yaml > encrypted/test-secret.enc.yaml

# 4. Commit and push
git add encrypted/test-secret.enc.yaml manifests/test-secret.yaml
git commit -m "Add new secret key"
git push
```

**Method 3: Using `sops set` (For Single Values)**

```bash
# Set a specific value in an encrypted file
sops --set '["stringData"]["new-key"] "new-secret-value"' encrypted/test-secret.enc.yaml

# This modifies the file in place
```

#### Removing a Key/Value from a Secret

```bash
# Method 1: Use SOPS editor
sops encrypted/test-secret.enc.yaml
# Delete the line you want to remove, save and exit

# Method 2: Update plain secret and re-encrypt
vim manifests/test-secret.yaml  # Remove the key
sops --encrypt manifests/test-secret.yaml > encrypted/test-secret.enc.yaml
```

#### Viewing Secret Values

```bash
# Decrypt and view the entire secret
sops -d encrypted/test-secret.enc.yaml

# Extract a specific value
sops -d --extract '["stringData"]["username"]' encrypted/test-secret.enc.yaml

# View in JSON format
sops -d --output-type json encrypted/test-secret.enc.yaml
```

### Managing Encryption Keys

#### Adding a New GPG Key to Existing Encrypted Secrets

You can add or remove GPG keys from encrypted files **without decrypting them** using SOPS key rotation. This is useful for team member onboarding/offboarding.

#### Add a New Key

```bash
# Add a new GPG key (by fingerprint) to an encrypted file
sops --rotate --add-pgp 1234567890ABCDEF encrypted/test-secret.enc.yaml

# Or add multiple keys at once
sops --rotate --add-pgp "1234567890ABCDEF,FEDCBA0987654321" encrypted/test-secret.enc.yaml
```

#### Remove a Key

```bash
# Remove a GPG key from an encrypted file
sops --rotate --rm-pgp 1234567890ABCDEF encrypted/test-secret.enc.yaml
```

#### Update All Files with New Keys

If you've updated `.sops.yaml` with new keys, rotate all encrypted files to match:

```bash
# Update .sops.yaml first with new key configuration
# Then rotate the encrypted file to use the new configuration
sops updatekeys encrypted/test-secret.enc.yaml

# Or rotate all encrypted files in a directory
find encrypted/ -name "*.enc.yaml" -exec sops updatekeys {} \;
```

#### Example: Team Member Onboarding

When a new team member joins and needs access to secrets:

```bash
# 1. They generate a GPG key and share their public key fingerprint
# 2. You add their key to .sops.yaml
echo "  - pgp: >-
      41DB93BCEEA30326,  # Your key
      NEW_TEAM_MEMBER_KEY  # New member's key" >> .sops.yaml

# 3. Update all encrypted files
sops updatekeys encrypted/test-secret.enc.yaml

# 4. Commit and push
git add .sops.yaml encrypted/test-secret.enc.yaml
git commit -m "Add new team member GPG key"
git push
```

#### Example: Team Member Offboarding

When removing access:

```bash
# 1. Remove their key from .sops.yaml
# 2. Rotate to remove their access
sops --rotate --rm-pgp REMOVED_MEMBER_KEY encrypted/test-secret.enc.yaml

# 3. Or use updatekeys if you've already updated .sops.yaml
sops updatekeys encrypted/test-secret.enc.yaml

# 4. Commit and push
git add .sops.yaml encrypted/test-secret.enc.yaml
git commit -m "Remove departed team member GPG key"
git push
```

### Viewing Current Keys

To see which keys can decrypt a file:

```bash
# Show metadata including all authorized keys
sops --show-metadata encrypted/test-secret.enc.yaml

# Extract just the PGP fingerprints
sops --show-metadata encrypted/test-secret.enc.yaml | grep "fp:"
```

### Important Notes on Key Management

- **Key rotation does NOT decrypt the file** - SOPS re-encrypts the data key with the new set of keys
- **You need at least one valid key** to perform rotation (your current key must be able to decrypt)
- **Update ArgoCD** - After adding/removing keys, update the `sops-gpg` secret in ArgoCD if the main key changes
- **Backup keys** - Always maintain secure backups of GPG private keys
- **Regular rotation** - Consider rotating keys periodically as a security best practice
- **SOPS uses envelope encryption**: The actual data is encrypted with a data key, which is then encrypted with your GPG/age keys. This allows key rotation without re-encrypting the entire file.

## Additional Resources

- [SOPS Documentation](https://github.com/mozilla/sops)
- [ArgoCD SOPS Integration](https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/)
- [Kustomize with SOPS](https://github.com/viaduct-ai/kustomize-sops)

## Notes

- The plain secret in `manifests/test-secret.yaml` is kept for reference and re-encryption purposes
- Never commit unencrypted secrets to Git (only commit files in `encrypted/`)
- The `.sops.yaml` file ensures consistent encryption across all team members
- Consider using age encryption instead of GPG for better performance and simpler key management
