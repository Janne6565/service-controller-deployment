# service-controller-deployment

Kustomize manifests for [service-controller-backend](https://github.com/Janne6565/service-controller-backend),
watched by ArgoCD. Nothing here is applied by hand.

```
base/                 deployment + service
overlays/main/        namespace, ingress, sealed secret, image tag pin (CI-owned)
argocd/main-app.yaml  reference copy of the Application (the live one lives in cluster-deployment)
```

- **Namespace:** `service-controller`
- **Host:** `service-controller.jannekeipert.de` (own cert via the cert-manager ingress shim)
- **Replicas: 1, and it must stay 1.** Agents hold a websocket to one pod and the registry of who
  is connected is in that pod's memory. A second replica would accept `/execute_action` for agents
  it cannot see and deliver the command nowhere.

## Secrets

`overlays/main/secret.sealed.yaml` is a SealedSecret holding `SERVICE_CONTROLLER_AGENT_TOKEN` and
`SERVICE_CONTROLLER_PASSWORD_HASH`. To rotate either:

```bash
kubectl create secret generic service-controller-secrets \
  --namespace=service-controller \
  --from-literal=SERVICE_CONTROLLER_AGENT_TOKEN='<token>' \
  --from-literal=SERVICE_CONTROLLER_PASSWORD_HASH="$(htpasswd -bnBC 12 '' '<password>' | tr -d ':\n')" \
  --dry-run=client -o yaml | kubeseal --format yaml > overlays/main/secret.sealed.yaml
```

Commit the result and let ArgoCD sync. Rotating the agent token disconnects every agent until each
one's URL is updated.
