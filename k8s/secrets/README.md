# Secrets — Never Commit Actual Values

Secrets are created manually on the cluster using `kubectl`.
They are NEVER stored in Git.

## Create secrets for prod

```bash
# PostgreSQL
kubectl create secret generic postgres-secret \
  --namespace=algo-merlin-prod \
  --from-literal=username=trader \
  --from-literal=password=YOUR_SECURE_PASSWORD

# Platform (BingX + Telegram)
kubectl create secret generic platform-secret \
  --namespace=algo-merlin-prod \
  --from-literal=bingx-api-key=YOUR_BINGX_KEY \
  --from-literal=bingx-api-secret=YOUR_BINGX_SECRET \
  --from-literal=telegram-bot-token=YOUR_TELEGRAM_TOKEN \
  --from-literal=telegram-chat-id=YOUR_CHAT_ID

# AI Service
kubectl create secret generic ai-service-secret \
  --namespace=algo-merlin-prod \
  --from-literal=openrouter-api-key=YOUR_OPENROUTER_KEY
```

## Create secrets for test

Same commands with `--namespace=algo-merlin-test`.
Test environment should use separate API keys where possible.

## Verify secrets exist

```bash
kubectl get secrets -n algo-merlin-prod
kubectl get secrets -n algo-merlin-test
```
