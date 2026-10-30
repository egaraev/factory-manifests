# secrets

Everything under `k8s/secrets/` except this file is git-ignored and never committed. Argo CD only watches `k8s/platform`, `k8s/llm`, ... so this folder is outside its sync/prune scope too.

A manifest references a secret by name; the secret itself gets created on the cluster with `kubectl`.

## litellm-secret (ai-models)

```sh
LITELLM_KEY=$(openssl rand -hex 32)
kubectl create namespace ai-models --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic litellm-secret \
  --namespace ai-models \
  --from-literal=master_key="$LITELLM_KEY"
```

## openrouter-secret (ai-models)

```sh
OPENROUTER_KEY=$(grep '^OPENROUTER_API_KEY=' ~/repos/factory-manifests/.env | head -1 | cut -d= -f2- | tr -d '"')
kubectl create namespace ai-models --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic openrouter-secret \
  --namespace ai-models \
  --from-literal=OPENROUTER_API_KEY="$OPENROUTER_KEY"
```

## factory-pipeline-env (pipeline)

Used by the phase-2 orchestrator CronJob and every per-ticket Job it spawns -- same values as `ai-software-factory`'s own local `.env`. `LITELLM_KEY` is the same value as `litellm-secret`'s `master_key` above; `LITELLM_BASE` can be left unset since the pipeline's in-cluster default already points at the `litellm` service in `ai-models`.

```sh
kubectl create namespace pipeline --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic factory-pipeline-env \
  --namespace pipeline \
  --from-literal=JIRA_URL="https://egaraev.atlassian.net" \
  --from-literal=JIRA_EMAIL="<your jira email>" \
  --from-literal=JIRA_API_TOKEN="<jira api token>" \
  --from-literal=GITHUB_TOKEN="<github pat>" \
  --from-literal=GITHUB_REPO="egaraev/factory-sample-app" \
  --from-literal=LITELLM_KEY="$LITELLM_KEY"
```

## Argo CD repo access (argocd)

```sh
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic factory-manifests-repo \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:egaraev/factory-manifests.git \
  --from-file=sshPrivateKey=<deploy key private half>
kubectl label secret factory-manifests-repo -n argocd \
  argocd.argoproj.io/secret-type=repository
```

Generate the key with `ssh-keygen -t ed25519 -f factory-manifests-deploy -N ""`, add the public half as a read-only deploy key on the GitHub repo.
