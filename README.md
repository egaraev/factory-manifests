# factory-manifests

Kubernetes manifests for the AI factory's infrastructure — GPU scheduling, llama-swap, and LiteLLM, kept in sync by Argo CD.

## Layout

```
k8s/
├── bootstrap/     Argo CD Applications
├── platform/      namespace, GPU RuntimeClass, device plugin
├── llm/           llama-swap, litellm
└── secrets/       how to create the real secrets (never committed)
```

## Bootstrapping

1. Point `kubectl` at the cluster.
2. Install Argo CD:
   ```sh
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
3. Create the repo-access secret and the app secrets — see `k8s/secrets/README.md`.
4. `kubectl apply -f k8s/bootstrap/platform-app.yaml`
5. `kubectl apply -f k8s/bootstrap/llm-app.yaml`
