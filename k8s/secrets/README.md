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

Used by the phase-2 orchestrator CronJob and every per-ticket Job it spawns -- same values as `ai-software-factory`'s own local `.env`. `LITELLM_KEY` is the same value as `litellm-secret`'s `master_key` above; `LITELLM_BASE` can be left unset since the pipeline's in-cluster default already points at the `litellm` service in `ai-models`. `REVIEWER_GITHUB_TOKEN` is optional -- a classic token (`repo` scope only) from a *second* GitHub account (invited as a `factory-sample-app` collaborator with Write access), used only so the reviewer agent can submit a real GitHub review instead of a plain comment. Without it, the reviewer agent just posts a comment. (Fine-grained PATs don't work for this -- GitHub won't let a fine-grained token on a personal account see a repo owned by a *different* personal account at all, even as a collaborator.)

```sh
kubectl create namespace pipeline --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic factory-pipeline-env \
  --namespace pipeline \
  --from-literal=JIRA_URL="https://egaraev.atlassian.net" \
  --from-literal=JIRA_EMAIL="<your jira email>" \
  --from-literal=JIRA_API_TOKEN="<jira api token>" \
  --from-literal=GITHUB_TOKEN="<github pat>" \
  --from-literal=GITHUB_REPO="egaraev/factory-sample-app" \
  --from-literal=LITELLM_KEY="$LITELLM_KEY" \
  --from-literal=REVIEWER_GITHUB_TOKEN="<second account's classic PAT, or omit this line entirely>"
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

## grafana-secret (observe)

```sh
kubectl create namespace observe --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic grafana-secret \
  --namespace observe \
  --from-literal=admin-user="admin" \
  --from-literal=admin-password="$(openssl rand -hex 16)"
```

## langfuse-postgres-secret (observe)

Dedicated postgres instance for langfuse (`k8s/observe/langfuse-postgres.yaml`) -- `password` is read by the postgres container itself, `DATABASE_URL` is the full connection string langfuse-web/worker read.

```sh
PG_PASSWORD=$(openssl rand -hex 16)
kubectl create namespace observe --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic langfuse-postgres-secret \
  --namespace observe \
  --from-literal=password="$PG_PASSWORD" \
  --from-literal=DATABASE_URL="postgresql://langfuse:$PG_PASSWORD@langfuse-postgres.observe.svc.cluster.local:5432/langfuse"
```

## langfuse-secret (observe)

NextAuth session secrets -- `SALT`/`ENCRYPTION_KEY` must each be exactly 32 bytes hex (64 hex chars) per LangFuse's own docs.

```sh
kubectl create secret generic langfuse-secret \
  --namespace observe \
  --from-literal=NEXTAUTH_SECRET="$(openssl rand -hex 32)" \
  --from-literal=SALT="$(openssl rand -hex 32)" \
  --from-literal=ENCRYPTION_KEY="$(openssl rand -hex 32)"
```

## langfuse-clickhouse-secret (observe)

```sh
kubectl create secret generic langfuse-clickhouse-secret \
  --namespace observe \
  --from-literal=CLICKHOUSE_USER="langfuse" \
  --from-literal=CLICKHOUSE_PASSWORD="$(openssl rand -hex 16)"
```

## langfuse-minio-secret (observe)

Doubles as the S3 access key/secret key langfuse itself uses to talk to its own minio.

```sh
kubectl create secret generic langfuse-minio-secret \
  --namespace observe \
  --from-literal=MINIO_ROOT_USER="langfuse" \
  --from-literal=MINIO_ROOT_PASSWORD="$(openssl rand -hex 16)"
```

## langfuse-litellm-secret (ai-models)

Lets litellm send traces to langfuse (`success_callback: ["langfuse"]` in `k8s/llm/litellm.yaml`). These keys don't exist yet at first deploy -- create a project in the langfuse UI (`http://<aibox-lan-ip>:30350`, or `http://localhost:30350` from the aibox host itself) after its first login, then grab its API keys from Project Settings and run this. litellm's secretKeyRef for these is `optional: true`, so it runs fine without this secret -- tracing just silently does nothing until it exists and the litellm pod restarts.

```sh
kubectl create secret generic langfuse-litellm-secret \
  --namespace ai-models \
  --from-literal=LANGFUSE_PUBLIC_KEY="<pk-lf-... from the langfuse project settings>" \
  --from-literal=LANGFUSE_SECRET_KEY="<sk-lf-... from the langfuse project settings>"
kubectl delete pod -n ai-models -l app=litellm
```
