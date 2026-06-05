# gitops-manifests
gitops-manifests

Kubernetes manifests demonstrating a secure GitOps deployment workflow with Argo CD.

This repository is the single source of truth for the desired state of a Kubernetes cluster. Whatever is described here, Argo CD automatically reconciles into the live cluster — no manual kubectl apply, no undocumented changes.
It was built as part of a Bachelor's thesis in cybersecurity (specialty 125, Igor Sikorsky Kyiv Polytechnic Institute) on secure web-application deployment in Kubernetes, and is shared publicly as a hands-on reference for anyone learning GitOps.
Why GitOps?
Traditional deployment relies on a push model: an external CI system pushes changes into the cluster and therefore has to store the cluster's credentials. This creates three problems:

No audit trail — manual kubectl changes leave no trace of who changed what and when.
No single source of truth — the real cluster state can silently drift from what anyone intended.
Larger attack surface — cluster credentials live in an external system that can be compromised.

GitOps flips this around with a pull model. An in-cluster agent (Argo CD) pulls the desired state from Git, so credentials never leave the cluster, every change is a Git commit with an author and timestamp, and the live state is continuously verified against the repository.
How it works
<img width="848" height="461" alt="{05618ED6-0B19-4DD5-A540-61B9989C992D}" src="https://github.com/user-attachments/assets/3545763a-b191-4a0b-9985-6fd27ec73382" />

You edit a manifest and commit it to main.
Argo CD detects the new commit and syncs the cluster (Auto-Sync).
Resources removed from the repo are also removed from the cluster (Prune).
Any manual change made directly in the cluster, bypassing Git, is automatically reverted (Self-Heal).

As a result, Git becomes a complete change log — you can always see who changed what and when. This auditability is GitOps's core security benefit.
Repository structure
FilePurposenamespace.yamlDefines the production namespacedeployment.yamlApplication deployment — container image and replica countservice.yamlNetwork service exposing the application
Getting started
Prerequisites

A Kubernetes cluster — k3s or kind work great for local testing
kubectl configured to talk to your cluster
Helm (used here to install Argo CD)

1. Install Argo CD
bash# Create a dedicated, isolated namespace for Argo CD
kubectl create namespace argocd

# Add the official Helm chart repo and install
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd

# Verify all components are running
kubectl get pods -n argocd

Note: Argo CD is installed in its own argocd namespace on purpose. If the application namespace is ever compromised, the attacker doesn't automatically gain access to the tool that controls the entire cluster.

2. Access the UI
bash# Forward the Argo CD server to your local machine
kubectl port-forward service/argocd-server -n argocd 8080:443

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
Open https://localhost:8080 and log in as admin.
3. Create the application
Point Argo CD at this repository — via the UI (+ NEW APP) or the CLI:
bashargocd app create gitops-app \
  --repo https://github.com/<your-username>/gitops-manifests.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated \
  --auto-prune \
  --self-heal
Argo CD will now deploy the application and keep its state synchronized with this repository.
Security notes
This setup demonstrates several security-relevant practices:

Least privilege via RBAC — a separate viewer role can only read application state, not modify it. Defining roles beyond the all-powerful admin is essential in real environments.
Namespace isolation — Argo CD runs separately from the workloads it manages.
Pull-based delivery — cluster credentials never leave the cluster.
Self-healing — protects against unauthorized direct changes (configuration drift).


⚠️ This is a learning environment. For production use, replace the self-signed certificate with a trusted one, manage secrets securely (e.g. Sealed Secrets or external secret stores — plain Kubernetes Secrets are only base64-encoded, not encrypted), and scan images for vulnerabilities before deployment.

Further reading

Argo CD Documentation
OpenGitOps — GitOps Principles
OWASP Kubernetes Top 10

