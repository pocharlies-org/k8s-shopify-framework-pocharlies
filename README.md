# k8s-shopify-framework-pocharlies

Shared Kustomize base for deploying skirmshop Shopify apps to k3s via ArgoCD.
The deployment counterpart to the [`@pocharlies/shopify-app-framework`](https://github.com/pocharlies/shopify-app-framework)
npm package (which provides the app-code side: OAuth, job queue, UI, Prisma).

All the boilerplate every Shopify app repeated — edge nodeSelector/tolerations,
`external-dns` + Traefik wiring, TLS store, shared-Postgres env, resource
defaults, the public IngressRoute — lives here once. Each app repo keeps a thin
overlay with only what differs (name, image, port, path, secret, extra env).

## Layout

```
base/                     Deployment + Service + IngressRoute (the 90% case)
  kustomizeconfig/        Traefik nameReference (so namePrefix rewrites refs)
components/
  forward-auth/           Gate an app behind Keycloak SSO (admin backends)
examples/                 Worked, validated overlays (relative base)
  bundles/  sii/  collections-tree/   standard embedded apps
  picker-admin/           bespoke admin tool: custom routing + SSO
```

## Add a new embedded Shopify app

In the app repo, create `k8s/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/pocharlies/k8s-shopify-framework-pocharlies//base?ref=deploy/prod
namePrefix: myapp-          # base name "app" -> "myapp-app"
labels:
  - includeSelectors: true  # unique selector so the Service only hits this app
    pairs: { app.kubernetes.io/instance: myapp }
images:
  - name: shopify-app
    newName: harbor.e-dani.com/homelab/shopify-myapp
    digest: sha256:...        # pin by digest (preferred) or newTag
patches:
  - path: deployment-env.yaml          # SHOPIFY_APP_URL, SCOPES, secret, extra env
    target: { kind: Deployment, name: app }
  - target: { group: apps, version: v1, kind: Deployment, name: app }   # if not :3000
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/ports/0/containerPort
        value: 3460
  - target: { group: traefik.io, version: v1alpha1, kind: IngressRoute, name: app }
    patch: |-
      - op: replace
        path: /spec/routes/0/match
        value: Host(`skirmshop.e-dani.com`) && PathPrefix(`/myapp`)
```

`deployment-env.yaml` is a strategic-merge patch (env merges by name; envFrom is
replaced):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: app }
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            - { name: PORT, value: "3460" }                                  # if not 3000
            - { name: SHOPIFY_APP_URL, value: "https://skirmshop.e-dani.com/myapp" }
            - { name: SCOPES, value: "read_products,write_products" }
          envFrom:
            - secretRef: { name: myapp-secrets }
```

See `examples/bundles` (minimal), `examples/collections-tree` (extra env, different
DB, bigger resources, liveness probe) for real shapes. Build locally with
`kubectl kustomize examples/bundles`.

## Base contract

- Namespace `skirmshop`; runs on `role: edge` amd64 nodes (nodeSelector + toleration).
- DB creds from the shared `shared-postgres-app` secret (username/password);
  `DATABASE_URL` defaults to the `skirmshop` database — override per app.
- App secret consumed via `envFrom` (overlay sets the real `*-secrets` name).
- Service publishes `:80 -> targetPort http` (named) so the port lives only on
  the container — change `containerPort` and you're done.
- IngressRoute: `skirmshop.e-dani.com/<prefix>` on `traefik-edge`, `external-dns`
  to `57.129.17.172` (Cloudflare-proxied), TLS via the shared `default` store.

## Auth invariants (read this — it's where OAuth breaks)

Shopify embedded-app OAuth is fragile about URLs. The base encodes the safe
defaults; keep them:

1. **`SHOPIFY_APP_URL` must equal the public path-prefixed URL exactly**
   (`https://skirmshop.e-dani.com/<prefix>`). A mismatch makes the OAuth
   callback redirect to the wrong place and the session never persists.
2. **Do NOT strip the prefix for embedded apps.** The app is mounted at
   `/<prefix>`; stripping it breaks App Bridge and the callback. (`strip` is only
   for non-embedded internal tools — see picker.)
3. **`PORT` must match `containerPort`.** The base sets `3000`; override both
   together if your app listens elsewhere, or the readiness probe (named `http`)
   passes while OAuth talks to the wrong port.
4. The app itself must expose the Remix auth routes (`_index.tsx` that triggers
   auth and redirects to `/app`, plus `auth.login.tsx` / `auth.$.tsx`). These come
   from `createApp()` in `@pocharlies/shopify-app-framework` — k8s can't supply
   them. If you get `/auth/login 404` or empty sessions, it's the app image, not
   this overlay.

## Protect an admin backend (Keycloak SSO)

Internal tools that are **not** Shopify-embedded (dashboards, the picker) should
sit behind Keycloak. For a standard single-route app, add the component:

```yaml
components:
  - github.com/pocharlies/k8s-shopify-framework-pocharlies//components/forward-auth?ref=deploy/prod
```

For an app with bespoke routing (multiple routes, redirects), inline the
`sso-chain` middleware ref directly in your IngressRoute instead — see
`examples/picker-admin/routing.yaml`. The middleware is `sso-chain` in namespace
`keycloak`.

> Never put SSO middleware in front of a Shopify-embedded app — it intercepts the
> OAuth/iframe flow.

## Gotchas

- **Middleware/Service refs and `namePrefix`.** Kustomize rewrites a Traefik ref
  only when the referenced resource lives in the *same layer*. Refs inside the
  base (the standard IngressRoute → base Service) rewrite fine. A ref **added by
  a component's patch is applied after the rename and is NOT rewritten** — that's
  why `forward-auth` only references the external `keycloak/sso-chain` middleware,
  and
  why bespoke routing (picker) is written inline with explicit names.
- **DB creds standardized** on `shared-postgres-app`. Apps that previously read
  `DB_USER`/`DB_PASSWORD` from their own `*-secrets` keep working (explicit env
  wins over `envFrom`), but confirm that secret is valid for your database.
- **Pin images by digest.** `newTag: latest` (picker) is a smell; prefer digest.
