# service-controller-deployment

Kustomize manifests for [service-controller-backend](https://github.com/Janne6565/service-controller-backend),
watched by ArgoCD. Nothing here is applied by hand.

```
base/backend/         deployment + service
base/postgres/        statefulset + service (clients, audit log, last-commanded state)
overlays/main/        namespace, the two ingress routers, forward-auth middleware,
                      sealed secret, image tag pin (CI-owned)
argocd/main-app.yaml  reference copy of the Application (the live one lives in cluster-deployment)
```

- **Namespace:** `service-controller`
- **Host:** `service-controller.jannekeipert.de` (own cert via the cert-manager ingress shim)
- **Two routers on that host:** the admin UI and `/api` behind Authentik forward auth; the agent
  websocket and the machine endpoints on a second, middleware-free router. See the headers of
  `overlays/main/ingress.yaml` and `authentik.yaml`.
- **Replicas: 1, and it must stay 1.** Agents hold a websocket to one pod and the registry of who
  is connected is in that pod's memory. A second replica would accept a command for agents it
  cannot see and deliver it nowhere.

## Access

Membership of the Authentik group **`service-controller-users`** is what gets a person past the
outpost. The provider, application and group live in
`cluster-deployment/infrastructure/authentik-service-controller-blueprint.yaml`.

## Secrets

`overlays/main/secret.sealed.yaml` holds only `SERVICE_CONTROLLER_DB_PASSWORD`. Client tokens are
not configuration: they are created in the admin UI and stored hashed in the database.

To rotate the database password (it is read by both the app and the Postgres StatefulSet, so they
roll together):

```bash
kubectl create secret generic service-controller-secrets \
  --namespace=service-controller \
  --from-literal=SERVICE_CONTROLLER_DB_PASSWORD='<password>' \
  --dry-run=client -o yaml | kubeseal --format yaml > overlays/main/secret.sealed.yaml
```

Changing it does **not** re-initialise an existing database — Postgres only reads
`POSTGRES_PASSWORD` on first init, so an existing volume keeps the old password and the app will
fail to connect. Change it inside Postgres with `ALTER ROLE` first, or accept a wipe.
