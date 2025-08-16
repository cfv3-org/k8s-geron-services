# K8s Helmfile Config

This repository ([cfv3-org/k8s-geron-services](https://github.com/cfv3-org/k8s-geron-services)) contains Helmfile-based configuration for my Kubernetes cluster.  
It ships environment-scoped values (e.g. `values/development/*.yaml`), hooks, and small utility charts (e.g. MetalLB IP allocation).

## Prerequisites

- Kubernetes cluster reachable from your machine
- macOS with Homebrew
- Access to your kubeconfig for the target cluster

```bash
brew install helm helmfile kubectl sops age
```

> Optional but useful: `brew install jq yq pre-commit`

## Folder layout (short)

```
.
├─ environments/
│  └─ development.yaml          # Helmfile environment config
├─ values/
│  └─ development/*.yaml        # per-chart values for the 'development' env
├─ hooks/                       # pre/post apply hooks (namespaces, manifests)
├─ metallb-config/              # tiny chart for L2 address pools etc.
├─ endpoints-config/             # endpoints/service small chart
└─ helmfile.yaml                # main entrypoint
```

## Services (current)

- nginx-ingress
- cert-manager
- external-dns
- external-secrets
- echo-server
- metrics-server
- nfs-safe
- cloudflare-tunnel-ingress-controller
- metallb

## Pointing Helmfile to your kubeconfig

Helmfile can read kubeconfig through:
- `KUBECONFIG` environment variable, or
- `--kubeconfig` flag, and optionally `--kube-context`.

Examples:

```bash
# 1) Using env var
export KUBECONFIG=~/.kube/config    # or a custom path
helmfile --environment development apply

# 2) Using explicit flags
helmfile --kubeconfig /path/to/kubeconfig          --kube-context my-cluster          -e development apply
```

You can also set the context via kubectl and let Helmfile pick it up:
```bash
kubectl config use-context geron
helmfile -e development apply
```

## Typical workflow

Dry-run and review diffs first:

```bash
helmfile -e development lint                    # list manifests for errors
helmfile -e development template                # render manifests
helmfile -e development diff                    # show changes vs cluster
```

Apply changes:

```bash
helmfile -e development apply
```

Destroy releases for the environment (use with care):

```bash
helmfile -e development destroy
```

Target a single release:

```bash
helmfile -e development -l name=nginx-ingress apply
```

## Environments

Helmfile environments are defined under `environments/`.  
Currently only `development` is present.  
Select the env via `-e <name>` (defaults to `default` if not provided).

Example:
```bash
helmfile -e development apply
```

---

## Roadmap (security & best practices)

1. **Adopt SOPS + age for encrypted values in Git**
   - Encrypted per-environment secret files (e.g. `values/development/secret.sops.yaml`).
   - Keys stored securely in password manager, not in repo.
   - Decrypt only in-memory or via tmpfs/ramdisk.

2. **External Secrets Operator (ESO) + Vault integration**
   - Move secret management into cluster-native workflows.
   - Store only references in Git, not raw secrets.

3. **Add production environment**
   - Create `environments/production.yaml` and corresponding values.

4. **RAM-only handling of decrypted secrets**
   - Mount tmpfs for decrypted files, shred after use.

5. **Self-hosted GitHub runner integration**
   - On each commit, automatically apply Helmfile to target cluster.

6. **CI integration hardening**
   - Require `--kube-context` in CI/CD pipelines to avoid deploying to wrong cluster.

7. **Validation & policy checks**
   - Add helm lint, kubeconform, and pre-commit hooks.

---

## Troubleshooting

- “No context found”: set `KUBECONFIG` or pass `--kubeconfig/--kube-context`.
- “DRY-RUN succeeded but apply failed”: re-run `helmfile diff` to see server-side hooks/CRDs readiness, then `apply`.
- CRDs (e.g., cert-manager) must be installed before dependent releases. Use `needs`/hooks or separate apply steps.

## References

- Helmfile: https://helmfile.readthedocs.io  
- Helm: https://helm.sh  
- kubectl: https://kubernetes.io/docs/reference/kubectl/  
- SOPS (Mozilla): https://github.com/getsops/sops  
- age: https://github.com/FiloSottile/age  
- External Secrets Operator: https://external-secrets.io  
- HashiCorp Vault: https://developer.hashicorp.com/vault
